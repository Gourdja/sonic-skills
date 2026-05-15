# VST3 Plugin Pitfalls Reference

Deep detail on the most common VST3 correctness issues and host compatibility quirks.

---

## Bus Arrangement Negotiation

### How it works

The host calls `setBusArrangements(inputs[], numIns, outputs[], numOuts)` and the plugin must either:
- Accept the arrangement by returning `kResultTrue` and applying the layout
- Reject it by returning `kResultFalse` (host then tries a different arrangement)

The plugin must never silently accept an arrangement it cannot honour.

### Common mistakes

**BAD — always returning `kResultTrue`:**
```cpp
tresult setBusArrangements(SpeakerArrangement* inputs, int32 numIns,
                            SpeakerArrangement* outputs, int32 numOuts) override {
    return kResultTrue; // accepts anything — process() will then misread channel count
}
```

**GOOD — validate and store:**
```cpp
tresult setBusArrangements(SpeakerArrangement* inputs, int32 numIns,
                            SpeakerArrangement* outputs, int32 numOuts) override {
    if (numIns == 1 && numOuts == 1 &&
        inputs[0] == SpeakerArr::kStereo &&
        outputs[0] == SpeakerArr::kStereo) {
        currentInputArrangement  = inputs[0];
        currentOutputArrangement = outputs[0];
        return kResultTrue;
    }
    return kResultFalse; // reject; host will try alternatives
}
```

### Checklist

| Item | Pass condition |
|------|---------------|
| All supported layouts explicitly listed | Mono, stereo, 5.1, etc. — not a catch-all `kResultTrue` |
| Negotiated arrangement stored | Used in `process()` to validate channel counts |
| `process()` guards channel access | `data.inputs[0].numChannels` checked before pointer access |
| Mono↔stereo upmix/downmix handled if offered | Declared capability matches actual DSP |

---

## Parameter ID Stability

### Requirements

`ParamID` values must be:
1. **Stable across plugin versions** — a saved project must restore the same parameter after an update
2. **Stable across save/load within one session** — IDs must not be generated at runtime from non-deterministic sources
3. **Unique within the plugin** — duplicate IDs cause silent parameter shadowing

### Common mistakes

**BAD — index as ID (breaks on reorder):**
```cpp
// params added in order; if order changes, all saved projects break
for (int i = 0; i < paramNames.size(); ++i)
    params[i].id = i; // fragile
```

**BAD — pointer cast as ID:**
```cpp
param.id = reinterpret_cast<ParamID>(this); // changes every run
```

**GOOD — fixed enum:**
```cpp
enum ParamIDs : ParamID {
    kParamGain    = 1000,
    kParamCutoff  = 1001,
    kParamResonance = 1002,
    // never reuse or renumber existing values
};
```

**GOOD — stable hash (when enum impractical for large sets):**
```cpp
ParamID stableId(const char* name) {
    // FNV-1a — deterministic, fast, no collision for typical param name sets
    uint32_t h = 2166136261u;
    while (*name) h = (h ^ (uint8_t)*name++) * 16777619u;
    return h;
}
```

---

## Latency Reporting

### What breaks when latency is wrong

- **Under-reported latency:** host applies less delay compensation than needed; plugin output arrives late relative to other tracks
- **Over-reported latency:** host pre-delays other tracks unnecessarily; adds unwanted latency to the mix
- **Changing latency without notification:** host's compensation is stale; timing drifts mid-session

### Implementation pattern

```cpp
// Return stable value; only change between processing sessions
int32 getLatencySamples() override {
    return static_cast<int32>(lookaheadSamples); // set in setupProcessing
}

// When latency must change at runtime (e.g. oversampling toggle):
tresult setActive(TBool state) override {
    if (state) {
        updateLatency(); // recalculate
        if (componentHandler)
            componentHandler->restartComponent(kLatencyChanged);
    }
    return kResultOk;
}
```

| Check | Pass condition |
|-------|---------------|
| `getLatencySamples()` overridden | Returns actual algorithmic latency in samples |
| Latency recalculated in `setupProcessing` | Accounts for current sample rate and oversampling factor |
| `restartComponent(kLatencyChanged)` called when latency changes | Host re-queries and updates compensation |

---

## Controller↔Processor Communication

### The split model

IComponent/IAudioProcessor live in one object (or DLL). IEditController lives separately and communicates via `IConnectionPoint::notify()` using `IMessage` objects. They share state through:

1. **`getState`/`setState`** on IComponent — binary blob for preset/project save
2. **`setComponentState`** on IEditController — controller mirrors processor state on load
3. **`IConnectionPoint::notify`** — runtime messages (e.g. VU meter values, parameter changes from audio thread)

### Common mistakes

| Mistake | Consequence |
|---------|-------------|
| Controller reads `setState` data but `setComponentState` reads different format | Parameter display stale after load |
| `notify()` called from audio thread with heap-allocated `IMessage` | Allocation on audio thread; potential lock |
| Controller modifies component state directly (bypassing `IConnectionPoint`) | Only works in single-process hosts; breaks in distributed/sandboxed scenarios |
| `performEdit` not called around programmatic parameter changes | Host automation recorder misses changes |

**GOOD — notify pattern for metering (audio → controller):**
```cpp
// In process() — use a lock-free atomic, drain on timer or idle
peakLevel.store(measuredPeak, std::memory_order_release);
// Separately, on main thread (timer callback):
if (connectionPoint)
    connectionPoint->notify(allocMessage(kMsgPeakLevel, peak));
```

---

## Bypass Handling

VST3 hosts commonly expose bypass in two ways:

1. **Soft bypass** — host automates a plugin parameter flagged with `ParameterInfo::kIsBypass`; plugin must handle it in `process()` and pass audio through cleanly
2. **Hard bypass** — host stops calling `process()` entirely; plugin has no control

### Requirements for soft bypass

```cpp
tresult process(ProcessData& data) override {
    bool bypassed = bypassParam.load();
    if (bypassed) {
        // Pass audio through — do NOT apply DSP
        for (int32 ch = 0; ch < data.inputs[0].numChannels; ++ch) {
            if (data.inputs[0].channelBuffers32[ch] != data.outputs[0].channelBuffers32[ch])
                memcpy(data.outputs[0].channelBuffers32[ch],
                       data.inputs[0].channelBuffers32[ch],
                       data.numSamples * sizeof(float));
        }
        // Still consume parameter changes to keep state consistent
        consumeParameterChanges(data.inputParameterChanges);
        return kResultOk;
    }
    // ... normal processing
}
```

---

## Offline Processing

When `ProcessModes::kOffline` is set in `ProcessSetup::processMode`, the plugin:

- May receive non-power-of-two or very large block sizes
- Should not rely on wall-clock time for tempo sync (use `ProcessContext`)
- Must not assume denormals are flushed (FTZ/DAZ flags may not be set)
- Should disable any realtime-dependent features (e.g. lookahead that changes with block size)

| Check | Pass condition |
|-------|---------------|
| `processMode` checked in `setupProcessing` or `process` | Offline path adjusts block-size-dependent state |
| No wall-clock `std::chrono` calls in offline path | Uses `ProcessContext::projectTimeSamples` instead |
| Tail handled correctly offline | Offline bouncer may ask for extra tail frames; `getTailSamples()` returns correct value |

---

## Host Compatibility Stress Cases

Avoid relying on DAW-specific folklore. Instead, test these spec-allowed host behaviours:

| Host behaviour to simulate | Mitigation |
|----------------------------|------------|
| `process()` called with `numSamples == 0` | Return successfully without dividing by block size or dereferencing empty buffers |
| Input and output buffers alias for in-place processing | Do not clear outputs before preserving aliased inputs |
| `setState` before activation or before the editor exists | Validate state blobs and store processor state independently of UI state |
| Controller and processor in separate components/processes | Communicate through `IConnectionPoint` / state transfer, never raw shared pointers |
| Latency or tail queried often | `getLatencySamples()` and `getTailSamples()` return cached values quickly |
| Offline / prefetch process modes | Check `ProcessData::processMode`; do not rely on wall-clock time |
