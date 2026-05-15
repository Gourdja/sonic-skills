# Numerical Pitfalls in Audio DSP

Reference for the `audio-numerics-review` skill. Applies to all platforms; platform-specific
notes are called out explicitly. The overriding rule: **floating-point is not real math**.
IEEE 754 has finite precision, special values (NaN, ±Inf, ±0, subnormals), and
non-associative arithmetic. These properties interact with audio code in predictable ways
once you know where to look.

## Table of Contents
1. [Denormal Floats](#1-denormal-floats)
2. [NaN and Inf Propagation](#2-nan-and-inf-propagation)
3. [Catastrophic Cancellation](#3-catastrophic-cancellation)
4. [Fixed-Point Overflow and Scaling](#4-fixed-point-overflow-and-scaling)
5. [Precision Loss in Accumulation](#5-precision-loss-in-accumulation)
6. [FTZ/DAZ Configuration by Platform](#6-ftzdaz-configuration-by-platform)
7. [Compound Pitfalls](#7-compound-pitfalls)
8. [Audit Checklist](#8-audit-checklist)

---

## 1. Denormal Floats

### Mechanism

IEEE 754 single-precision uses a 23-bit mantissa plus an implicit leading 1-bit,
giving values from `±1.18e-38` (minimum normalized) to `±3.40e+38`. Below `1.18e-38`
the leading bit is zero — this is the *subnormal* (denormal) range, from `1.40e-45`
to `1.17e-38`.

Hardware handles normalized floats in fast silicon paths. Subnormals are handled in
microcode on x86, causing 10–100× slowdown per operation. On ARM Cortex-A the penalty
is similar unless FZ (Flush-to-Zero) is set in FPCR.

### Why audio code generates them

A feedback loop with gain < 1 lets the signal decay. Each sample is multiplied by a
coefficient slightly less than 1. When the magnitude crosses `1.18e-38`, the CPU enters
the slow microcode path, but processing *continues* — just extremely slowly.

```
Sample magnitude: 1.0 → 0.5 → 0.25 → ... → 1e-20 → 1e-30 → 1e-38 → [subnormal range]
                                                                          ^^^^^^^^^
                                                              10-100x slowdown here
```

After a few hundred samples below the subnormal threshold, the audio callback starts
missing its deadline: a guaranteed glitch.

### Patterns that trigger denormals

```cpp
// IIR filter with decaying signal
float y = a1 * x + a2 * z1 - b1 * z2;  // z1, z2 decay into denormal range

// Reverb tank: feedback coefficient < 1 with no excitation
float tank = feedback * delayLine[pos];  // decays over several seconds

// Envelope follower at silence
envelope *= releaseCoeff;  // releases toward 0.0 through subnormal region

// Allpass chain (common in reverb)
float v = in - coeff * s;
float out = coeff * v + s;
s = v;  // s decays to subnormal when in goes silent
```

### Safe patterns

**Option A — FTZ + DAZ (preferred, lowest overhead)**

Set the processor's floating-point mode at stream-open time, or use an RAII guard inside each
audio callback. See Section 6 for platform details.

```cpp
// JUCE — RAII guard for the duration of the audio callback
void processBlock(juce::AudioBuffer<float>& buffer, juce::MidiBuffer&) {
    juce::ScopedNoDenormals noDenormals;
    // ... process feedback paths here
}
```

**Option B — DC-offset clamp (hardware-independent)**

Injects a tiny DC offset that prevents the value reaching the denormal range without
affecting audio quality (offset is below the noise floor of 24-bit digital audio).

```cpp
// DC_CLAMP must be larger than the subnormal threshold but far below any audible signal.
// 1e-25 is about -500 dBFS, far below 24-bit quantization noise.
static constexpr float DC_CLAMP = 1e-25f;

float process(float x) {
    float y = b0 * x + b1 * z1 + b2 * z2 + DC_CLAMP;
    z2 = z1;
    z1 = y - DC_CLAMP;  // remove the offset from the state (optional — often negligible)
    return y;
}
```

**Option C — explicit zero check on state (for sparse cases)**

```cpp
inline float flushDenormal(float x) {
    return (std::abs(x) < 1e-37f) ? 0.f : x;
}
z1 = flushDenormal(z1);
z2 = flushDenormal(z2);
```

---

## 2. NaN and Inf Propagation

### Mechanism

IEEE 754 defines:
- `0.0 / 0.0` → NaN (quiet or signalling depending on hardware)
- `x / 0.0` for x ≠ 0 → ±Inf
- `sqrt(x)` for x < 0 → NaN
- `log(x)` for x ≤ 0 → NaN or -Inf
- `asin(x)`, `acos(x)` for |x| > 1 → NaN

NaN is *infectious*: any arithmetic operation with NaN produces NaN. Inf is similar:
`Inf + Inf` = Inf, `Inf - Inf` = NaN, `0 * Inf` = NaN. Both propagate silently — the
CPU does not signal an exception by default. After one NaN enters the audio path, every
subsequent output sample is NaN, which most DAW hosts will either silence (clip to 0) or
not — either way the output is wrong.

### Common sources in audio code

**Division by variable that can be zero:**
```cpp
// Envelope follower normalizer
float gain = peak > 0.f ? target / peak : 1.f;  // correct
float gain = target / peak;                      // BAD — Inf when peak == 0

// Pitch detector autocorrelation peak
float period = 1.0f / lagAtPeak;  // BAD — lagAtPeak can be 0 if no pitch found

// Band-limited oscillator sinc
float sinc = std::sin(M_PI * x) / (M_PI * x);   // BAD — 0/0 when x == 0
float sinc = (x == 0.f) ? 1.f : std::sin(M_PI * x) / (M_PI * x);  // correct
```

**Square root of a possibly-negative value:**
```cpp
// RMS — sumSq can be tiny-negative due to float rounding
float rms = std::sqrt(sumSq / N);                  // BAD — NaN if sumSq/N == -epsilon
float rms = std::sqrt(std::max(0.f, sumSq / N));   // correct

// Magnitude of complex FFT bin
float mag = std::sqrt(re*re + im*im);  // always non-negative — this is safe
```

**Inverse trig outside domain:**
```cpp
// Stereo correlation meter
float r = dot / (rmsL * rmsR);
float angle = std::acos(r);                              // BAD — NaN if |r| > 1.0
float angle = std::acos(std::clamp(r, -1.f, 1.f));      // correct
```

### Defensive patterns

```cpp
// 1. Input validation at the entry point of processing
void processBlock(float* in, float* out, int n) {
    for (int i = 0; i < n; i++) {
        // Clamp out-of-range input (e.g. from an uncalibrated ADC or plugin chain)
        in[i] = std::clamp(in[i], -2.f, 2.f);
    }
    // ...
}

// 2. Debug assertions before propagation
#ifndef NDEBUG
#define ASSERT_FINITE(x) assert(!std::isnan(x) && !std::isinf(x))
#else
#define ASSERT_FINITE(x)
#endif

// 3. Recover at output (last resort — indicates a bug upstream)
float safeOut = std::isfinite(y) ? y : 0.f;
```

---

## 3. Catastrophic Cancellation

### Mechanism

IEEE 754 float has 24 bits of mantissa (single) or 53 bits (double). When two nearly-equal
values are subtracted, the leading significant bits cancel, and the result is dominated by
the rounding error in the operands. The relative error of the result can be enormous.

Example:
```
a = 1.000001 (stored as 1.0000010)
b = 1.000000 (stored as 1.0000000)
a - b = 0.0000010  — only 2 significant bits remain (7 were consumed by cancellation)
```

### Audio patterns where it occurs

**Naive DC-blocking one-pole filter:**
```cpp
float y = x - x_prev + R * y_prev;
// When x is near-DC, x ≈ x_prev — the difference loses bits.
```

**Direct-form biquad near Nyquist:**

For a high-shelf filter with cutoff near Nyquist, the bilinear transform produces
coefficients like `b0 ≈ 1.0, b1 ≈ -1.9999, b2 ≈ 0.9999`. The sum `b0*x[n] + b1*x[n-1] + b2*x[n-2]`
involves near-equal positive and negative terms — all significant bits cancel.

**First-order finite difference of slowly varying signal:**
```cpp
float velocity = (pos - prevPos) / dt;
// If pos and prevPos differ by less than epsilon relative to their magnitude, result is noise.
```

### Safe alternatives

**Use double for coefficient calculation:**
```cpp
// Compute biquad coefficients in double, store in float, process in float
double omega = 2.0 * M_PI * (double)cutoffHz / (double)sampleRate;
double cos_w = std::cos(omega);
double alpha = std::sin(omega) / (2.0 * Q);
double a0_inv = 1.0 / (1.0 + alpha);
b0 = (float)((1.0 - cos_w) / 2.0 * a0_inv);  // less catastrophic in double
b1 = (float)( (1.0 - cos_w)       * a0_inv);
b2 = b0;
a1 = (float)(-2.0 * cos_w         * a0_inv);
a2 = (float)( (1.0 - alpha)        * a0_inv);
```

**Use numerically stable filter structures:**

Direct form II transposed is more numerically stable than direct form I for biquads.
For extremely high-frequency filters, use the SVF (state-variable filter) topology which
avoids the near-cancellation inherent in direct-form coefficient sets.

```cpp
// State variable filter — no catastrophic cancellation in coefficient set
struct SVF {
    float ic1eq = 0.f, ic2eq = 0.f;
    void process(float x, float g, float k, float& lp, float& bp, float& hp) {
        float v1 = (ic1eq + g * (x - ic2eq)) / (1.f + g * (g + k));
        float v2 = ic2eq + g * v1;
        ic1eq = 2.f * v1 - ic1eq;
        ic2eq = 2.f * v2 - ic2eq;
        lp = v2; bp = v1; hp = x - k * v1 - v2;
    }
};
```

**Double-precision processing path (nuclear option):**
```cpp
// For a single critical filter where precision matters most
double process(double x) {
    double y = b0d * x + b1d * z1d + b2d * z2d;
    z2d = z1d; z1d = y;
    return y;
}
```

---

## 4. Fixed-Point Overflow and Scaling

### Q-format basics

A Q15 number is a signed 16-bit integer representing a value in [-1, 1) with 15 fractional
bits. The full range is -32768 to 32767 → [-1.0, 0.99997).

Multiplying two Q15 values produces a Q30 result in 32 bits. The intermediate value must
be shifted right by 15 and then either truncated or saturated back to Q15.

### Overflow patterns

```cpp
// BAD — int16 * int16 intermediate overflows int16
int16_t coeff = 16384;   // 0.5 in Q15
int16_t state = 32767;   // ~1.0 in Q15
int16_t result = (coeff * state) >> 15;  // coeff*state = 536854528 > INT16_MAX — UB

// GOOD — promote to int32 before multiply
int16_t result = (int16_t)(((int32_t)coeff * state) >> 15);

// BAD — accumulating many Q15 products into int32 eventually overflows
int32_t acc = 0;
for (int i = 0; i < 128; i++)
    acc += (int32_t)coeff[i] * state[i];  // 128 * (32767*32767) = 1.4e11 > INT32_MAX
// Fix: use int64_t for large accumulations, or scale coefficients

// BAD — SIMD multiply without widening
__m128i a = _mm_set1_epi16(0x7FFF);
__m128i b = _mm_set1_epi16(0x7FFF);
__m128i result = _mm_mullo_epi16(a, b);  // lower 16 bits only — wrong

// GOOD — SIMD widening multiply (produces int32 from int16)
__m128i lo = _mm_mullo_epi16(a, b);   // lower 16 bits
__m128i hi = _mm_mulhi_epi16(a, b);   // upper 16 bits (arithmetic shift right behavior)
__m128i wide_lo = _mm_unpacklo_epi16(lo, hi);  // int32 lane
__m128i wide_hi = _mm_unpackhi_epi16(lo, hi);
```

### Saturation arithmetic

Many DSP targets provide saturating instructions that clamp to INT_MIN/INT_MAX instead of
wrapping. Always prefer these for audio code.

```cpp
// ARM CMSIS-DSP Q15 multiply with saturation
arm_mult_q15(pSrcA, pSrcB, pDst, blockSize);

// ARM intrinsic saturating add
int16_t result = __QADD16(a, b);  // saturates at ±32767

// GCC/Clang __builtin_add_overflow for detecting overflow in portable code
int32_t result;
if (__builtin_add_overflow(a, b, &result)) {
    result = (a > 0) ? INT32_MAX : INT32_MIN;  // manual saturation
}

// Portable saturating add
int32_t saturatingAdd(int32_t a, int32_t b) {
    int64_t r = (int64_t)a + b;
    if (r > INT32_MAX) return INT32_MAX;
    if (r < INT32_MIN) return INT32_MIN;
    return (int32_t)r;
}
```

### Bit-growth in FIR filters

A length-N FIR filter in Q15 requires log2(N) extra bits of headroom in the accumulator
to avoid overflow. A 128-tap FIR needs 7 extra bits → use Q22 accumulator (int32 with
7 bits of headroom above Q15).

```cpp
// 64-tap Q15 FIR — needs 6 extra bits (log2(64) = 6)
// Accumulate into int32 (Q30 = Q15 * Q15), then shift right by 15 + 6 = 21
int32_t acc = 0;
for (int i = 0; i < 64; i++)
    acc += (int32_t)coeffs[i] * state[i];
output = (int16_t)(acc >> 21);  // correct scaling for 64 taps
```

---

## 5. Precision Loss in Accumulation

### Mechanism

Each floating-point addition produces a rounding error of approximately `ε * |result|`
where `ε = 1.19e-7` for float. Summing N values in a naive loop accumulates
O(N · ε · max|xᵢ|) total error. For N = 10,000 and float, the error is ~0.1%.

In audio: summing many channels at unity gain for a 10ms block at 48kHz = 480 samples.
For float, the accumulated RMS error is ~57 dB below full-scale — below the noise floor.
But for energy/RMS calculations where the individual samples are squared, and the result
must be accurate to determine gain in a limiter, this error matters.

### Naive summation error example

```cpp
// Compute energy (sum of squares) over 1 second at 48kHz
float sumSq = 0.f;
for (int i = 0; i < 48000; i++) {
    sumSq += buf[i] * buf[i];  // rounding error accumulates across 48000 additions
}
// Error: O(48000 * 1.19e-7) ≈ 0.57% — may cause incorrect gain decisions in a limiter
```

### Double accumulator (recommended)

```cpp
// Same loop with double accumulator — error: O(48000 * 2.22e-16) ≈ negligible
double sumSq = 0.0;
for (int i = 0; i < 48000; i++) {
    double x = buf[i];
    sumSq += x * x;
}
float result = (float)sumSq;
```

Cost: scalar double arithmetic is often close to float on modern desktop CPUs, but SIMD double
has half as many lanes as SIMD float and may matter in wide inner loops. Profile before changing
large per-sample kernels.

### Kahan compensated summation

Maintains a running error correction term. Achieves machine-precision accuracy for float
at the cost of ~2× the arithmetic. Preferred when double is unavailable (e.g., GPU shaders,
certain embedded targets).

```cpp
// Kahan summation algorithm
float kahanSum(const float* data, int n) {
    float sum = 0.f;
    float c   = 0.f;  // running compensation
    for (int i = 0; i < n; i++) {
        float y = data[i] - c;    // compensated input
        float t = sum + y;        // new sum (may lose low bits of y)
        c = (t - sum) - y;        // recover what was lost
        sum = t;
    }
    return sum;
}

// Kahan-Babuska-Neumaier variant — handles the case where |y| > |sum|
float kbnSum(const float* data, int n) {
    float sum = 0.f, c = 0.f;
    for (int i = 0; i < n; i++) {
        float t = sum + data[i];
        c += (std::abs(sum) >= std::abs(data[i]))
             ? (sum - t) + data[i]   // standard Kahan
             : (data[i] - t) + sum;  // handles large data[i]
        sum = t;
    }
    return sum + c;
}
```

### Pairwise summation (SIMD-friendly)

Recursively sum pairs. Error is O(log N · ε) instead of O(N · ε). Compatible with SIMD
vectorisation.

```cpp
float pairwiseSum(const float* data, int n) {
    if (n <= 16) {
        // Base case: use double to accumulate small block
        double s = 0.0;
        for (int i = 0; i < n; i++) s += data[i];
        return (float)s;
    }
    int half = n / 2;
    return pairwiseSum(data, half) + pairwiseSum(data + half, n - half);
}
```

---

## 6. FTZ/DAZ Configuration by Platform

FTZ (Flush-to-Zero): subnormal *results* are flushed to ±0.
DAZ (Denormals-Are-Zero): subnormal *inputs* are treated as ±0 before any arithmetic.

On x86/x64, set both. FTZ alone does not protect against subnormal *inputs* arriving from
external sources (e.g. a delay line read from a previous block). Other architectures expose
different controls.

### x86 / x64 (SSE, SSE2)

The MXCSR register controls FTZ (bit 15) and DAZ (bit 6) for SSE/AVX arithmetic. Note:
x87 floating-point unit has its own control word and does not support FTZ/DAZ — avoid
x87 in audio code (compile with `-msse2 -mfpmath=sse` on GCC/Clang).

```cpp
#include <xmmintrin.h>  // SSE
#include <pmmintrin.h>  // SSE3 (DAZ)

// Set both FTZ and DAZ
void setDenormalProtection() {
    _MM_SET_FLUSH_ZERO_MODE(_MM_FLUSH_ZERO_ON);         // FTZ
    _MM_SET_DENORMALS_ZERO_MODE(_MM_DENORMALS_ZERO_ON); // DAZ
}

// Read current state (useful for saving/restoring around nested calls)
unsigned int saved = _mm_getcsr();
_mm_setcsr(saved | _MM_FLUSH_ZERO_ON | _MM_DENORMALS_ZERO_ON);
// ... processing
_mm_setcsr(saved);  // restore
```

### ARM (AArch32 / AArch64)

ARM uses the FPSCR register (AArch32) or FPCR (AArch64). The FZ bit (bit 24) enables
FTZ. There is no separate DAZ bit — FZ flushes both inputs and results to zero.

```cpp
// AArch64 — set FZ bit via inline assembly
void setDenormalProtectionARM64() {
    uint64_t fpcr;
    __asm__ volatile("mrs %0, fpcr" : "=r"(fpcr));
    fpcr |= (1ULL << 24);  // FZ bit
    __asm__ volatile("msr fpcr, %0" : : "r"(fpcr));
}

// AArch32 (Cortex-A, NEON)
void setDenormalProtectionARM32() {
    uint32_t fpscr;
    __asm__ volatile("vmrs %0, fpscr" : "=r"(fpscr));
    fpscr |= (1U << 24);  // FZ bit
    __asm__ volatile("vmsr fpscr, %0" : : "r"(fpscr));
}
```

JUCE's `ScopedNoDenormals` handles both x86 and ARM correctly.

### Emscripten / WebAssembly

WebAssembly does not expose MXCSR or FPCR registers. Denormal handling depends on the
underlying CPU (the browser's JS engine), but there is no portable API to control it.

Options:
1. Compile with `-ffast-math` — permits the compiler to flush denormals (not guaranteed).
2. Use a software denormal clamp explicitly in the code.
3. On Chrome/Firefox running x86 hardware, FTZ is typically set by the JS engine — but
   this is implementation-defined and may change.

```cpp
// Portable software clamp — safe for all WASM targets
inline float flushDenormal(float x) {
    const float threshold = std::numeric_limits<float>::min();  // 1.18e-38
    return (std::abs(x) < threshold) ? 0.f : x;
}

// Apply to all feedback state variables at the end of each callback
z1 = flushDenormal(z1);
z2 = flushDenormal(z2);
```

### iOS (Apple Silicon / Arm)

iOS audio callbacks (RemoteIO, AUv3) run on the main audio thread. The FZ bit in FPCR is
typically set by CoreAudio on Apple Silicon for audio callbacks, but this is undocumented.
Always set it explicitly.

```cpp
// iOS AUv3 — set in `allocateRenderResourcesAndReturnError`
- (BOOL)allocateRenderResourcesAndReturnError:(NSError**)error {
    [super allocateRenderResourcesAndReturnError:error];
    setDenormalProtectionARM64();  // see AArch64 code above
    return YES;
}
```

### Comparison table

| Platform | Register | FTZ bit | DAZ bit | JUCE coverage |
|----------|----------|---------|---------|---------------|
| x86 SSE | MXCSR | bit 15 | bit 6 | Yes |
| x64 SSE | MXCSR | bit 15 | bit 6 | Yes |
| AArch64 | FPCR | bit 24 | (same as FTZ) | Yes |
| AArch32 | FPSCR | bit 24 | (same as FTZ) | Yes |
| WASM | — | — | — | No — manual clamp needed |
| iOS (A-series) | FPCR | bit 24 | (same as FTZ) | Yes |

---

## 7. Compound Pitfalls

Real-world bugs often combine multiple issues from the above categories.

### Reverb that glitches after silence

**Root cause chain:** signal decays → IIR state hits subnormal range → CPU slows down
100× → audio callback misses deadline → dropout. Simultaneously: reverb feedback path
with coefficient > 1.0 (or NaN coefficient from bad parameter update) → Inf → NaN in output.

**Fix:**
1. Set FTZ+DAZ in `prepareToPlay`
2. Assert coefficient bounds on parameter set: `assert(feedback >= 0.f && feedback < 1.f)`
3. Add `ASSERT_FINITE` on reverb output in debug builds

### Limiter that produces NaN after silence

**Root cause chain:** signal is silent for several blocks → RMS = 0 → gain = target/rms
= Inf → output *= Inf → NaN.

**Fix:**
```cpp
float rms = std::sqrt(std::max(0.f, sumSq / n));
float gain = (rms > 1e-6f) ? (targetLevel / rms) : 1.0f;
```

### Pitch detector with spurious octave errors at onset

**Root cause chain:** autocorrelation sum uses float → precision loss at low lag values
(high frequency, many samples) → peak detection selects wrong lag → octave error.

**Fix:** compute autocorrelation in double or use Kahan summation for the inner product.

### Fixed-point IIR filter that clips unpredictably

**Root cause chain:** Q15 coefficients, Q15 state, multiply produces Q30 → stored in int16
without saturation → wraps → aliasing distortion.

**Fix:** accumulate into int32, apply `__SSAT(acc >> 15, 16)` once per sample.

---

## 8. Audit Checklist

```
Denormals
[ ] FTZ+DAZ set in prepareToPlay/reset/stream-open
[ ] Or: DC-offset clamp applied to all feedback state variables
[ ] No WASM target relying on host FTZ (use software clamp)
[ ] JUCE: ScopedNoDenormals used or FTZ set globally per processor

NaN / Inf
[ ] Every division guarded: denom > epsilon before dividing
[ ] sqrt(x): always sqrt(max(0.f, x))
[ ] acos/asin: input clamped to [-1, 1] before call
[ ] log/log2/log10: input clamped to > 0 before call
[ ] Debug: ASSERT_FINITE after each major processing stage
[ ] Input validation at processBlock entry for untrusted plugin chains

Catastrophic Cancellation
[ ] Biquad coefficients computed in double when near Nyquist
[ ] Direct-form biquad replaced by SVF/TPT for extreme frequency settings
[ ] DC-blocking implemented with biquad, not naive first-difference

Fixed-point
[ ] All Q-format multiplies promote to wider type before result
[ ] Accumulators sized for worst-case bit growth (log2(N) extra bits for N-tap FIR)
[ ] Saturation used instead of wrap-on-overflow for intermediate results
[ ] SIMD int16 multiplies use widening instructions (_mm_mulhi_epi16, vmull_s16)

Accumulation Precision
[ ] Energy/RMS sums use double accumulator or Kahan summation
[ ] Long mixing loops (>256 channels) use double accumulator
[ ] Coefficient sums for FIR normalization computed in double

General
[ ] No magic numeric constants without comments explaining origin/units
[ ] Sample rate stored as double in coefficient computation
[ ] All numeric paths exercised in unit tests with edge-case inputs (silence, full-scale,
    1Hz sine, Nyquist sine)
```
