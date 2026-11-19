# 6 — Tests, deployment, demo (2 h)

**Goal:** four automated tests, a standalone Linux build, a README, and a rehearsed demo. Milestones **M5** and **M6**.

| Test | What | Passes if |
|---|---|---|
| T1 | USB, 1 kHz tone, $f_c$ = 10 kHz, $f_s$ = 48 kHz | power at 11 kHz ≥ 40 dB above power at 9 kHz |
| T2 | LSB, same | power at 9 kHz ≥ 40 dB above power at 11 kHz |
| T3 | loopback (modulator → demodulator), 1 kHz tone | output power at 1 kHz within ±1 dB of the input's |
| T4 | invalid carrier (23 kHz at 48 kHz) | throws `SSB:invalidCarrier` |

**Course link:** Lecture 7 (`mySqrt_testfunctions.m`: function-based tests), the deployment lecture,
`MatlabStyle2 book.pdf` (style guide).

---

## 6.1 Tests (1 h)

```matlab
function tests = ssbTests
%SSBTESTS Unit tests for SSBModulator and SSBDemodulator (T1-T4).
tests = functiontests(localfunctions);
end

function setupOnce(testCase)
    addpath(fullfile(fileparts(mfilename('fullpath')), '..', 'src'));
end
```

Every local function whose name starts with `test` is a test. Run them all with `runtests('tests')`.

Two helpers make all four tests short:

🛠 **Task 6.1** Write `y = runInFrames(sysObj, x, n)` (feeds `x` to a System object in frames of `n` and concatenates the outputs) and
`p = powerAt(x, fs, f)` (power at frequency `f` from a Hann-windowed FFT).

<details>
<summary><b>💡 Hint — powerAt</b></summary>

If `numel(x) == fs` (exactly 1 s), the FFT bins are exactly 1 Hz apart, so bin `f+1` is frequency `f`. Hann window in base MATLAB:
`w = 0.5 - 0.5*cos(2*pi*(0:n-1)'/(n-1))`.
</details>

Here's **T1 in full**, as a pattern for the others:

```matlab
function testUsbToneAboveCarrier(testCase)
    fs = 48e3;  fc = 10e3;  f0 = 1e3;
    x = 0.5*cos(2*pi*f0*(0:1.5*fs-1)'/fs);                  % 1.5 s tone
    modulator = SSBModulator('SampleRate', fs, 'CarrierFrequency', fc, 'Sideband', 'USB');
    s = runInFrames(modulator, x, 1024);
    s = s(end-fs+1:end);                                    % last 1 s: skip the start-up transient
    ratioDb = 10*log10(powerAt(s, fs, fc+f0) / powerAt(s, fs, fc-f0));
    verifyGreaterThan(testCase, ratioDb, 40);
end
```

🛠 **Task 6.2** Write T2 (LSB), T3 (loopback: output power at 1 kHz within ±1 dB of the input's), and T4 (invalid carrier).

<details>
<summary><b>💡 Hint — T4</b></summary>

```matlab
modulator = SSBModulator('SampleRate', 48e3, 'CarrierFrequency', 23e3);
verifyError(testCase, @() modulator(zeros(1024, 1)), 'SSB:invalidCarrier');
```
</details>

🧠 **Q1.** In T3, why compare **power at 1 kHz** instead of comparing the waveforms sample by sample?

<details>
<summary><b>✅ Answer</b></summary>

The filters delay the signal by a few hundred samples, and you don't know the exact number (it's inside the toolbox objects).
Comparing the power at one frequency is immune to delay, and it's exactly what T3 checks.
</details>

## 6.2 Deployment on Linux (30 min)

- In App Designer: **Share → Standalone Desktop App**. Or from code:
  `compiler.build.standaloneApplication('app/SSBApp.mlapp', 'OutputDir', 'deploy')`.
- MATLAB Compiler builds for the platform it runs on, so building on your Linux machine gives you a **Linux** app.
  (A Windows build would need MATLAB on Windows.)
- The output folder contains the executable and a launcher script, `run_SSBApp.sh`. Start it with the Runtime's install folder as the argument:
  ```bash
  ./run_SSBApp.sh /path/to/MATLAB_Runtime/R20xxx
  ```
- The target machine needs the free **MATLAB Runtime for Linux**, **same release** as your MATLAB (download it from MathWorks).
  On your own machine you can test with your MATLAB installation folder in place of the Runtime folder.
- `src/` classes are found automatically because the app calls them by name.

🛠 **Task 6.3** Build **early**, as soon as the GUI streams (don't leave it to the last evening). Check these on the built app:

| Check | Why |
|---|---|
| does the `spectrumAnalyzer` window open? | not every scope necessarily works in a compiled app. If it doesn't, plot the spectrum into a `UIAxes` yourself (+1 h: Hann window, FFT, update `YData` every few frames) |
| do the mic and speaker work? | on Linux, MATLAB talks to the sound card through ALSA. If the default device is wrong or silent, list the devices with `getAudioDevices(audioDeviceReader)` and set the `Device` property. Test on the machine you'll present on |
| can you choose files outside the install folder? | use `uigetfile`/`uiputfile`, never a fixed path; the install folder may not be writable |
| does a saved WAV play outside MATLAB? | checks that the file writer was released properly |

## 6.3 README and style (15 min)

- [ ] README: what it does, requirements (MATLAB release, DSP System + Audio Toolbox, Linux + MATLAB Runtime for the standalone app), how to run the app, `liveDemo` and the tests, a screenshot
- [ ] help comment (H1 line) on both classes: `help SSBModulator` shows something useful
- [ ] no magic numbers scattered around: 3400, 3600, 500 … as named properties or constants
- [ ] no Code Analyzer warnings (the orange marks)
- [ ] commits along the way, not one at the end

## 6.4 Demo script (15 min to rehearse)

1. **File, listen to "Output":** "This is the transmitted and received speech."
2. **Listen to "SSB":** "This is what the radio signal itself sounds like: everything shifted up by 10 kHz."
3. **USB ↔ LSB** with the scope visible: the spectrum mirrors around the carrier.
4. **Drag the carrier:** the SSB band moves in the scope while the output stays clean, because both sides follow.
5. **Drag detune ±150 Hz:** Donald Duck. One sentence of theory: every frequency shifts by the same Hz (chapter 1, Q5).
6. **Microphone:** say something live.
7. **Save to file:** process a file into a WAV, then play the WAV outside the app.
8. **Tests:** `runtests('tests')`, all green, and one sentence about what T1–T4 check.
9. ⭐ If you did the selective demodulator: two files on USB and LSB at once, switch the receiver between them.

Have `liveDemo` ready as a fallback in case the GUI or the presentation PC's audio misbehaves.

---

🎯 **Done (M5, M6):** tests pass, the standalone app runs, the README is written, and the demo is rehearsed. That's the project. 🎉
