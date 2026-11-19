# 2 — Toolbox building blocks (1 h 15)

**Goal:** try each toolbox object on its own in a scratch script (`scratch/explore.m`), then combine them into
a first modulator **without classes**. You end with a correct USB/LSB spectrum on screen (milestone **M1**).

**Course link:** Lecture 5: `stream_function.m` (look at the `'SystemObject'` case: reader → filter → writer in a loop)
and `realtime_audio.m` (`dsp.SpectrumAnalyzer`).

---

## 2.1 Frames and state: why toolbox objects

A stream arrives in **frames** (e.g. 1024 samples). Filters, the Hilbert transform and the oscillator all "remember the past".
If each frame is processed as if it were a separate signal, every frame boundary has a small error, and at 47 frames/s you hear it as a buzz.

Toolbox System objects (`dsp.LowpassFilter`, `dsp.AnalyticSignal`, …) **keep their state between calls**, so frame
boundaries are invisible. That's why we use them instead of `filter(b,1,x)` or `hilbert(x)` on each frame.
Their life cycle: create → call it like a function on each frame (`y = obj(x)`) → `reset(obj)` / `release(obj)`.

🧠 **Q1.** MATLAB's `hilbert(x)` uses the FFT of `x`. Why does that go wrong if you call it on each 1024-sample frame separately?

<details>
<summary><b>✅ Answer</b></summary>

The FFT treats each frame as if it repeated periodically, so the edges of every frame are wrong. That causes clicks at every
frame boundary and an imperfectly suppressed sideband. `dsp.AnalyticSignal` uses an FIR filter with memory instead, so it's correct across boundaries.
</details>

## 2.2 Reading a file in frames

```matlab
reader = dsp.AudioFileReader('Filename', 'yourfile.wav', 'SamplesPerFrame', 1024);
fs = reader.SampleRate;
while ~isDone(reader)
    x = reader();
    x = mean(x, 2);          % stereo → mono
    % … process …
end
release(reader);
```

🛠 **Task 2.1** Read your test file this way and show it in a `spectrumAnalyzer` (`scope = spectrumAnalyzer('SampleRate', fs); scope(x)` in the loop).
In older releases the object is called `dsp.SpectrumAnalyzer`, as in the lecture.

## 2.3 The message low-pass filter

```matlab
lpf = dsp.LowpassFilter('SampleRate', fs, 'PassbandFrequency', 3400, 'StopbandFrequency', 4000);
fvtool(lpf)                  % look at its frequency response
```

🛠 **Task 2.2** Filter your file with it and compare the spectrum before and after (two channels in one scope: `scope([x, xf])`).

## 2.4 `dsp.AnalyticSignal`

```matlab
hilb = dsp.AnalyticSignal('FilterOrder', 500);
a = hilb(x);                 % complex frame: approximately m + j·m̂
```

The Hilbert part is delayed by `FilterOrder/2` = 250 samples (it's an FIR filter). For the formula $m\cos - \hat m\sin$ to work,
**the real part must be delayed by the same amount.** The documentation doesn't say clearly whether it is, so test it yourself.

🧠 **Q2.** Feed a pure tone $\cos(\omega n)$ into it. If real and imaginary parts are aligned, $a = e^{j\omega(n-D)}$. What is `abs(a)` then?
What would `abs(a)` look like if they were **not** aligned?

<details>
<summary><b>✅ Answer</b></summary>

Aligned: `abs(a)` = 1, constant (after the start-up transient). Not aligned: the real part is $\cos(\omega n)$ and the imaginary part $\sin(\omega(n-D))$,
so $|a|^2 = 1 - \sin\theta\,\sin(2\omega n - \theta)$ with $\theta = \omega D$. It **oscillates** at $2\omega$ between $1-|\sin\theta|$ and $1+|\sin\theta|$.
For 1 kHz at 48 kHz and D = 250, $|\sin\theta| \approx 0.97$, so you can't miss it. A constant magnitude is a one-line test.
</details>

🛠 **Task 2.3** Generate 0.5 s of a 1 kHz cosine at 48 kHz, push it through `hilb` in frames of 1024, and plot `abs(a)` for the second half.
If it's flat at 1: aligned, nothing to do. If it oscillates: put a `dsp.Delay(250)` on the real part yourself.

🧠 **Q3.** Why `FilterOrder` 500 and not the default 100? (Hint: what does a short FIR Hilbert filter do at low frequencies?)

<details>
<summary><b>✅ Answer</b></summary>

An FIR Hilbert transformer can't have gain 1 near 0 Hz. The shorter it is, the higher up its gain starts to drop. With order 100 at 48 kHz,
the low part of a voice (100–500 Hz) is transformed badly, so that part leaks into the wrong sideband. A longer filter fixes it, at the cost of
more delay (250 samples = 5 ms, which nobody hears). You can try both in Task 2.5 and compare the unwanted sideband.
</details>

## 2.5 The carrier: phase accumulator

```matlab
phase = 0;                                  % before the loop
dPhi  = 2*pi*fc/fs;
% inside the loop, for a frame of n samples:
phi   = phase + dPhi*(0:n-1)';
phase = mod(phase + dPhi*n, 2*pi);
```

🧠 **Q4.** Why not simply `t = (0:n-1)'/fs; cos(2*pi*fc*t)` in each frame? And why keep a phase instead of a global time `t`
(think of the carrier slider in the GUI)?

<details>
<summary><b>✅ Answer</b></summary>

`t` restarting at 0 restarts the carrier every frame: a phase jump at every boundary, heard as a buzz. With a global time `t`,
changing `fc` while running makes the phase $2\pi f_c t$ jump by a huge amount at once. With the accumulator, only the step size changes, so the
frequency changes smoothly. `mod 2π` keeps the number small, so `cos` stays accurate.
</details>

## 2.6 Put it together: first modulator, no classes

🛠 **Task 2.4** In the loop: read → mono → low-pass → `hilb` → carrier phase → `s = real(a).*cos(phi) - sb*imag(a).*sin(phi)`.
Show `[x, s]` in the spectrum analyzer. Set the scope's frequency span to 0…24 kHz.

🛠 **Task 2.5** Use a 1 kHz test tone instead of the file (make it frame by frame with its own phase accumulator).
Check: USB gives one line at 11 kHz and LSB at 9 kHz, with nothing at 10 kHz.

<details>
<summary><b>💡 Hint — compact form with complex numbers</b></summary>

`s = real(a .* exp(1j*phi))` is the USB, and `s = real(conj(a) .* exp(1j*phi))` is the LSB. Expand them to see they match the boxed formula.
</details>

⭐ **Optional (10 min):** add the demodulator here too (multiply by `cos(phi)`, a second `dsp.LowpassFilter`, ×2) and listen with
`audioDeviceWriter`. Chapter 4 does it properly.

---

🎯 **Checkpoint (M1):** the spectrum analyzer shows the correct USB and LSB for the test tone and your file. You know whether
`dsp.AnalyticSignal` needs an extra delay. ➡️ [Chapter 3](03-system-objects.md)
