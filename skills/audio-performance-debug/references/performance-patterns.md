# Performance Patterns — Diagnostic Reference

Concrete examples, fix details, and platform profiling workflows for each anti-pattern
in `audio-performance-debug`.

---

## O(N²) Algorithms

### Symptom
CPU time grows quadratically with voice count. Profile shows the voice-loop function
consuming most exclusive time.

### Example — naive voice × voice interaction
```cpp
// BAD: O(N^2) — every voice checks every other voice
for (int i = 0; i < numVoices; ++i)
    for (int j = 0; j < numVoices; ++j)
        voices[i].addModulation(voices[j].getOutput());
```

### Fix
```cpp
// GOOD: accumulate once, apply once — O(N)
float modSum = 0.f;
for (auto& v : voices) modSum += v.getOutput();
for (auto& v : voices) v.addModulation(modSum);
```

---

## Wrong FFT Size / Zero-Padding Waste

### Symptom
FFT takes longer than expected; spectral leakage; processing frame and FFT size are
unrelated primes.

### Fix choices
- Prefer power-of-two frame sizes wherever possible (`256`, `512`, `1024`, `2048`).
- If the frame must be non-power-of-two, use FFTW with `FFTW_MEASURE` — it factorizes
  automatically and finds the fastest plan.
- For JUCE: `juce::dsp::FFT` requires power-of-two; pad input if needed.

---

## Unnecessary Buffer Copies

### Example
```cpp
// BAD: copies entire buffer into a temporary, then processes
std::vector<float> temp(buffer.getNumSamples());
std::copy(channelData, channelData + numSamples, temp.data());
processInPlace(temp.data(), numSamples);
std::copy(temp.data(), temp.data() + numSamples, channelData);
```

### Fix
```cpp
// GOOD: process in-place on the host buffer
processInPlace(channelData, numSamples);
```

If an intermediate buffer is truly needed (e.g., overlap-add), allocate it once in
`prepareToPlay` as a member and reuse it.

---

## Container Lookups and Rehashing on the Audio Thread

### `std::map` — O(log N) with cache misses
```cpp
// BAD: tree traversal on every sample
float gain = gainMap[noteNumber];
```
```cpp
// GOOD: flat array lookup — O(1), cache-friendly
float gain = gainTable[noteNumber]; // pre-filled in prepareToPlay
```

### `std::unordered_map` — rehash triggers allocation
```cpp
// BAD: inserting at runtime causes rehash → heap alloc
frequencyMap[midiNote] = noteToHz(midiNote);
```
```cpp
// GOOD: pre-populate all 128 MIDI notes at init
void prepareToPlay(double sampleRate, int)
{
    for (int n = 0; n < 128; ++n)
        frequencyTable[n] = 440.f * std::pow(2.f, (n - 69) / 12.f);
}
```

---

## Heap Allocation on the Audio Thread

See `audio-dsp-review` for the full violation list. Common audio-performance-specific cases:

| Pattern | Fix |
|---------|-----|
| `voices.push_back(Voice())` at note-on | Pre-allocate a pool of `maxVoices` in `prepareToPlay`; activate by index |
| `std::string` label in DSP loop | Format the string on the UI thread; store as `std::array<char, N>` |
| `std::function` with lambda capture that allocates | Store as a plain function pointer + `void*` context |

---

## First-Touch Page Faults

### Symptom
Spike on the very first audio callback after `prepareToPlay`, then stable. Instruments
shows `vm_fault` in the first buffer.

### Fix
```cpp
void prepareToPlay(double sampleRate, int maxBlockSize)
{
    delayLine.resize(maxBlockSize * 4);
    // CRITICAL: touch every byte to force OS to map physical pages now
    std::fill(delayLine.begin(), delayLine.end(), 0.f);
}
```

---

## SIMD Vectorization

### When to apply
- Scalar loop over all samples with a simple arithmetic body (gain, mix, clip, apply envelope).
- Loop body has no data-dependent branches.
- Input/output arrays are 16-byte aligned (JUCE `AudioBuffer` is aligned).

### JUCE helpers
```cpp
juce::FloatVectorOperations::multiply(channelData, gainValue, numSamples);
juce::FloatVectorOperations::add(destData, srcData, numSamples);
juce::FloatVectorOperations::clip(channelData, numSamples, -1.f, 1.f);
```

### Manual SSE2 example (for custom inner loops)
```cpp
// Process 4 floats at once with SSE2
__m128 gainVec = _mm_set1_ps(gain);
for (int i = 0; i < numSamples; i += 4)
{
    __m128 x = _mm_loadu_ps(data + i);
    _mm_storeu_ps(data + i, _mm_mul_ps(x, gainVec));
}
```

### xsimd (cross-platform SIMD abstraction)
```cpp
using batch = xsimd::batch<float>;
constexpr std::size_t simdWidth = batch::size;
for (std::size_t i = 0; i < numSamples; i += simdWidth)
    xsimd::store_unaligned(data + i,
        xsimd::load_unaligned(data + i) * batch(gain));
```

---

## Denormal-Induced Slowdown

Subnormal (denormal) floats bypass the FPU fast path on x86 and ARM and can cause
100× slowdown in feedback loops (IIR filters, reverb tanks, envelope followers).

### Diagnostic
Profile shows time in the filter/feedback loop is disproportionate at low signal levels
(silent tail, slow release). Swap `float` state variables for `double` and observe whether
the slowdown moves — if yes, it is denormal-driven.

### Fix — set FTZ+DAZ at stream open
```cpp
void prepareToPlay(double, int)
{
    // x86 — flush denormals to zero (FTZ) and treat denormal inputs as zero (DAZ)
    _MM_SET_FLUSH_ZERO_MODE(_MM_FLUSH_ZERO_ON);
    _MM_SET_DENORMALS_ZERO_MODE(_MM_DENORMALS_ZERO_ON);
}
```

Or use JUCE's `ScopedNoDenormals` per block:
```cpp
void processBlock(AudioBuffer<float>& buffer, MidiBuffer&) override
{
    juce::ScopedNoDenormals noDenormals;
    // ... process ...
}
```

See `audio-numerics-review` for full denormal treatment.

---

## Lock Contention

See `audio-dsp-review` for the full lock-violation list. Performance-specific note:

A mutex between the UI and audio thread causes the audio thread to stall for the entire
duration the UI thread holds the lock. Under sustained load, this is a consistent floor
added to audio-thread time, not a spike — it appears as a baseline CPU elevation that
worsens when the UI is busy (e.g., repainting a spectrum analyzer).

### Fix patterns
| Pattern | Replacement |
|---------|-------------|
| `std::mutex` protecting a shared parameter | `std::atomic<float>` (lock-free for trivial types) |
| `std::mutex` protecting a shared struct | SPSC ring buffer (`moodycamel::ReaderWriterQueue`) |
| `std::condition_variable` involving audio thread | Audio thread polls a lock-free flag/queue; any waiting happens only on non-audio threads |

---

## Platform Profiling Workflows

### macOS — Instruments Time Profiler
1. Product → Profile (Cmd+I) in Xcode, or `xcrun xctrace record --template 'Time Profiler'`.
2. In the Time Profiler pane: right-click → "Focus on Thread" → select the CoreAudio I/O thread.
3. Enable "Invert Call Tree" to see exclusive hotspots (leaf functions) at the top.
4. Enable "Hide System Libraries" to suppress OS frames.
5. Look for your `processBlock` / `process` function in the top 5 entries.

### macOS — Instruments Allocations (finding audio-thread allocs)
1. Profile → Allocations template.
2. Add a "VM Tracker" instrument to catch page faults.
3. Filter the Allocations timeline to the audio thread.
4. Any spike during playback (not during `prepareToPlay`) is a violation.

### Linux — perf
```bash
# Record with call graph (dwarf for C++ template-heavy code)
perf record -g --call-graph dwarf -p <pid> -- sleep 10

# View hotspots
perf report --stdio --sort=dso,symbol

# Cache miss rate
perf stat -e cache-misses,cache-references,instructions -p <pid> -- sleep 5
```

### Windows — VTune
1. Open VTune → New Analysis → Hotspots.
2. Run with "Platform Profiler" to also capture hardware counters.
3. Filter the "Thread" dropdown to the WASAPI/ASIO realtime thread.
4. Sort by "CPU Time (Self)" — this is exclusive time.

### Tracy (cross-platform, in-process)
```cpp
// Mark the audio callback for per-buffer timing
void processBlock(AudioBuffer<float>& buf, MidiBuffer& midi) override
{
    ZoneScoped; // Tracy macro — zero cost when TRACY_ENABLE not defined
    // ... process ...
}
```
Connect Tracy Profiler GUI to the running process. Each frame = one buffer.
Frame-time histogram shows worst-case spike distribution directly.

### JUCE PerformanceCounter
```cpp
juce::PerformanceCounter counter("processBlock", 100); // log every 100 calls
void processBlock(...) override
{
    counter.start();
    // ... process ...
    counter.stop();
}
```
Logs min/max/avg to the JUCE logger — useful for a quick sanity check without a full profiler.
