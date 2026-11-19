# 10 — 🔶 Phase 2: live streaming (optional)

> 🔶 **This whole chapter is optional.** Do it only after M6, or if your instructor expects live audio
> (SPEC §12, question 4).

**Goal:** mic → modulator → demodulator → speaker, live, with the GUI sliders working in real time.
If Phase 1 follows SPEC §9.2, this means **adding** the items in SPEC §9.1 and changing nothing else.
Milestone *M7*. Expected effort: 8–12 hours, mostly testing with real devices.

**Course link:** Lecture 5: `realtime_audio.m`, `stream_function.m` ('SystemObject' case),
Exercise 5 `acousticRealTimePlayback.m`, `acousticRealTimeEcho.m`.

⚠️ **Use desktop MATLAB.** Live audio devices might not be available in MATLAB Online.
Check your toolboxes with `ver`: you need `audioDeviceWriter` (Audio Toolbox or DSP System Toolbox) and, for the mic,
`audioDeviceReader` (Audio Toolbox).

---

## 10.0 Before you start: the honesty check

🧠 **Q0.** Search your Phase 1 code for each of these. Every hit is a place where Phase 2 would force you to rewrite something:

- `hilbert(`, or `fft(` on a whole signal inside the chain (rule R-1)
- `max(abs(` used to scale output (R-3)
- `isa(`, `FileSource`, `ToneSource` anywhere except `createSource` (R-6)
- a `while` loop in the app instead of in the engine (R-5)
- anything assuming `totalFrames` is finite (R-2)

If the list is empty and T7, T11 and T13 pass, carry on.

## 10.1 Audio device System objects

```matlab
reader = audioDeviceReader('SampleRate', fs, 'SamplesPerFrame', N, 'NumChannels', 1);
writer = audioDeviceWriter('SampleRate', fs);

[x, numOverrun] = reader();     % blocks until N new samples are available
numUnderrun     = writer(y);    % queues y for playback
release(reader); release(writer);
```

- **Overrun:** you didn't read fast enough, so the input buffer filled up and samples were lost.
- **Underrun:** you didn't write fast enough, so the speaker ran out of samples (you hear a click or gap).
- Both counts should be **0 in steady state**. A few at start-up are normal.
- `getAudioDevices(reader)` lists the devices. Not every sound card supports every sample rate, so try 48 kHz and 44.1 kHz.

🧠 **Q1.** In Phase 1 the engine ran "as fast as possible". What sets the pace now, with a mic? And with a looping file + speaker?

<details>
<summary><b>✅ Answer</b></summary>

With a mic, `reader()` blocks until a frame is ready, so the hardware clock sets the pace. With a looping file and
a speaker, `writer()` blocks once its buffer is full, so the speaker sets the pace. In both cases **the engine doesn't change**:
the blocking happens inside `nextFrame` or `write`. Don't add `pause()`; compare with the choppy `sound()` + `pause()` version
in `stream_function.m`.
</details>

🧠 **Q2.** Frame size trade-off: what do you gain and lose with N = 256 versus N = 4096?

<details>
<summary><b>✅ Answer</b></summary>

Small N gives low latency (5 ms per frame) but leaves little time per frame, and the fixed per-call overhead (MATLAB
dispatch, GUI callbacks) matters more, so it's more prone to underruns. Large N is robust but adds 85 ms of
latency and makes slider changes feel sluggish. 1024 is a good default.
</details>

## 10.2 The new classes

🛠 **Task 10.1 — `MicSource < FrameSource`.** The constructor creates the `audioDeviceReader`; `nextFrame` reads one frame
(and adds up overruns in an `Overruns` property); `isDone` → `false`; `totalFrames` → `Inf`; `release` releases the reader.

🛠 **Task 10.2 — `SpeakerSink < FrameSink`.** The constructor creates the `audioDeviceWriter`; `write` plays `frames.monitor` and adds up
underruns in an `Underruns` property; `release` releases the writer.

<details>
<summary><b>✅ Solution — MicSource</b></summary>

```matlab
classdef MicSource < FrameSource
    %MICSOURCE Live microphone input as a FrameSource (Phase 2).
    properties (SetAccess = private)
        Overruns = 0
    end
    properties (Access = private)
        Reader
    end
    methods
        function obj = MicSource(fs, frameSize)
            obj.SampleRate = fs;
            obj.FrameSize  = frameSize;
            obj.Reader = audioDeviceReader('SampleRate', fs, ...
                'SamplesPerFrame', frameSize, 'NumChannels', 1);
        end
        function x = nextFrame(obj)
            [x, numOverrun] = obj.Reader();
            obj.Overruns = obj.Overruns + numOverrun;
            x = double(x);
        end
        function tf = isDone(~)
            tf = false;
        end
        function n = totalFrames(~)
            n = Inf;
        end
        function release(obj)
            release(obj.Reader);
        end
    end
end
```
</details>

🛠 **Task 10.3 (optional) — `MultiSink < FrameSink`.** Holds a cell array of sinks and forwards `write` and `release` to each,
so you can listen live **and** record into a `BufferSink` at the same time.

🧠 **Q3.** A live `BufferSink` keeps growing as long as the stream runs. What limit makes sense, and what should happen when it's reached?

<details>
<summary><b>✅ Answer</b></summary>

A maximum duration (e.g. 5 minutes ≈ 14.4 M samples × 4 signals × 8 bytes ≈ 460 MB, so maybe less). When it's full, stop
recording but **keep streaming**. Don't throw an error that kills the live run. Show "recording full" in the status line.
</details>

## 10.3 Wiring it in

🛠 **Task 10.4** `createSource`: add `case 'mic'`. `createSink`: return a `SpeakerSink` (or `MultiSink` of speaker + buffer) when
`settings.liveOutput` is true. Add `liveOutput` (false) to `defaultSettings`. If you did the 🔶 `Loop` option in chapter 7, pass
`settings.loop` to the file and tone sources.

🛠 **Task 10.5** GUI: add "Microphone" to the source dropdown, enable the "Live output" checkbox, and show
over/underruns in `onFrame` when the source or sink has them (`isprop(source, 'Overruns')`).
Adjust `CallbackEvery` for real time (chapter 8, Q5).

🧠 **Q4.** Which files did you have to change in this chapter, and which did you have to **open**? Compare with SPEC §9.1.

<details>
<summary><b>✅ What it should be</b></summary>

New: `MicSource.m`, `SpeakerSink.m`, (`MultiSink.m`). Changed: `createSource.m`, `createSink.m`, `defaultSettings.m`, the app
(dropdown item, checkbox, status text). **Not touched:** everything in `src/dsp/`, `SSBEngine.m`, `SpectrumMonitor.m`, the tests.
If you had to change the engine or the DSP, write down why. That's a good lesson for the presentation.
</details>

## 10.4 Experiments

| Experiment | What to look for |
|---|---|
| Mic loopback with **headphones** | your voice, ~19 ms + buffer late, clean |
| Monitor = SSB | garbled speech: every frequency is shifted up by $f_c$ |
| Drag detune during live speech | the Donald Duck effect, live |
| Switch the receiver to the wrong sideband | near silence (only noise) |
| N = 128 | do underruns start? what is the max load? |
| Two laptops: one runs modulator only → speaker, the other mic → demodulator only | does it survive the acoustic path? (fun demo, FR-3) |

⚠️ Use **headphones** for mic loopback, otherwise you'll get acoustic feedback (howling).

## 10.5 Phase 2 tests

The DSP and engine tests already cover the logic. Hardware can't be unit-tested reliably, so add:

- a **manual test protocol** (a checklist in the README): 60 s live loopback with 0 underruns after start-up; sliders, Stop and window close during live streaming;
- optionally an automated test that **skips itself** when no audio device is present:
  `testCase.assumeNotEmpty(getAudioDevices(audioDeviceWriter))`.

---

🎯 **Checkpoint (M7):** 60 s of live loopback with 0 underruns after start-up, sliders audible in real time, and you only added
code: the DSP and the engine stayed untouched.
