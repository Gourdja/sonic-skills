# Artifact Patterns — Diagnostic Reference

Deep-dive for each artifact type: diagnostic questions, common root causes, and
pointers to specific checks in `audio-dsp-review` and `audio-numerics-review`.

---

## Clicks / Pops

### Diagnostic questions
- Does it occur at a fixed interval matching the buffer size? → boundary discontinuity.
- Does the host xrun log fire at the same time? → CPU overload / late callback.
- Does it happen only at note-on/off or parameter change? → missing sample-accurate fade or
  discontinuous parameter update.
- Does it stop if you increase the buffer size? → xrun caused by insufficient processing time.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| Buffer boundary discontinuity | Last sample of buffer N and first sample of buffer N+1 differ because state is reset or uninitialized | Carry state across calls; never reset mid-stream without a fade |
| Uninitialized memory in tail | `memset` only the header, not the tail; `calloc` vs `malloc` confusion | `prepareToPlay`: `std::fill` entire state to zero |
| xrun (CPU overload) | Allocation, mutex, printf inside the audio callback | Apply `audio-dsp-review` — scan all three violation categories |
| Sample-accurate discontinuity | Parameter change applied at block boundary without interpolation | Use smoothed values (`SmoothedValue` in JUCE, manual lerp) |

### Checks in audio-dsp-review
- Memory allocation scan: any `new`/`delete`/`push_back` in the call graph.
- Lock scan: any `std::mutex`, `lock_guard`, `unique_lock` (including `std::try_to_lock` — the destructor's `unlock()` is a syscall), `condition_variable`, or spinlock without `try_lock`-only audio-thread usage reachable from callback.
- System call scan: any `printf`, `DBG()`, or file I/O.

---

## Crackling / Distortion

### Diagnostic questions
- Does it worsen as the signal level increases? → clipping or overflow.
- Does it appear only during silent decay (tail of a reverb, release of an envelope)? → denormals.
- Is it worse on ARM vs x86 or vice versa? → FTZ/DAZ differences between platforms.
- Is it only on fixed-point paths (embedded, SIMD int16)? → Q-format overflow.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| Clipping | Output exceeds ±1.0 before DAC/host limiter | Clamp at output: `std::clamp(y, -1.f, 1.f)` |
| Saturation / integer overflow | Q15 multiply product exceeds 32-bit without saturation | Use `__SSAT`, CMSIS saturating intrinsics, or promote to int64 before shift |
| Denormal floats | IIR state slowly decaying; FTZ not set | `audio-numerics-review`: denormal section; set FTZ+DAZ in `prepareToPlay` |
| Fixed-point accumulator wrap | int16 accumulator overflows on loud signals | Accumulate in int32 or int64; scale before output |

### Checks in audio-numerics-review
- Denormal section: verify `ScopedNoDenormals` or `_MM_SET_FLUSH_ZERO_MODE` at stream open.
- Overflow section: check Q-format multiply width and shift.
- FTZ/DAZ: confirm set on every thread that runs DSP (not just the main audio thread).

---

## Silence / Dropout

### Diagnostic questions
- Is the output all-zeros or NaN? (Check with `std::isnan` guard or scope.)
- Does it happen immediately or after a period of correct output? → gradual NaN propagation vs hard dropout.
- Does it correlate with CPU load or voice count? → xrun.
- Does it recover if you stop and restart the stream? → state corruption, not resource exhaustion.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| NaN propagation | `sqrt(-epsilon)`, division by zero, `acos` out of range | `audio-numerics-review`: NaN/Inf section; add `isnan` debug guards |
| xrun → callback skipped | Host drops the callback when the previous one overran | `audio-dsp-review`: eliminate all blocking calls |
| CPU overload | Too many voices; O(N²) algorithm | `audio-performance-debug`: profile and apply fix table |
| Buffer too small | Ring buffer underflow; FIFO drained before re-fill | Size ring buffer to worst-case latency, not average |

### Checks in audio-numerics-review
- NaN / Inf: every `sqrt`, `log`, `acos`, `asin`, `pow` call — are inputs clamped?
- Division: is every divisor guarded against zero?

---

## DC Offset

### Diagnostic questions
- Is DC present from the very first output sample? → uninitialized state (filter memory, delay line).
- Does DC appear only after a long period of near-silence? → FTZ leaving state near zero without flush.
- Does it drift slowly? → precision loss in a long accumulator, or missing high-pass filter.
- Is it in both channels equally? → common-mode; check mono processing path.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| Uninitialized filter state | `float state;` declared but not zeroed | `prepareToPlay`: `state = 0.f` on every reset |
| Missing high-pass filter | Integration or leaky accumulator with no DC-blocking | Insert first-order high-pass (pole at ~5 Hz) after any integrator |
| FTZ leaving near-zero residue | Denormal flushed to zero mid-computation, leaving DC bias | Apply DC-blocking after feedback loops; use `ScopedNoDenormals` |
| Precision loss in accumulator | `float` sum of many small values in a long loop | `audio-numerics-review`: precision-loss section; use `double` accumulator |

### Checks in audio-numerics-review
- Precision loss: any `float` accumulator running for thousands of samples.
- FTZ/DAZ: confirm DC-blocking is present after all feedback paths.

---

## Aliasing

### Diagnostic questions
- Do you hear mirror tones that are pitch-symmetric around Nyquist? → missing anti-alias filter.
- Is aliasing present at all sample rates or only at low ones? → hard-coded frequency rather than normalized.
- Is it from an oscillator? → waveform not bandlimited (naive saw/square with harmonics above Nyquist).
- Is it from a downsampler? → no low-pass filter before decimation.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| Oscillator without bandlimit | Naive wavetable or direct synthesis of saw/square | Use BLEP/BLAM, PolyBLEP, or pre-generated wavetables bandlimited per octave |
| No anti-alias before downsampling | Decimate by N without a low-pass at Fs/(2N) | Insert half-band FIR or elliptic LP before decimation |
| Hard-coded frequency | `cutoff = 1000.0f` instead of `cutoff / sampleRate` | Normalize all frequency parameters to [0, 0.5] |
| Resampler quality too low | Linear interpolation in a pitch-shifter | Use a windowed-sinc or polyphase resampler |

---

## Phase Issues

### Diagnostic questions
- Does the stereo image collapse when summed to mono? → polarity inversion or comb filtering.
- Is there a comb filtering "hollow" sound? → unmatched latency on L and R or on parallel paths.
- Does the problem appear only in mid-side processing? → incorrect M/S encode or decode matrix.
- Does it happen only with certain plugin chain orders? → latency compensation mismatch.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| Latency mismatch on parallel paths | Two branches with different buffer delays | Match delays with a delay compensation buffer; use `getLatencySamples()` |
| Polarity inversion | Accidental negation or wrong sign in matrix | Check M/S encode: M=L+R, S=L-R (pre-scaled by 0.5) |
| All-pass not matched | One channel has an all-pass that the other lacks | Apply all-pass to both channels symmetrically |
| Plugin reports wrong latency | `getLatencySamples()` returns zero for a non-zero-latency plugin | Compute and report true latency; update in `prepareToPlay` |

### Checks in audio-dsp-review
- Symmetry: does every transformation applied to the left channel also apply to the right?
- Latency reporting: does `getLatencySamples()` account for all internal delays?

---

## Latency / Delay Compensation

### Diagnostic questions
- Is the wet signal delayed relative to the dry in a parallel chain? → plugin not reporting latency.
- Does the problem change with buffer size? → latency reported in samples but computed from seconds using wrong rate.
- Is compensation correct in one DAW but not another? → plugin assumes a specific buffer size.
- Does it appear only after a stream restart? → latency cached at construction, not updated in `prepareToPlay`.

### Root causes

| Cause | Code pattern | Fix |
|-------|-------------|-----|
| `getLatencySamples()` not implemented | Returns 0 for a plugin with internal delay (look-ahead, resampler) | Compute as `ceil(lookaheadSeconds * sampleRate)` and return it |
| Latency computed before `prepareToPlay` | Uses a hardcoded sample rate or buffer size | Recalculate and call `setLatencySamples()` every `prepareToPlay` |
| Buffer size assumed constant | Latency correct at 512 samples but wrong at 256 | Use `maximumExpectedSamplesPerBlock` from `prepareToPlay`, not a constant |
