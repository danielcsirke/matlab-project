# 3 — Offline prototype (whole signal at once)

**Goal:** see SSB working before worrying about frames. Process a whole signal in one go,
plot the spectra, and listen to the result. This is milestone **M1**. You'll throw this code away
later; it's for building intuition.

**MATLAB you'll use:** `audioread`, `sound`/`audioplayer` (Lecture 4–5), `fft`, `plot`, `subplot`/`tiledlayout`,
and optionally `hilbert` (Signal Processing Toolbox).

---

## 3.1 A spectrum helper

You'll need spectrum plots over and over, so write one helper first.

🛠 **Task 3.1** Write `plotSpectrum(ax, x, fs, label)`. It plots the magnitude spectrum in dB over 0…fs/2 (kHz) into axes `ax`.
Use a window to reduce leakage. A Hann window in base MATLAB is `w = 0.5 - 0.5*cos(2*pi*(0:N-1)'/(N-1))`.

<details>
<summary><b>💡 Hint</b></summary>

`X = fft(x.*w); P = abs(X(1:floor(N/2)+1)); f = (0:floor(N/2))'*fs/N;` then `plot(ax, f/1e3, 20*log10(P/max(P) + eps))`.
Use `hold(ax,'on')` so you can overlay several signals.
</details>

## 3.2 The analytic signal via FFT

For the whole-signal prototype you can get $\hat m$ from the FFT, because you know the whole signal.
This is exactly what MATLAB's `hilbert()` does internally. Note that `hilbert()` returns the
**analytic signal** $m + j\hat m$, not $\hat m$.

🧠 **Q1.** From chapter 1 Q3: $M_a(f) = 2M(f)$ for $f > 0$ and $0$ for $f<0$. In the FFT bins of an
even-length signal, which bins get factor 2, which get 1, and which get 0?

<details>
<summary><b>✅ Answer</b></summary>

Bin 1 (DC) gets ×1. Bins 2 … N/2 (positive frequencies) get ×2. Bin N/2+1 (Nyquist) gets ×1.
The rest (negative frequencies) get ×0. Then `ma = ifft(X .* g)` and `mHat = imag(ma)`.
</details>

🛠 **Task 3.2** Write `analyticSignal(x)` this way. If you have the SPT, compare with `hilbert(x)`; the difference should be around 1e-15.

## 3.3 Modulate, look, listen

🛠 **Task 3.3** In `scratch/prototype_offline.m`:

1. Make a test message: a sum of 3–4 tones between 500 and 3000 Hz, 2 s long, fs = 48 kHz. (Later, use speech: `audioread`, convert to mono with `mean(x,2)`, and `resample` if needed.)
2. Build the USB and LSB signals with the boxed formula from chapter 1.
3. Plot message, USB and LSB spectra in one figure (3 tiles).
4. Listen: `sound(0.5*sUsb/max(abs(sUsb)), fs)`. What does SSB sound like?

🧠 **Q2.** Before you run it: at what frequencies should the four peaks of a 500/1000/2000/3000 Hz message appear for USB with $f_c$ = 10 kHz? And for LSB?

<details>
<summary><b>✅ Answer</b></summary>

USB: 10.5, 11, 12, 13 kHz. LSB: 9.5, 9, 8, 7 kHz, so the spectrum is **mirrored**.
That mirroring is why LSB speech sounds "upside down" if you demodulate it as USB.
</details>

## 3.4 Demodulate

🛠 **Task 3.4** Implement the **simple product detector** (chapter 1 §1.4) on the whole signal: multiply by
$\cos(\omega_c t)$, then low-pass. For now use the LPF from chapter 2 with `filter`,
or a quick FFT mask. Remember the factor 2 (Q8 gave $m/2$).

🛠 **Task 3.5 — experiments** (write down what you hear and see; these make good slides for the presentation):

| Experiment | Expected |
|---|---|
| LO phase offset θ = 90° | Waveform looks different, sounds the same (Q9) |
| LO offset Δf = +50 Hz on USB, then on LSB | pitch moves down / up (Q10) |
| Demodulate LSB as if USB with the product detector | still works! (Q8), so why do we need the phasing demodulator? |
| Add both a USB and an LSB signal carrying **different** messages, product-detect | you hear both mixed: the product detector can't separate them |

🛠 **Task 3.6** Now implement the **phasing demodulator** from chapter 1 §1.5, still on the whole signal (use your `analyticSignal` to get $\mathcal H\{Q\}$).
Repeat the last experiment: you should now be able to pick either message.

<details>
<summary><b>✅ Solution sketch (whole-signal, not the final code)</b></summary>

```matlab
fs = 48e3; fc = 10e3; t = (0:2*fs-1)'/fs;
m  = 0.2*(cos(2*pi*500*t) + cos(2*pi*1000*t) + cos(2*pi*2000*t) + cos(2*pi*3000*t));
mHat = imag(analyticSignal(m));
sb = +1;                                            % USB
s  = m.*cos(2*pi*fc*t) - sb*mHat.*sin(2*pi*fc*t);

hLp = designLowpassFir(255, 4e3, fs, 8);
I = filter(hLp, 1,  s.*cos(2*pi*fc*t));
Q = filter(hLp, 1, -s.*sin(2*pi*fc*t));
y = I - sb*imag(analyticSignal(Q));                 % ≈ m delayed by 127 samples
```
</details>

---

🎯 **Checkpoint (M1):** one figure that clearly shows message, USB and LSB spectra, plus working product and
phasing demodulators on the whole signal. **Commit it.**

➡️ Next: why the real program processes even files in frames: [chapter 4](04-frames-and-state.md).
