# 7 — Sources, sinks and the engine

**Goal:** build the machinery that feeds frames through your System objects: sources, sinks, the
`SSBEngine` run loop, a `SpectrumMonitor`, and the command-line `processFile`. This is milestone **M4**,
and it's the chapter that makes the code **streaming-ready** (SPEC §9).

**Course link:** Lecture 5 (objects, handle classes, `Custom/myFrac.m`; `stream_function.m` has the same
"read frame → process → write frame" loop), Lecture 7 (`onCleanup`, error handling), Lecture 8 (preallocation).

---

## 7.1 The idea: program against interfaces

```
FrameSource ──▶ SSBEngine (modulator → channel → demodulator) ──▶ FrameSink
 ToneSource                         │ every K frames                BufferSink
 FileSource                         ▼                              🔶 SpeakerSink
🔶 MicSource                   FrameCallback (GUI, plots, Stop)
```

The engine only knows "something that gives frames" and "something that takes frames". It doesn't know
whether that's a file or a microphone. That's why Phase 2 is just two new classes (SPEC rule R-6).

🧠 **Q1.** Which of these would break the "engine doesn't care" rule? (a) `if isa(source, 'FileSource')` inside the engine,
(b) the engine calling `totalFrames(source)` for the progress bar, (c) the engine calling `audioread`.

<details>
<summary><b>✅ Answer</b></summary>

(a) and (c) break it: the engine would depend on a concrete class or a file. (b) is fine because it's part of the interface,
and a live source simply answers `Inf` (that's why T13 checks `Inf` doesn't cause an error).
</details>

## 7.2 Abstract classes in MATLAB

The interfaces are **abstract handle classes** (SPEC §8.2):

```matlab
classdef (Abstract) FrameSource < handle
    properties (SetAccess = protected)
        SampleRate
        FrameSize
    end
    methods (Abstract)
        x  = nextFrame(obj)
        tf = isDone(obj)
        n  = totalFrames(obj)
    end
    methods
        function release(obj) %#ok<MANU>
        end
    end
end
```

A subclass is declared with `classdef FileSource < FrameSource`, and it **must** implement every abstract
method, otherwise MATLAB refuses to create it. That's the compiler (well, MATLAB) checking your contract.

🧠 **Q2.** Why `handle` and not a value class? (Remember `myFrac` from Lecture 5, and think about what `nextFrame` must change.)

<details>
<summary><b>✅ Answer</b></summary>

`nextFrame` must advance a read position. With a value class, `x = nextFrame(src)` would change a **copy**, and the
next call would return the same frame again (unless you wrote `[src, x] = nextFrame(src)` everywhere). Handle objects are
references: the engine, the GUI and the test all see the same object.
</details>

🛠 **Task 7.1** Write `FrameSource.m` and `FrameSink.m` (`write(obj, frames)` abstract, `release` default no-op) in `src/io/`.

## 7.3 Sources

🛠 **Task 7.2 — `ToneSource(frequencies, duration, fs, frameSize)`.** One or more tones (two-tone = `[700 1900]`),
equal amplitudes summing to at most 0.9. Use **phase accumulators** (chapter 4) so it's click-free. `isDone` becomes true after `duration` seconds;
the last frame may be shorter.

🛠 **Task 7.3 — `FileSource(fileName, fs, frameSize, 'Normalize', true)`.** In the constructor: `audioread`, mono (`mean(x,2)`),
resample if needed, normalize **once** if requested (this is allowed by R-1, since it's preparation in the source, not processing in the chain).
Throw `SSB:invalidFile` with a clear message if the file can't be read.

<details>
<summary><b>💡 Hint — resampling without a hard toolbox dependency</b></summary>

```matlab
if fsFile ~= fs
    if ~isempty(which('resample'))          % Signal Processing Toolbox present
        sig = resample(sig, fs, fsFile);
    else                                    % base-MATLAB fallback (lower quality)
        tOld = (0:numel(sig)-1)'/fsFile;
        tNew = (0:1/fs:tOld(end))';
        sig  = interp1(tOld, sig, tNew, 'linear');
    end
end
```
</details>

<details>
<summary><b>✅ Solution — FileSource frame methods</b></summary>

```matlab
function x = nextFrame(obj)
    last = min(obj.Position + obj.FrameSize - 1, numel(obj.Signal));
    x = obj.Signal(obj.Position:last);
    obj.Position = last + 1;
end

function tf = isDone(obj)
    tf = obj.Position > numel(obj.Signal);
end

function n = totalFrames(obj)
    n = ceil(numel(obj.Signal) / obj.FrameSize);
end
```
</details>

> 🔶 **Optional — Phase 2.** Add a `Loop` property to both sources. When it's true, `isDone` is always false, `totalFrames` is `Inf`,
> and reading wraps around: `idx = mod(obj.Position - 1 + (0:N-1)', numel(obj.Signal)) + 1`. A looping file is a good
> live test signal. You don't need it in Phase 1.

## 7.4 The `BufferSink`

It stores the four signals of each frame (`message`, `ssb`, `demodulated`, `monitor`) so the GUI can plot, play and save them afterwards.

🧠 **Q3.** You don't know in advance how many frames will come (in Phase 2, potentially endless). How do you avoid
`data = [data; frame]` (which copies the whole array every frame, Lecture 8)?

<details>
<summary><b>✅ Answer</b></summary>

Preallocate a capacity (e.g. 10 s), keep a `Count`, and when it's full, **double** the capacity. Doubling means only
about log₂(n) copies in total instead of one per frame. A trick to grow a column with zeros: `A(newCapacity, 1) = 0;`.
(An endless live recording would need a limit too, but that's a Phase 2 question.)
</details>

🛠 **Task 7.4** Write `BufferSink` with `write(obj, frames)`, `getSignal(obj, name)` (returns the filled part only), and a `Count` property.

## 7.5 The engine

🛠 **Task 7.5** Write `SSBEngine` (SPEC §8.2). Two core methods:

**`frames = processFrame(obj, x)`** runs one frame through the chain according to `Mode`:

```
'loopback':     ssb = Modulator(x);  demodulated = Demodulator(ssb + noise)
'modulator':    ssb = Modulator(x);  demodulated = zeros
'demodulator':  ssb = x;             demodulated = Demodulator(x)
monitor = clip(Volume * frames.(MonitorSignal), -1, 1)       ← fixed gain, never "divide by max" (R-3)
```

**`stats = run(obj, source, sink)`** is the loop:

```
StopRequested = false
make sure everything is released at the end, even on error     ← onCleanup
while not isDone(source) and not StopRequested
    x = nextFrame(source)
    time processFrame(x)  → load %
    write(sink, frames)
    every CallbackEvery frames: FrameCallback(obj, frames, info)
return frames processed, mean/max load, whether it was stopped
```

🧠 **Q4.** `requestStop` only sets `StopRequested = true`. But `run` is busy in its `while` loop, so how can *anything*
call `requestStop` while it runs?

<details>
<summary><b>✅ Answer</b></summary>

Through the `FrameCallback`. In the GUI it calls `drawnow limitrate`, which processes pending GUI events, including a
pressed Stop button, whose callback calls `requestStop`. In tests (T13) the callback calls `requestStop` itself.
Without a callback that yields to the event queue, nothing can interrupt the loop. That's SPEC rule R-5.
</details>

🧠 **Q5.** Why release the System objects at the **end** of every run (in the cleanup)?

<details>
<summary><b>✅ Answer</b></summary>

(1) Released objects are unlocked, so the GUI can change non-tunable settings (sample rate, frame size) between runs.
(2) The next run's first call triggers `setupImpl` again, so every run starts from zero state.
(3) In Phase 2, `release` frees the sound card. If a run crashes without releasing it, the device stays blocked until MATLAB restarts.
</details>

<details>
<summary><b>✅ Solution — run</b></summary>

```matlab
function stats = run(obj, source, sink)
    obj.StopRequested = false;
    cleaner = onCleanup(@() releaseAll(obj, source, sink)); %#ok<NASGU>
    total     = totalFrames(source);
    frameTime = source.FrameSize / source.SampleRate;
    k = 0;  sumLoad = 0;  maxLoad = 0;
    while ~isDone(source) && ~obj.StopRequested
        x  = nextFrame(source);
        t0 = tic;
        frames = processFrame(obj, x);
        loadPct = 100*toc(t0)/frameTime;
        write(sink, frames);
        k = k + 1;
        sumLoad = sumLoad + loadPct;
        maxLoad = max(maxLoad, loadPct);
        if ~isempty(obj.FrameCallback) && mod(k, obj.CallbackEvery) == 0
            info = struct('frameIndex', k, 'totalFrames', total, 'loadPercent', loadPct);
            obj.FrameCallback(obj, frames, info);
        end
    end
    stats = struct('frames', k, 'meanLoad', sumLoad/max(k, 1), ...
                   'maxLoad', maxLoad, 'stopped', obj.StopRequested);
end
```
with a private `releaseAll(obj, source, sink)` that calls `release` on the modulator, demodulator, source and sink.
</details>

## 7.6 The `SpectrumMonitor`

The GUI's spectrum plot must build up **while** processing (it's fed from the callback), not by an FFT of the whole result at the end.
That's the same code that will drive a live spectrum in Phase 2.

🛠 **Task 7.6** `SpectrumMonitor(frameSize, fs)` with `update(obj, frames)` (Hann window, FFT, exponential average
$P \leftarrow \alpha P + (1-\alpha)|X|^2$ per signal, $\alpha \approx 0.8$), `Frequencies` (Hz) and `spectraDb(obj)` (N/2+1 × 3 matrix).

🧠 **Q6.** The last frame of a file can be shorter than N. What happens in your FFT, and what's the simplest fix?

<details>
<summary><b>✅ Answer</b></summary>

`fft(x .* window)` fails because the sizes don't match. Zero-pad first: `x(end+1:N) = 0;` (this does nothing when `numel(x) == N`).
</details>

## 7.7 Settings, factories and `processFile`

The GUI, `processFile` and the tests all need the same parameters. Collect them in **one** settings struct:

🛠 **Task 7.7** `settings = defaultSettings()` returns a struct with every parameter and its default value: `sampleRate`, `frameSize`,
`source` (`'tone'|'twotone'|'file'`), `fileName`, `toneFrequency`, `duration`, `normalize`, `mode`, `carrierFrequency`,
`modSideband`, `demodSideband`, `detune`, `noiseLevelDb`, `monitorSignal`, `volume`.

🛠 **Task 7.8** `source = createSource(settings)` and `sink = createSink(settings)`: a `switch` on `settings.source` (and for now
`sink` is always a `BufferSink`). Unknown names throw `SSB:unknownSource`. **These two files are the only place that knows the concrete
classes** (R-6), so they're the only place Phase 2 touches.

🛠 **Task 7.9** `processFile(inFile, outFile, options)`: start from `defaultSettings()`, override with `options`
(an `arguments` block with Name=Value options is nice here, see Lecture 7), create source, sink and engine, `run`, and `audiowrite` the requested signal.

<details>
<summary><b>💡 Hint — overriding defaults from Name=Value pairs</b></summary>

```matlab
function stats = processFile(inFile, outFile, options)
    arguments
        inFile  (1,1) string
        outFile (1,1) string
        options.Mode (1,1) string = "loopback"
        options.Output (1,1) string = "demodulated"
        % … one line per option you want to expose
    end
    settings = defaultSettings();
    settings.source   = 'file';
    settings.fileName = inFile;
    settings.mode     = char(options.Mode);
    % …
end
```
</details>

## 7.8 Close M4 with the tests

Go back to [chapter 6, §6.6](06-testing.md#66-engine-tests-come-back-here-after-chapter-7) and write T13 and T14, and switch T11 to measure `processFrame`.

> 🔶 **Optional — Phase 2 preview.** Look at your engine and ask: "If `source` were a microphone and `sink` a speaker,
> what would I need to change?" If the honest answer is "nothing in the engine", Phase 1 is done right.
> The details are in [chapter 10](10-phase2-live-streaming.md).

---

🎯 **Checkpoint (M4):** `processFile('speech.wav', 'out.wav')` works from the command line; T11, T13 and T14 pass. ➡️ [Chapter 8](08-gui.md)
