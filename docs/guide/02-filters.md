# 2 — Designing the filters (base MATLAB only)

**Goal:** write four small design functions and check them with plots:

| Function | Used for |
|---|---|
| `w = designKaiserWindow(L, beta)` | all the others |
| `h = designHilbertFir(L, beta)` | Hilbert branch in the modulator and demodulator |
| `h = designLowpassFir(L, fCut, fs, beta)` | demodulator I/Q low-pass |
| `h = designBandpassFir(L, fLow, fHigh, fs, beta)` | message band-limiting in the modulator |

**Why by hand?** The Signal Processing Toolbox has `firpm`, `kaiser` and `designfilt`, but
(1) writing them yourself shows you understand the math, (2) the core then needs only base
MATLAB (NFR-4), and (3) deploying it is simpler. If you have the toolbox, use it to **check** your results.

**MATLAB you'll use:** `besseli`, `fft`, element-wise operators, logical indexing, `plot`/`semilogx`,
function files with help comments (Lecture 3), input validation (Lecture 7).

---

## 2.1 The ideal discrete Hilbert transformer

For a discrete-time system, $H(e^{j\omega}) = -j\operatorname{sgn}(\omega)$ for $\omega \in (-\pi,\pi)$, and the impulse
response is the inverse DTFT:

$$h[n] = \frac{1}{2\pi}\int_{-\pi}^{\pi} H(e^{j\omega})\,e^{j\omega n}\,d\omega$$

🧠 **Q1.** Compute $h[n]$. What is it for even $n$, and what is it for odd $n$?

<details>
<summary><b>💡 Hint</b></summary>

Split the integral into $(-\pi,0)$ and $(0,\pi)$. You'll get $\dfrac{1-\cos(\pi n)}{\pi n}$, and $h[0]$ needs separate handling.
</details>

<details>
<summary><b>✅ Answer</b></summary>

$$h[n] = \begin{cases}\dfrac{2}{\pi n} & n \text{ odd}\\[4pt] 0 & n \text{ even (including } 0)\end{cases}$$
It's infinitely long, antisymmetric, and non-causal.
</details>

## 2.2 From ideal to practical: truncate, window, delay

1. **Truncate** to $n = -D \dots D$, which gives $L = 2D+1$ taps. Use an **odd** $L$ so the delay $D$ is a whole number of samples.
2. **Window** it. Simply cutting the response off causes ripple (Gibbs phenomenon). Multiplying by a smooth window trades ripple for a wider transition region.
3. **Delay** it. Index $n=-D$ becomes tap 1, which makes the filter causal. It now delays everything by $D$ samples, so **the other branch ($m$ itself) must be delayed by $D$ too**, or the two branches won't line up and the sidebands won't cancel.

🧠 **Q2.** Why does an odd-length, antisymmetric FIR always have gain 0 at DC and at $f_s/2$?
What does that mean for the lowest message frequency you can use?

<details>
<summary><b>✅ Answer</b></summary>

An antisymmetric odd-length filter (type III) has zeros at $\omega = 0$ and $\omega = \pi$ by construction.
Near DC the gain therefore rolls off, so the 300 Hz lower band edge is where the Hilbert filter is
least accurate, and it decides how long the filter must be. That's also why the message is
band-pass filtered to 300–3400 Hz first.
</details>

### The Kaiser window

$$w[k] = \frac{I_0\!\left(\beta\sqrt{1-\left(\frac{2k}{L-1}-1\right)^2}\right)}{I_0(\beta)},\quad k=0\dots L-1$$

$I_0$ is the modified Bessel function of the first kind, order 0: `besseli(0, x)` in base MATLAB.
A larger $\beta$ gives lower ripple but a wider transition.

🛠 **Task 2.1** Write `designKaiserWindow(L, beta)`. It returns a **column** vector. Check it against `kaiser(L, beta)` if you have the SPT.

🛠 **Task 2.2** Write `designHilbertFir(L, beta)`. Throw an error with ID `SSB:invalidLength` if `L` is even.

<details>
<summary><b>💡 Hint</b></summary>

`n = (-(L-1)/2 : (L-1)/2)'`. Make `h = zeros(L,1)`, set the odd entries using logical indexing
(`mod(n,2) ~= 0` works for negative `n` too in MATLAB), then multiply element-wise by the window.
</details>

<details>
<summary><b>✅ Solution</b></summary>

```matlab
function w = designKaiserWindow(L, beta)
%DESIGNKAISERWINDOW Kaiser window as a column vector.
%   w = designKaiserWindow(L, beta) returns L samples of a Kaiser window
%   with shape parameter beta. Base MATLAB only (uses besseli).
k = (0:L-1)';
r = 2*k/(L-1) - 1;
w = besseli(0, beta*sqrt(1 - r.^2)) / besseli(0, beta);
end
```

```matlab
function h = designHilbertFir(L, beta)
%DESIGNHILBERTFIR Windowed type-III FIR Hilbert transformer.
%   h = designHilbertFir(L, beta) returns an odd-length L column vector.
%   Group delay is (L-1)/2 samples.
if mod(L, 2) ~= 1
    error('SSB:invalidLength', 'Hilbert filter length must be odd, got %d.', L);
end
n = (-(L-1)/2 : (L-1)/2)';
h = zeros(L, 1);
isOdd = mod(n, 2) ~= 0;
h(isOdd) = 2 ./ (pi*n(isOdd));
h = h .* designKaiserWindow(L, beta);
end
```
</details>

## 2.3 Checking a filter: frequency response without toolboxes

The DTFT of an FIR, sampled on a grid, is just a zero-padded FFT:

```matlab
nfft = 2^16;
Hf = fft(h, nfft);
f  = (0:nfft/2)' * fs/nfft;          % 0 … fs/2
Hf = Hf(1:nfft/2+1);
```

🛠 **Task 2.3** Plot `abs(Hf)` of your Hilbert filter (L = 511, β = 8, fs = 48 kHz) with a log frequency axis.
Then plot the **magnitude error** `abs(abs(Hf) - 1)` in dB over 300–3400 Hz.

🧠 **Q3.** How would you check the phase? A delayed Hilbert filter should have phase $-\pi/2 - \omega D$.

<details>
<summary><b>💡 Hint</b></summary>

Remove the delay: `Hf .* exp(1j*2*pi*f/fs*D)` should be close to `-1j` in the passband. Look at `angle(...)` there.
</details>

🛠 **Task 2.4 (experiment, important):** fill in this table. Use Q7 from chapter 1 to turn the worst
in-band magnitude error ε into a sideband suppression $20\log_{10}(\varepsilon/2)$.

| L | window | worst ε in 300–3400 Hz | suppression (dB) |
|---|---|---|---|
| 255 | Kaiser β=8 | | |
| 383 | Kaiser β=8 | | |
| 511 | Kaiser β=8 | | |
| 511 | Hamming | | |

<details>
<summary><b>✅ Reference values</b> (fs = 48 kHz)</summary>

| L | window | suppression at the worst point |
|---|---|---|
| 255 | Kaiser β=8 | ≈ −30 dB (fails at 300 Hz) |
| 383 | Kaiser β=8 | ≈ −62 dB |
| 511 | Kaiser β=8 | ≈ −83 dB |
| 511 | Hamming | ≈ −61 dB |

**Lesson:** the low band edge (300 Hz) decides the length. 511 taps at 48 kHz is a delay of
255 samples (5.3 ms), which is fine for real-time use.
</details>

## 2.4 Low-pass and band-pass by windowed sinc

The ideal low-pass with cutoff $f_\text{cut}$ has

$$h[n] = \frac{2f_\text{cut}}{f_s}\,\operatorname{sinc}\!\left(\frac{2f_\text{cut}}{f_s}n\right),\qquad \operatorname{sinc}(x)=\frac{\sin \pi x}{\pi x}$$

⚠️ The `sinc` function itself is in the Signal Processing Toolbox. In base MATLAB, write
$\sin(\pi x)/(\pi x)$ yourself and handle $x = 0$ (value 1) separately.

🧠 **Q4.** How do you get a band-pass filter from two low-pass filters? Why must both have the **same length**?

<details>
<summary><b>✅ Answer</b></summary>

$h_\text{BP} = h_\text{LP}(f_\text{high}) - h_\text{LP}(f_\text{low})$. The responses subtract: about 1 − 1 = 0 below
$f_\text{low}$, 1 − 0 = 1 between the edges, and 0 − 0 = 0 above. This only works if both filters have the same
delay, which means the same length.
</details>

🛠 **Task 2.5** Write `designLowpassFir` (normalize so `sum(h) == 1`, i.e. unity gain at DC) and `designBandpassFir`.
Plot both responses in dB. Check: the demod LPF (255 taps, 4 kHz cutoff) should be below −60 dB at 16 kHz,
where the $2f_c$ products start for $f_c = 10$ kHz.

<details>
<summary><b>✅ Solution</b></summary>

```matlab
function h = designLowpassFir(L, fCut, fs, beta)
%DESIGNLOWPASSFIR Kaiser-windowed sinc low-pass FIR, unity DC gain.
n = (-(L-1)/2 : (L-1)/2)';
x = 2*fCut/fs * n;
h = ones(L, 1);
isNonzero = x ~= 0;
h(isNonzero) = sin(pi*x(isNonzero)) ./ (pi*x(isNonzero));
h = h .* designKaiserWindow(L, beta);
h = h / sum(h);
end
```

```matlab
function h = designBandpassFir(L, fLow, fHigh, fs, beta)
%DESIGNBANDPASSFIR Band-pass FIR as the difference of two low-pass filters.
h = designLowpassFir(L, fHigh, fs, beta) - designLowpassFir(L, fLow, fs, beta);
end
```
</details>

## 2.5 Style points (graders notice these)

- Every function starts with an H1 help line (`%NAME One-line description`) and a usage line. Try `help designHilbertFir`.
- Validate inputs. Use an `arguments` block or `validateattributes` (Lecture 7). Example: `L` must be a positive odd integer and `fCut < fs/2`.
- Prefer column vectors throughout. Frames are columns too.

---

🎯 **Checkpoint (M2):** four functions, one script `scratch/plotFilters.m` that plots all responses, and your filled-in table from Task 2.4.
