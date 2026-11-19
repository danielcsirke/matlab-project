# 4 — The streaming loop (1 h)

**Goal:** `scratch/liveDemo.m`: a plain function (no GUI yet) that streams a file **or** the microphone through your modulator and
demodulator to the speaker **and/or** a file, with a live spectrum. Getting this working without a GUI makes chapter 5 much easier to debug,
and it's also your **fallback demo** if the GUI misbehaves at the exam. Milestone **M3**.

**Course link:** Lecture 5 `realtime_audio.m` (mic → scope → speaker) and `stream_function.m` ('SystemObject' case);
Exercise 5 `acousticRealTimeEcho.m` (a complete loop with `release` at the end). Your script has the same shape.

⚠️ **Headphones** for the microphone, otherwise you get feedback howl.

---

## 4.1 Audio devices

```matlab
mic     = audioDeviceReader('SampleRate', fs, 'SamplesPerFrame', 1024, 'NumChannels', 1);
speaker = audioDeviceWriter('SampleRate', fs);

x = mic();                       % blocks until a full frame is recorded
numUnderrun = speaker(y);        % queues y for playback; >0 means you were too slow (a click)
```

Writing to a file works the same way, frame by frame:

```matlab
fileOut = dsp.AudioFileWriter('Filename', 'out.wav', 'SampleRate', fs);
fileOut(y);                      % appends one frame to the file
release(fileOut);                % finalizes the file. Without this the WAV may be unreadable!
```

🧠 **Q1.** There's no `pause` in the Lecture 5 loops. What keeps the loop from running too fast, (a) with the mic, (b) with a file?

<details>
<summary><b>✅ Answer</b></summary>

(a) `mic()` waits until 1024 new samples have been recorded, so the sound card's clock sets the pace.
(b) The file reader doesn't wait, but `speaker(y)` blocks once its buffer is full, so the playback clock sets the pace.
Never add `pause`; it makes playback choppy (compare the `'Function'` case in `stream_function.m`).
</details>

🧠 **Q2.** File in, file out, **no speaker**: what paces the loop now? Is that a problem, and when should the loop stop?

<details>
<summary><b>✅ Answer</b></summary>

Nothing paces it, so it runs as fast as the CPU allows, usually many times faster than real time. That's not a problem, it's useful:
it's the "process a whole file" case. But a fixed `duration` makes no sense then. Read the file once (`'PlayCount', 1`) and stop when
`isDone(reader)` is true. With the mic there's no end, so stop after `duration` seconds (or with the Stop button in the GUI).
</details>

## 4.2 The function

🛠 **Task 4.1** Write `liveDemo.m`. Settings at the top: `useMic` (true/false), `fileName`, `fc`, `sideband`, `detune`, `listenTo`
(`'message'|'ssb'|'output'`), `volume`, `playToSpeaker` (true/false), `outputFile` (`''` = don't save), `duration` (e.g. 30 s, used for the mic).

Steps:
1. Create the source: `dsp.AudioFileReader` or `audioDeviceReader`. For a file, take `fs` from the file and reject anything below 44.1 kHz (chapter 1, Q6).
   Loop the file forever (`'PlayCount', Inf`) when you only play it, and read it once when you write it to a file (Q2).
2. Create `SSBModulator`, `SSBDemodulator`, a `spectrumAnalyzer` with three channels (`'ChannelNames', {'message','SSB','output'}`, `'ShowLegend', true`),
   and the outputs you need: `audioDeviceWriter` if `playToSpeaker`, `dsp.AudioFileWriter` if `outputFile` isn't empty.
3. Make sure everything is **released even if you press Ctrl+C** or an error happens. For the file writer, that's what makes the WAV valid.
4. Loop: read → mono → modulate → demodulate → pick the signal to hear → `volume` → clip to [−1, 1] → speaker and/or file; scope gets `[x, s, y]`.
5. Count underruns and print the total at the end.

<details>
<summary><b>💡 Hint — guaranteed cleanup</b></summary>

`onCleanup` (Lecture 7) runs when its variable is destroyed, including on Ctrl+C and errors:
```matlab
cleaner = onCleanup(@() releaseAll(source, modulator, demodulator, speaker, scope));
```
with a local function that loops over `varargin` and calls `release` on each. Outputs you didn't create (e.g. no file writer)
can simply be `[]`, so let `releaseAll` skip empty inputs.

⚠️ This only works inside a **function**: a function's variables are destroyed when it exits, but a script's variables stay
in the base workspace, so `cleaner` would never fire. Make `liveDemo` a function (`function liveDemo()` with the settings
as variables at the top, or as Name=Value arguments).
</details>

<details>
<summary><b>💡 Hint — "pick the signal to hear" without an if-chain</b></summary>

Put the three signals in a struct: `sig = struct('message', x, 'ssb', s, 'output', y);` and use `sig.(listenTo)`.
The GUI will use the same trick with a dropdown's value.
</details>

<details>
<summary><b>✅ Solution — the loop</b></summary>

```matlab
underruns = 0;
tStart = tic;
while keepRunning(source, useMic, tStart, duration)
    x = mean(source(), 2);
    s = modulator(x);
    y = demodulator(s);
    sig = struct('message', x, 'ssb', s, 'output', y);
    out = max(min(volume*sig.(listenTo), 1), -1);
    if ~isempty(speaker)
        underruns = underruns + speaker(out);
    end
    if ~isempty(fileOut)
        fileOut(out);
    end
    scope([x, s, y]);
end
fprintf('Underruns: %d\n', underruns);
```
with a local function: for the mic, `keepRunning` is `toc(tStart) < duration`; for a file it's `~isDone(source)`
(which is never true with `'PlayCount', Inf`, so add the time limit there too).
</details>

## 4.3 Experiments (write down what you hear, it's material for the presentation)

| Experiment | Expected |
|---|---|
| listen to `'ssb'` | garbled, "underwater" speech: every frequency shifted up by $f_c$ |
| listen to `'output'` | the original speech back |
| `detune` = +150 Hz, USB, then LSB | "Donald Duck", the voice moves down (USB) or up (LSB) |
| change `sideband` from USB to LSB | output sounds the same (the simple demodulator doesn't care, chapter 1 Q4), but the scope shows the mirrored spectrum |
| frame size 256 instead of 1024 | do underruns start? the scope update costs time too |
| file in, file out, no speaker | finishes much faster than the file's length; open `out.wav` and listen to it |

🧠 **Q3.** Underruns only appear while the scope window is open or being resized. What does that tell you?

<details>
<summary><b>✅ Answer</b></summary>

The DSP is fast enough; **drawing** is the bottleneck. Remedies: a larger frame size, or giving the scope only every 2nd–3rd frame.
Keep this in mind for the GUI, where plotting is also the expensive part.
</details>

---

🎯 **Checkpoint (M3):** `liveDemo` runs 30 s from a file and from the mic, with 0 underruns after start-up; file → file produces a
playable WAV; and Ctrl+C leaves no device blocked (you can run it again immediately). ➡️ [Chapter 5](05-gui.md)
