# Game Audio Middleware Patterns

Reference patterns for Wwise plugin authoring, FMOD DSP effect authoring, Unreal MetaSounds, and thread/memory safety in each middleware.

---

## Wwise Plugin Structure

### Minimal effect plugin layout

```cpp
// MyEffect.h
class MyEffect : public AK::IAkInPlaceEffectPlugin {
public:
    AKRESULT Init(AK::IAkPluginMemAlloc* in_pAllocator,
                  AK::IAkEffectPluginContext* in_pContext,
                  AK::IAkPluginParam* in_pParams,
                  AkAudioFormat& io_rFormat) override;
    AKRESULT Term(AK::IAkPluginMemAlloc* in_pAllocator) override;
    AKRESULT Reset() override;
    AKRESULT GetPluginInfo(AkPluginInfo& out_rPluginInfo) override;
    void Execute(AkAudioBuffer* io_pBuffer) override;
};

// MyEffect.cpp — registration
AK::IAkPlugin* CreateMyEffect(AK::IAkPluginMemAlloc* in_pAllocator) {
    return AK_PLUGIN_NEW(in_pAllocator, MyEffect());
}

AkPluginDesc MyEffectDesc = {
    AkPluginTypeEffect,
    AKCOMPANYID_MYCOMPANY,
    MY_EFFECT_ID,
    CreateMyEffect,
    CreateMyEffectParams
};
```

### Wwise memory allocation

```cpp
// BAD — bypasses Wwise memory tracking, and allocation must not happen in Execute()
float* buf = new float[numSamples];

// GOOD — use the plugin allocator passed at init, then reuse during Execute()
float* buf = (float*)AK_PLUGIN_ALLOC(m_pAllocator, numSamples * sizeof(float));
// free with:
AK_PLUGIN_FREE(m_pAllocator, buf);
```

### Wwise Execute() bypass handling

```cpp
void MyEffect::Execute(AkAudioBuffer* io_pBuffer) {
    if (io_pBuffer->uValidFrames == 0)
        return; // nothing to process

    // Check bypass: if bypassed, leave audio untouched and return
    // Wwise handles routing bypass at the bus level, but custom logic must not corrupt buffers
    const AkUInt32 numChannels = io_pBuffer->NumChannels();
    const AkUInt16 numFrames   = io_pBuffer->uValidFrames;

    for (AkUInt32 ch = 0; ch < numChannels; ++ch) {
        AkReal32* pData = io_pBuffer->GetChannel(ch);
        // process pData[0..numFrames-1]
    }
}
```

### Wwise RTPC parameter reading (thread-safe)

```cpp
// MyEffectParams.h — parameters are copied to the plugin before Execute()
// Wwise copies param struct on the audio thread before calling Execute().
// Always clamp before use.

void MyEffect::Execute(AkAudioBuffer* io_pBuffer) {
    auto* params = static_cast<MyEffectParams*>(m_pParams);
    float gain = AkClamp(params->fGain, 0.0f, 2.0f); // clamp RTPC value
    // use gain safely
}
```

---

## FMOD DSP Effect Structure

### Minimal DSP effect layout

```cpp
FMOD_DSP_DESCRIPTION myDspDesc = {
    FMOD_PLUGIN_SDK_VERSION,
    "MyDSP",          // name
    0x00010000,       // version
    1,                // input buffers
    1,                // output buffers
    MyDSP_create,     // create callback
    MyDSP_release,    // release callback
    MyDSP_reset,      // reset callback
    MyDSP_read,       // read/process callback
    nullptr,          // process callback (alternative to read)
    nullptr,          // setposition
    NUM_PARAMS,
    myParamDescs,
    MyDSP_setParam,
    MyDSP_getParam,
    MyDSP_shouldIProcess,
    nullptr,          // userdata
    nullptr, nullptr, nullptr
};
```

### FMOD memory allocation

```cpp
// BAD
MyState* state = new MyState();

// GOOD — use FMOD allocator in create/release callbacks, not in read/process
FMOD_DSP_STATE* dsp_state; // passed to create callback
MyState* state = (MyState*)dsp_state->functions->alloc(
    sizeof(MyState), FMOD_MEMORY_NORMAL, __FILE__);
// free with:
dsp_state->functions->free(state, FMOD_MEMORY_NORMAL, __FILE__);
```

### FMOD process callback (non-blocking)

```cpp
FMOD_RESULT MyDSP_read(FMOD_DSP_STATE* dsp_state,
                        float* inbuffer, float* outbuffer,
                        unsigned int length, int inchannels, int* outchannels) {
    // BAD: file I/O, mutex, sleep — never here
    // GOOD: pure computation only
    MyState* state = (MyState*)dsp_state->plugindata;
    for (unsigned int i = 0; i < length * inchannels; ++i)
        outbuffer[i] = inbuffer[i] * state->gain;
    return FMOD_OK;
}
```

### FMOD parameter indices

```cpp
// BAD — magic number hides the descriptor dependency
float gain;
FMOD_DSP_GetParameterFloat(dsp, 0, &gain, nullptr, 0);

// GOOD — one enum shared by descriptor setup and callbacks
enum ParamIndex {
    kParamGain,
    kParamCutoff,
    kParamCount
};

static FMOD_DSP_PARAMETER_DESC* paramDescs[kParamCount] = {
    &gainDesc,
    &cutoffDesc,
};

FMOD_RESULT setparameterfloat(FMOD_DSP_STATE* state, int index, float value) {
    auto* s = static_cast<MyState*>(state->plugindata);
    switch (index) {
        case kParamGain:   s->gain.store(value); return FMOD_OK;
        case kParamCutoff: s->cutoff.store(value); return FMOD_OK;
        default: return FMOD_ERR_INVALID_PARAM;
    }
}
```

---

## Unreal MetaSounds Thread Model

### Thread ownership

| Context | Thread | Safe operations |
|---------|--------|-----------------|
| `GetNextBuffer()` | AudioThread | Pure DSP, atomics, lock-free queues |
| `UAudioComponent` methods | GameThread | All UObject access |
| Parameter updates | GameThread | `SetFloatParameter`, `SetBoolParameter` |
| Blueprint/C++ game logic | GameThread | Everything UObject-related |

### Passing parameters GameThread → AudioThread

```cpp
// GameThread side — safe
MyAudioComponent->SetFloatParameter(FName("Gain"), NewGain);

// AudioThread side (GetNextBuffer) — BAD: accessing UAudioComponent
// float g = MyAudioComponent->GetFloatParam(); // CRASH — GameThread object

// GOOD — use MetaSounds parameter binding; the engine copies atomically
// Read from the FMetaSoundParameterTransmitter provided by the runtime
```

### Marshalling a one-shot event to AudioThread

```cpp
// From GameThread
AsyncTask(ENamedThreads::AudioThread, [this]() {
    // now safe to touch audio-thread state
    bTriggerOneShot.store(true, std::memory_order_release);
});
```

---

## Cross-middleware Thread Safety Checklist

- [ ] Setup/teardown allocations use the middleware allocator (`AkAlloc`, FMOD DSP allocator, or Unreal `FMemory`)
- [ ] No allocation, even through middleware allocators, occurs inside the audio callback
- [ ] No `std::mutex`, `new`, `malloc`, or I/O inside the audio callback
- [ ] Parameters are read from a thread-safe copy updated before the callback
- [ ] RTPC / parameter values are clamped to valid range before use
- [ ] Channel count is read dynamically, not hardcoded
- [ ] Bypass path leaves buffer contents unmodified
- [ ] All allocations made in `Init`/create are freed in `Term`/release
- [ ] Debug logging is guarded by `#ifndef AK_OPTIMIZED` or equivalent
