# DSP Math Reference

Dense formula tables and gotchas. Use as a lookup when SKILL.md's quick answers aren't enough.

---

## Fourier Transform / DFT / FFT

**Intuition:** Every finite signal can be expressed as a sum of sine waves at integer multiples of a base frequency.

**Key formula:**
```
X[k] = sum_{n=0}^{N-1} x[n] * e^{-j2pi*k*n/N}    (DFT)
```

**Properties / gotchas:**
- Output is complex: magnitude = `|X[k]|`, phase = `atan2(Im, Re)`
- Bins 0..N/2 are meaningful; bins N/2+1..N-1 are conjugate mirrors (for real input)
- Bin k represents frequency `f = k * sampleRate / N`
- DC lives at bin 0; Nyquist lives at bin N/2
- FFT is O(N log N); DFT is O(N^2) — always use FFT for N > ~32
- Zero-padding in time domain → interpolation in frequency domain (not more resolution)
- True frequency resolution = `sampleRate / N` — increase N to resolve closer partials

**Audio code connection:** `fftSize` sets N; `hopSize` sets overlap in STFT; magnitude spectrum = `std::abs(X[k])`.

---

## Convolution

**Intuition:** The output at each sample is a weighted sum of the input's past values, where the weights are the impulse response.

**Key formula:**
```
y[n] = (x * h)[n] = sum_{k=0}^{M-1} h[k] * x[n-k]
```

**Properties / gotchas:**
- LTI (Linear Time-Invariant) systems are fully described by their impulse response h
- Convolution in time = multiplication in frequency domain (use FFT for long IRs)
- Output length = `len(x) + len(h) - 1` — allocate accordingly or you'll alias
- Overlap-add / overlap-save: standard techniques for block-based convolution reverb
- Circular convolution (raw IFFT of product) wraps around — use zero-padding to get linear conv

**Audio code connection:** FIR filter loop is direct convolution; IR reverb uses FFT-based partitioned convolution (`juce::dsp::Convolution`).

---

## Z-Transform

**Intuition:** The z-transform is the discrete-time equivalent of the Laplace transform — it lets you reason about filter stability and frequency response using poles and zeros on the complex plane.

**Key formula:**
```
H(z) = sum_{n=0}^{inf} h[n] * z^{-n}
     = B(z) / A(z)    (rational for IIR)
```

**Properties / gotchas:**
- Evaluate on the unit circle (`z = e^{jwT}`) to get the frequency response
- Poles inside the unit circle → stable filter; poles on or outside → unstable / ringing
- Pole radius r sets decay rate; angle theta sets resonant frequency: `f = theta * sr / (2*pi)`
- Zeros cancel frequencies; poles boost (or ring at) frequencies
- Biquad has 2 poles, 2 zeros — enough for most EQ/filter shapes
- Moving a pole toward the unit circle makes the filter more resonant (higher Q)

**Audio code connection:** `a1`, `a2` coefficients in a biquad difference equation come directly from the pole positions after bilinear transform from s-plane to z-plane.

---

## Biquad / IIR Filters

**Intuition:** Each output sample depends on past output samples as well as past inputs — the feedback is what gives IIR filters steep rolloffs at low cost.

**Key formula (Direct Form I):**
```
y[n] = b0*x[n] + b1*x[n-1] + b2*x[n-2]
             - a1*y[n-1] - a2*y[n-2]
```

**Properties / gotchas:**
- `b0,b1,b2` are feedforward (zeros); `a1,a2` are feedback (poles)
- Cookbook coefficients (RBJ, Zolzer): pre-calculated from `fc`, `Q`, `gain`
- Q > 0.707 → overshoot; Q approaching infinity → oscillation at fc
- `a1^2 - 4*a2 > 0` → real poles (overdamped); `< 0` → complex conjugate poles (ringing)
- Transposed Direct Form II is commonly used for floating-point biquads because it has compact state and good roundoff behaviour; cascaded SOS sections are safer than one high-order direct form
- Cascaded biquads (SOS) avoid coefficient sensitivity of high-order single-stage designs
- At very low `fc/sr` ratios, bilinear transform warps frequency — pre-warp `fc` before design

**Audio code connection:** `juce::dsp::IIR::Coefficients`, `juce::dsp::StateVariableTPTFilter`, or hand-rolled arrays `b[3]`, `a[3]` with a two-sample state buffer per channel.

---

## FIR Filters

**Intuition:** Output is a finite weighted sum of current and past inputs only — no feedback, so the filter is always stable.

**Key formula:**
```
y[n] = sum_{k=0}^{M-1} h[k] * x[n-k]
```

**Properties / gotchas:**
- Always stable (no poles, only zeros)
- Linear phase: symmetric coefficients `h[k] = h[M-1-k]` → constant group delay = `(M-1)/2` samples
- Length M sets transition bandwidth: narrower transition requires more taps (~O(1/BW))
- Design methods: windowed-sinc (simple), Parks-McClellan/equiripple (optimal stopband)
- Computational cost = O(M) per sample — expensive for long filters; use FFT convolution for M > ~64
- Fractional delay FIR: sinc interpolation for pitch shifting / resampling

**Audio code connection:** Oversampling anti-alias filters, linear-phase EQ matching, and crossover filters are typically FIR. `juce::dsp::FIR::Filter` wraps the tap-loop.

---

## Windowing

**Intuition:** Abruptly truncating a signal before FFT creates artificial high-frequency edges — a window function tapers the signal smoothly to zero at both ends, reducing that spectral splatter.

**Key formula:**
```
X_windowed[k] = DFT{ x[n] * w[n] }
```
Hann window: `w[n] = 0.5 * (1 - cos(2*pi*n / (N-1)))`

**Properties / gotchas:**
- Rectangular window = no windowing = worst spectral leakage, sharpest main lobe
- Hann: good general purpose; -31 dB first sidelobe
- Blackman-Harris: very low sidelobes (-92 dB), wider main lobe — use for high-dynamic-range analysis
- Flat-top: accurate amplitude, wide lobe — use for level measurement
- Window broadens the main lobe by ~2x (Hann) relative to rectangular; you lose some frequency resolution
- For overlap-add STFT synthesis, use a window/hop pair that satisfies constant-overlap-add; Hann works for common 50% or 75% overlap cases when paired consistently

**Audio code connection:** Multiply the input buffer by `window[n]` before calling FFT; in STFT vocoders and spectral processors this is the first operation in each analysis frame.

---

## Sample Rate / Nyquist

**Intuition:** You need at least two samples per cycle to represent a frequency — so the highest frequency a digital system can hold is exactly half the sample rate.

**Key formula:**
```
f_nyquist = sampleRate / 2
f_alias   = |f_signal - round(f_signal / sampleRate) * sampleRate|
```

**Properties / gotchas:**
- Signals above Nyquist fold back (alias) into the audible band — they cannot be removed after sampling
- Anti-alias (reconstruction) filter must cut everything above Nyquist before ADC and after DAC
- Oversampling: run DSP at 2x/4x/8x sr, apply nonlinearity, then downsample — aliases land above new Nyquist
- sinc interpolation is the ideal reconstruction filter; practical implementations use windowed-sinc FIR
- A 44.1 kHz system reproduces up to 22.05 kHz; 48 kHz up to 24 kHz

**Audio code connection:** Waveshaper / saturation / oscillator code should check if harmonics will exceed `sampleRate/2`; distortion plugins almost always need oversampling.

---

## Decibels

**Intuition:** Decibels compress the enormous range of audible amplitudes (1:1,000,000) into a manageable linear scale where +6 dB ≈ double the amplitude.

**Key formulas:**
```
dBFS (amplitude) = 20 * log10(|amplitude|)    // 0 dBFS = full scale
dBFS (power)     = 10 * log10(power)
amplitude        = 10^(dB / 20)
power            = 10^(dB / 10)
```

**Properties / gotchas:**
- 0 dB amplitude = 1.0 full scale; -6 dB ≈ half amplitude; -20 dB = 1/10 amplitude
- Power vs amplitude: use factor 20 for voltages/samples, 10 for power/energy
- `log10(0)` = -inf — always clamp or add a small epsilon before converting: `max(|x|, 1e-9)`
- dBu / dBV / dBSPL are absolute references; dBFS is relative to full scale
- Gain staging: sum of gains in dB = multiplication of linear gains

**Audio code connection:** Fader values, metering, threshold/ratio in compressors, and all UI gain controls live in dB; convert to linear before multiplying audio buffers (`gain_linear = pow(10.0f, gain_db / 20.0f)`).
