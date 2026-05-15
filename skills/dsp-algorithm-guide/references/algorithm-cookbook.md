# DSP Algorithm Cookbook

Key formulas, stability conditions, coefficient mappings, and numerical tips per algorithm family. These are reference stubs — not full implementations.

---

## Biquad (IIR) — Direct Form II Transposed

**Difference equation (Direct Form II transposed):**
```
y[n] = b0*x[n] + s1[n-1]
s1[n] = b1*x[n] - a1*y[n] + s2[n-1]
s2[n] = b2*x[n] - a2*y[n]
```
State variables `s1`, `s2` hold inter-sample memory. Transposed DF-II is compact and commonly used for floating-point biquads; cascaded SOS sections are safer than one high-order direct form.

**Stability condition:** both poles inside the unit circle. For a normalized denominator
`1 + a1*z^-1 + a2*z^-2`, the Jury checks are `1 + a1 + a2 > 0`,
`1 - a1 + a2 > 0`, and `1 - a2 > 0`. Re-check after every coefficient update.

**Parameter → coefficient mapping (Audio EQ Cookbook, R. Bristow-Johnson):**
```
w0 = 2*pi*fc/fs
alpha = sin(w0) / (2*Q)

Lowpass:  b0=(1-cos(w0))/2, b1=1-cos(w0), b2=(1-cos(w0))/2,  a0=1+alpha, a1=-2*cos(w0), a2=1-alpha
Highpass: b0=(1+cos(w0))/2, b1=-(1+cos(w0)), b2=(1+cos(w0))/2, [same a's]
Bandpass: b0=sin(w0)/2,     b1=0,           b2=-sin(w0)/2,    [same a's]
Peaking:  b0=1+alpha*A, b1=-2*cos(w0), b2=1-alpha*A, a0=1+alpha/A, a1=-2*cos(w0), a2=1-alpha/A
```
Normalise all coefficients by `a0` before storing.

**Numerical tips:**
- Compute coefficients in `double`, store as `float` only after normalisation.
- Add `1e-25f` DC offset or `JUCE_SNAP_TO_ZERO(s1); JUCE_SNAP_TO_ZERO(s2)` to kill denormals.
- For cascaded EQ bands, avoid arbitrary reordering if the user expects a specific response; when sections are mathematically interchangeable, place high-gain/high-Q sections later to reduce internal overload risk.

---

## State Variable Filter (SVF)

**State update (Andrew Simper TPT form):**
```
g  = tan(pi*fc/fs)
k  = 1/Q
a1 = 1 / (1 + g*(g + k))
a2 = g * a1
a3 = g * a2

v3 = x - ic2eq
v1 = a1*ic1eq + a2*v3
v2 = ic2eq + a2*ic1eq + a3*v3
ic1eq = 2*v1 - ic1eq
ic2eq = 2*v2 - ic2eq

lp = v2
bp = v1
hp = x - k*v1 - v2
```

**Stability condition:** stable for `0 < fc < Nyquist` and positive `Q` when coefficients remain finite. Clamp `fc` below Nyquist so `tan(pi*fc/fs)` does not explode.

**Numerical tips:** SVF is preferred over biquad when `fc` must be modulated at audio rate (e.g., filter FM) because stability is not endangered by rapid coefficient changes.

---

## Feedback Delay Network (FDN) Reverb

**Core update:**
```
// N delay lines of lengths L[0..N-1] (mutually prime, tuned to room size)
for each output i:
    y[i] = delayLine[i].read()
x_mixed = H * y               // H = Hadamard or Householder mixing matrix
for each line i:
    delayLine[i].write(x_mixed[i] * g[i] + input_diffused)
    // g[i] = decay gain for line i: g[i] = 10^(-3*L[i]/(fs*T60))
```

**Stability condition:** the feedback matrix multiplied by per-line gains must have spectral radius < 1. If the mixing matrix is unitary, this reduces to all loop gains `|g[i]| < 1`. For a target T60: `g[i] = 10^(-3*L[i]/(fs*T60))`.

**Parameter → coefficient mapping:**
- `T60` (reverb time in seconds) controls per-line decay gain `g[i]`.
- `fc` of a lowpass on each line controls high-frequency damping (shorter T60 at high freq).
- Modulate delay lengths ±0.1–1 % with a slow LFO to suppress metallic ringing.

**Numerical tips:** compute decay gains and modulation increments in `double`; use enough internal headroom to prevent feedback-matrix buildup from clipping.

---

## Dynamics — Level Detector + Gain Computer

**Level detection (RMS):**
```
// One-pole smoother with separate attack/release
input = x[n]^2
alpha = (input > env[n-1]) ? alpha_a : alpha_r
env[n] = alpha * env[n-1] + (1 - alpha) * input
rms[n] = sqrt(env[n])
// alpha_a = exp(-1/(fs*attackTime_s)); alpha_r = exp(-1/(fs*releaseTime_s))
```

**Peak detection:** replace `x[n]^2` with `|x[n]|`; remove the sqrt.

**Gain computer (soft-knee compressor):**
```
over = level_dB - threshold_dB
if over <= -knee/2:     gc = 0                            // below knee
elif over < knee/2:     gc = (1/ratio - 1) * (over + knee/2)^2 / (2*knee)
else:                   gc = over * (1/ratio - 1)
gainReduction_dB = gc
```

**Stability condition:** not applicable; purely feed-forward. Ensure `attackTime > 0` to avoid click on instantaneous gain changes.

**Numerical tips:** operate in dB domain for threshold/knee arithmetic; convert back to linear for the final gain multiplication. Use `std::max(level, 1e-10f)` before `log10` to avoid -Inf.

---

## Overlap-Add (OLA) FFT Convolution

**Block processing:**
```
// Partition IR into blocks of size N (= next power of 2 >= block_size + IR_block_size - 1)
for each input block of hop_size samples:
    zero-pad input to N; FFT → X
    multiply X * H_k (kth IR partition, pre-computed FFT)
    IFFT → time-domain product (length N)
    add first IR_block_size-1 samples to overlap buffer
    output hop_size samples; shift overlap buffer
```

**Latency:** `hop_size` samples (one block) for uniform partitioning; use non-uniform partitioning to reduce to a few samples at the cost of CPU.

**Stability condition:** not applicable; linear convolution is inherently stable.

**Numerical tips:** use `float` FFT for speed; accumulate the overlap buffer in `double` to avoid rounding artefacts when summing many partitions.

---

## Phase Vocoder (Pitch Shift / Time Stretch)

**Analysis → modification → synthesis:**
```
// Analysis
frame = windowed input (Hann, hop H, FFT size N)
FFT(frame) → magnitude[k], phase[k]
instantaneous_freq[k] = (phase[k] - prev_phase[k]) / H  (unwrapped, scaled to Hz)

// Modification
// Time stretch: output hop = H * stretch_factor
// Pitch shift:  resample in frequency domain (bin remapping)

// Synthesis
reconstruct phase from accumulated instantaneous_freq
IFFT + overlap-add
```

**Stability condition:** no feedback; stability is not a concern. Phase unwrapping errors cause "phasiness" artefacts, not instability.

**Numerical tips:** use `atan2f` for phase; wrap phase difference to `[-pi, pi]` before accumulating. For transient material, use a transient detector and switch to a bypass frame around the transient.

---

## PolyBLEP Oscillator

**Correction function (subtracted from naive waveform at discontinuity):**
```
// t = phase mod 1.0, dt = freq/sampleRate
// Apply at rising edge (phase wraps 0→1):
polyblep(t, dt):
    if t < dt:    u = t/dt;       return u + u - u*u - 1
    if t > 1-dt:  u = (t-1)/dt;   return u*u + u + u + 1
    else:         return 0.0

sawtooth[n] = (2*phase - 1) - polyblep(phase, dt)
square[n]   = (phase < 0.5 ? 1 : -1)
            + polyblep(phase, dt)
            - polyblep(fmod(phase + 0.5, 1.0), dt)
```

**Stability condition:** not applicable; no feedback path.

**Numerical tips:** maintain phase as a `double` accumulator to avoid phase drift at very low frequencies. PolyBLEP is first-order; for better alias suppression above ~10 kHz, use BLEP tables (sinc-based) or wavetable with mipmapping.
