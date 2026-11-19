# 6 — Testing

**Goal:** an automated test suite for SPEC §9 (T1–T11). Run it after every change with one command:

```matlab
results = runtests('tests');
table(results)
```

**Course link:** Lecture 7 (`mySqrt_testfunctions.m`, `testquadrangle*.m`), Lecture 8 (`timeit`, `exercise5TestingFramework*.m`).

DSP code is a great fit for automated tests: you *know* the right answer from the math (chapter 1).
Many people only "look at the plot and listen". Measurable criteria are better engineering and better for your grade.

---

## 6.1 Function-based test file

```matlab
function tests = ssbTests
%SSBTESTS Unit tests for the SSB modulator and demodulator (SPEC section 9).
tests = functiontests(localfunctions);
end

function setupOnce(testCase)
    addpath(genpath(fullfile(fileparts(mfilename('fullpath')), '..', 'src')));
    testCase.TestData.fs = 48e3;
    testCase.TestData.fc = 10e3;
end

function testSomething(testCase)
    verifyEqual(testCase, 1 + 1, 2);
end
```

Every local function whose name starts with `test` is a test. Use the `verify*` methods
(`verifyGreaterThan`, `verifyLessThan`, `verifyEqual(…, 'AbsTol', …)`, `verifyError`) so a failure doesn't stop the remaining tests.

## 6.2 Two helpers you'll use everywhere

🛠 **Task 6.1** Write these as local functions at the bottom of the test file:

1. `y = runInFrames(sysObj, x, frameSizes)` runs a System object over `x` frame by frame. `frameSizes` can be a scalar or a vector that's cycled through (for irregular splits).
2. `p = powerAt(x, fs, f)` gives the power at frequency `f`. Use a Hann window and an FFT.
   **Tip:** if `numel(x) == fs` (1 s), the FFT bins are exactly 1 Hz apart, so bin `f+1` is frequency `f`.

<details>
<summary><b>✅ Solution</b></summary>

```matlab
function y = runInFrames(sysObj, x, frameSizes)
y = zeros(size(x));
first = 1;
k = 1;
while first <= numel(x)
    n = frameSizes(mod(k-1, numel(frameSizes)) + 1);
    idx = first:min(first + n - 1, numel(x));
    y(idx) = sysObj(x(idx));
    first = idx(end) + 1;
    k = k + 1;
end
end

function p = powerAt(x, fs, f)
n = numel(x);
w = 0.5 - 0.5*cos(2*pi*(0:n-1)'/(n-1));
X = fft(x .* w);
p = abs(X(round(f/fs*n) + 1))^2;
end
```
</details>

## 6.3 One complete example test (T1)

This one is given in full so you can see the pattern. Use it as a template for the others.

```matlab
function testUsbToneLandsAboveCarrier(testCase)
    fs = testCase.TestData.fs;  fc = testCase.TestData.fc;  f0 = 1e3;
    x = 0.5*cos(2*pi*f0*(0:1.5*fs-1)'/fs);
    modulator = SSBModulator('SampleRate', fs, 'CarrierFrequency', fc, 'Sideband', 'USB');

    y = runInFrames(modulator, x, 1024);
    y = y(end-fs+1:end);                    % last 1 s: start-up transient is gone

    ratioDb = 10*log10(powerAt(y, fs, fc+f0) / powerAt(y, fs, fc-f0));
    verifyGreaterThan(testCase, ratioDb, 60);
end
```

🧠 **Q1.** Why drop the first 0.5 s?

<details>
<summary><b>✅ Answer</b></summary>

The filters start with zero state, so the first ~latency samples are a transient (a few ms).
Dropping 0.5 s is generous, and it leaves exactly 1 s, which makes the FFT bins land on whole Hz.
</details>

## 6.4 Your tests

🛠 **Task 6.2** Write T2–T10 from SPEC §9. Hints for the trickier ones:

<details>
<summary><b>💡 T4 — loopback accuracy</b></summary>

Use a multitone **inside** 500–3000 Hz (the band-pass filter changes anything near 300/3400 Hz). Compare
`y(k)` with `x(k - D)` where `D = modulator.latency() + demodulator.latency()`. Skip the first 0.5 s.
Relative RMS error: `sqrt(mean(e.^2)/mean(ref.^2))`. Expect around 0.2%; the spec limit is 1%.
</details>

<details>
<summary><b>💡 T6 — detune</b></summary>

Find the peak: `[~, k] = max(abs(fft(y))); fPeak = (k-1)*fs/numel(y);` using only the first half of the FFT.
With 1 s of signal the resolution is 1 Hz, so `verifyEqual(testCase, fPeak, 900)` works for USB.
</details>

<details>
<summary><b>💡 T7 — frame-size invariance (the most valuable test)</b></summary>

Two **fresh** objects of each kind (or `reset` between runs). Run once with `1024`, once with something
irregular like `[1 7 100 333 1024 4096]`. Compare: `verifyLessThan(testCase, max(abs(a - b)), 1e-9)`.
If this fails, one of the three kinds of state from chapter 4 is wrong. Test the modulator and demodulator separately to find which.
</details>

<details>
<summary><b>💡 T8 — carrier change while running</b></summary>

Run half of the signal at 10 kHz, set `modulator.CarrierFrequency = 11e3`, then run the second half.
Look at `max(abs(diff(y)))` in a short window around the switch and compare it with the same value from a run
with a constant 11 kHz carrier. They should be about the same size; a phase jump would show up as a much larger spike.
(Use a constant-amplitude tone as input, so the only thing that can cause a spike is the carrier.)
</details>

<details>
<summary><b>💡 T9 — invalid carrier</b></summary>

```matlab
modulator = SSBModulator('SampleRate', 48e3, 'CarrierFrequency', 23e3);
verifyError(testCase, @() modulator(zeros(1024,1)), 'SSB:invalidCarrier');
```
</details>

## 6.5 Performance test (T11)

Phase 1 doesn't play anything live, so why measure this? Because it's the proof that the code **is fast enough for
real time** (SPEC NFR-1, rule R-4). If this test passes, Phase 2 won't fail on speed.

🛠 **Task 6.3** Measure the mean time of one modulator + demodulator step for N = 1024 with `timeit` (Lecture 8),
and check it's < 50% of 1024/48000 s. (After chapter 7, measure `processFrame` of the engine instead, which is the whole chain.)

<details>
<summary><b>💡 Hint</b></summary>

`timeit` needs a function handle with no inputs: make the objects and a frame first, then
`t = timeit(@() demodulator(modulator(x)));`. Warm the objects up first (one call each) so setup isn't timed.
</details>

🧠 **Q2.** Use the profiler (`profile on … profile viewer`, Lecture 8) on a 10 s run. Where does the time go?
Is it worth optimising?

<details>
<summary><b>✅ What to expect</b></summary>

Most of the time goes to `filter` with long FIRs and to `cos`/`sin`. Both are already built-in and vectorised.
The whole chain should take a small fraction of the frame time, so plotting in the GUI (chapter 8) will be the
real bottleneck, not the DSP.
</details>

## 6.6 Engine tests (come back here after chapter 7)

These go into a second file, `tests/engineTests.m`.

**T13 — the engine is ready for an endless stream.** This is the test that stands in for a microphone. Write a tiny test helper
class `tests/EndlessSource.m` (`< FrameSource`): `nextFrame` returns noise, `isDone` is always `false`, and
`totalFrames` is `Inf`. Give the engine a `FrameCallback` that calls `requestStop` once `info.frameIndex >= 50`.

🧠 **Q3.** What exactly should T13 verify? List at least three things.

<details>
<summary><b>✅ Answer</b></summary>

1. `run` **returns** at all (it doesn't hang). If it hangs, the stop flag isn't checked.
2. Exactly 50 frames were processed, e.g. the sink received 50 frames (use a `BufferSink`) or the stats say so.
3. The System objects are released afterwards: `isLocked(engine.Modulator)` is `false`.
4. Bonus: `totalFrames == Inf` didn't cause an error (e.g. in a progress calculation).
</details>

**T14 — two-step round trip.** Run `processFile` in "Modulator only" mode into a 32-bit float WAV, then in "Demodulator only" mode from that file
(with `FileSource` normalization off), and compare with a loopback run on the same input. The difference should be below 1e-6.

<details>
<summary><b>💡 Hint</b></summary>

`audiowrite(file, y, fs, 'BitsPerSample', 32)` stores floats, so there's (almost) no quantization error.
Write the temporary files with `tempname` and delete them at the end of the test
(`testCase.addTeardown(@delete, file)`).
</details>

---

🎯 **Checkpoint (M3):** `runtests('tests')` passes T1–T10. Commit. ➡️ [Chapter 7](07-engine.md)
(After chapter 7: T11, T13, T14 pass too, which completes **M4**.)
