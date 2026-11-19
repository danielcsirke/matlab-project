# 4 — Frames and state: the core of streaming

**Goal:** understand why "process the whole signal" code breaks when the signal arrives in small
pieces, and learn the three kinds of **state** that fix it. This is the hardest *idea* in the project.
Once it makes sense, the System objects in chapter 5 are mostly packaging.

**Course link:** Lecture 5, `stream_function.m` (the "Function" vs "SystemObject" comparison). Look at how
the function version passes `z` into and out of `filter`. Lecture 3, `myf_persistent.m`, shows another way
to keep values between calls.

---

## 4.1 The streaming model, and why we use it for files too

A live stream arrives as **frames** of N samples (e.g. N = 1024, which is 21.3 ms at 48 kHz). The code is called once per
frame and must return N output samples **before the next frame arrives**.

In Phase 1 the input is a file, so you *could* process it in one piece. The spec still requires frames
(SPEC §1, rule R-1). If the DSP already works frame by frame, live streaming later (Phase 2) only needs a different
source and sink, and the DSP stays as it is. Code that processes the whole signal at once would have to be
rewritten for streaming.

The rule that drives everything else:

> **Processing a signal in frames must give exactly the same output as processing it in one piece.**

That's SPEC NFR-3, and it's also your most powerful test (chapter 6).

Anything that "remembers the past" breaks this rule unless you **carry its memory from one frame to the next**.
The SSB chain has three such things:

| Kind of state | Where | What to remember |
|---|---|---|
| FIR filter memory | band-pass, Hilbert, low-pass | the last L−1 input samples |
| Delay line | the "plain" branch, delayed by D to match the Hilbert filter | the last D samples |
| Oscillator phase | carrier (modulator) and LO (demodulator) | the phase where the last frame ended |

## 4.2 FIR filter state

🛠 **Task 4.1 — see the bug.** Take a 1 kHz tone and filter it with your low-pass filter in frames of 512,
calling `y = filter(h, 1, frame)` **without** state. Plot the result around a frame boundary.

🧠 **Q1.** What do you see at each frame boundary, and why? What will it sound like?

<details>
<summary><b>✅ Answer</b></summary>

Every frame starts as if the past were zero, so there's a start-up transient at every boundary.
At 48 kHz / 512 that happens 94 times per second, which you hear as a buzz or clicks.
</details>

`filter` supports state directly:

```matlab
[y, zf] = filter(b, a, x, zi)   % zi: initial state, zf: final state
```

For an FIR (`a = 1`) the state is a column of `numel(b)-1` values. Start with zeros and feed each frame's `zf`
in as the next frame's `zi`.

🛠 **Task 4.2** Fix Task 4.1 with state. Then **compare against the whole-signal result**:
`max(abs(yFrames - yWhole))` should be about 1e-15.

## 4.3 The delay line

The plain branch must be delayed by D = (L−1)/2 samples. Within a whole signal that's just
`[zeros(D,1); m(1:end-D)]`. Across frames, the last D samples of each frame must come out at the start of the next one.

🛠 **Task 4.3** Implement the delay with a buffer variable `buf` (length D, initially zeros). It must also work when
**N < D** (e.g. N = 128, D = 255).

<details>
<summary><b>💡 Hint</b></summary>

Stick the new frame onto the end of the buffer. The output is the first N samples of the combined vector,
and the new buffer is what's left. Two lines of indexing.
</details>

<details>
<summary><b>✅ Solution</b></summary>

```matlab
combined = [buf; x];
y   = combined(1:n);
buf = combined(n+1:end);
```
This works for any N, larger or smaller than D.
(Alternatively, a delay is an FIR with taps `[zeros(D,1); 1]`, so `filter` with state works too, but it costs D multiplications per sample for nothing.)
</details>

## 4.4 The oscillator: phase accumulator

The tempting code is `cos(2*pi*fc*t)` with `t = (0:N-1)'/fs`, but that restarts the carrier in every frame.

🧠 **Q2.** Two fixes: (a) keep a global sample counter $n_0$ and use $t = (n_0 + (0:N-1))/f_s$; (b) keep a **phase** $\varphi_0$ and use
$\varphi = \varphi_0 + \Delta\varphi\cdot(0{:}N{-}1)$ with $\Delta\varphi = 2\pi f_c/f_s$, then update $\varphi_0 \leftarrow (\varphi_0 + N\Delta\varphi) \bmod 2\pi$.
Both are continuous for a fixed $f_c$. Which one stays continuous when the user **moves the carrier slider** while it's running (FR-1.4)? Why the `mod 2π`?

<details>
<summary><b>✅ Answer</b></summary>

(b). With (a), changing $f_c$ changes $2\pi f_c t$ for *all* $t$ at once, so the phase jumps by
$2\pi\Delta f_c\, n_0/f_s$, which can be huge after a minute of streaming. With (b) only the *step size* changes, so
the phase stays continuous (frequency changes, phase doesn't jump).
`mod 2π` keeps $\varphi_0$ small. After hours the plain sum would grow to around $10^{9}$ and `double`
would lose precision in `cos`.
</details>

🛠 **Task 4.4** Write the phase-accumulator version. Check: generate 1 s of carrier in frames of 1000 and in one piece and compare (difference ~1e-12, not 0. Why?).

## 4.5 Why not just call `hilbert()` on each frame?

(This is why SPEC rule R-1 forbids whole-signal operations in the chain.)

`hilbert()` uses the FFT of the frame, which assumes the frame repeats periodically. The edges are wrong in every frame.

🛠 **Task 4.5 (experiment):** build a frame-wise USB modulator that calls `hilbert()` per frame (with a correct, continuous carrier),
and one with your stateful FIR Hilbert filter. Measure the unwanted sideband for a 1 kHz tone, and listen to both.

<details>
<summary><b>✅ What you should find</b></summary>

With a reference model at N = 1024, the per-frame `hilbert()` left the unwanted sideband at about **−50 dB**
and adds broadband clicks at the frame rate. The stateful FIR version reached about **−100 dB**.
The FFT method is fine for files, but wrong for streams.
</details>

## 4.6 Latency

Every FIR adds (L−1)/2 samples of delay, plus the audio device buffers. For this design:

🧠 **Q3.** Add up the latency from input to output in samples and ms (band-pass 511, Hilbert 511 in the modulator; LPF 255, Hilbert 511 in the demodulator; fs = 48 kHz). Ignore the audio buffers.

<details>
<summary><b>✅ Answer</b></summary>

255 + 255 + 127 + 255 = **892 samples = 18.6 ms**. You'll need this number in the loopback test to line up input and output.
</details>

## 4.7 Putting it together: one frame of the modulator

🛠 **Task 4.6** Write a **plain function** (no classes yet)

```matlab
[y, state] = modulateFrame(x, state, params)
```

where `state` is a struct with fields for the band-pass memory, Hilbert memory, delay buffer and phase, and
`params` holds `fc`, `fs`, `sb` and the filter coefficients. Also write a matching `state = initModulatorState(params)`.

Then run a 2 s multitone through it in frames of 1024, and again in frames of irregular sizes (e.g. 100, 333, 1024, 7, …),
and compare the outputs. **They must match to ~1e-12.** If not, one of the three kinds of state is wrong.

<details>
<summary><b>💡 Hint — order of operations inside one frame</b></summary>

1. band-pass the input (with state) → `m`
2. Hilbert-filter `m` (with state) → `mHat`
3. delay `m` by D (with buffer) → `mDelayed`
4. phase vector from accumulator; update accumulator
5. `y = mDelayed.*cos(phi) - sb*mHat.*sin(phi)`
</details>

This function is the core of `SSBModulator.stepImpl` in the next chapter. It's ok if
`modulateFrame` stays in `scratch/` afterwards.

---

🎯 **Checkpoint:** frame-wise modulator matches itself for any frame split; you can explain each of the three kinds of state.
➡️ [Chapter 5](05-system-objects.md)
