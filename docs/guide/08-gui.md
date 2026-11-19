# 8 — The GUI with App Designer

**Goal:** `app/SSBApp.mlapp` covering FR-6 and FR-8. You'll reuse **everything** from chapters 5–7. The app
builds a settings struct from its controls, creates source, sink and engine, calls `run`, and draws what the
`FrameCallback` gives it. Milestone **M5**.

**Course link:** Lecture 6 (App Designer vs. GUIDE; use App Designer), Exercise 6 `appdesigner/Solution/App.mlapp`
(the throw-simulator app is a good template for layout and callbacks), Lecture 3 `try_callback.m`,
Lecture 5 `audiotest.m` (`audioplayer`), Lecture 7 `test_simple_gui_appdesigner.m` (testing apps).

---

## 8.1 Layout

Sketch it on paper first. A suggested layout using a `GridLayout` (it resizes nicely):

```
┌────────────────────────────┬──────────────────────────────────────────────┐
│ SOURCE                     │                                              │
│  Source   [Test tone   ▼]  │        Spectrum (dB vs kHz)                  │
│  File     [Browse…] name   │        — message  — SSB  — demodulated       │
│  Tone [1000] Hz  Dur [10] s│                                              │
│ SETUP (locked during a run)│                                              │
│  Mode     [Loopback    ▼]  ├──────────────────────────────────────────────┤
│  fs       [48000 ▼] N [1024▼]                                             │
│ MODULATOR                  │        Waveform  [Demodulated ▼]             │
│  Sideband (•USB ○LSB)      │                                              │
│  Carrier  ──●───── 10.0 kHz│                                              │
│ CHANNEL  Noise ─●── -60 dBFS├──────────────────────────────────────────────┤
│ DEMODULATOR                │ ▓▓▓▓▓▓▓▓░░░░ 62 %   frame 812/1310           │
│  Sideband (•USB ○LSB)      │ load: mean 3 %, max 9 %  → real-time capable │
│  Detune   ───●─── +0 Hz    │                                              │
│ OUTPUT                     │                                              │
│  Signal [Demodulated ▼]  Volume ──●──                                     │
│ [Process] [Stop]  [Play] [Save…]                                          │
│ 🔶 ☐ Live output (Phase 2)  │                                              │
└────────────────────────────┴──────────────────────────────────────────────┘
```

Give components meaningful names in the Component Browser (`CarrierSlider`, `ProcessButton`, …) instead of `Slider_3`.
The 🔶 checkbox is optional: you can leave it out, or add it now as **disabled** so the layout doesn't need to change in Phase 2.

## 8.2 App properties

In Code View add **private properties**, roughly:

| Property | Holds |
|---|---|
| `Settings` | the struct from `defaultSettings()`, kept in sync with the controls |
| `Engine` | the `SSBEngine` of the current/last run (empty before the first run) |
| `Sink` | the `BufferSink` with the last run's signals |
| `Monitor` | `SpectrumMonitor` |
| `IsRunning`, `CloseRequested` (logical) | see §8.3 and §8.5 |
| `SpectrumLines` (1×3), `WaveLine` | plot handles, created **once** |
| `Player` | the `audioplayer` (see Q3!) |

🧠 **Q1.** Why keep a `Settings` struct at all, instead of reading the controls directly when Process is pressed?

<details>
<summary><b>✅ Answer</b></summary>

Because `createSource`, `createSink`, `SSBEngine` and `processFile` all take the settings struct (chapter 7). The app becomes
"just another user" of the same code that the tests already cover. Each control's callback updates one field;
Process passes the struct on. That's less GUI code, and less untested GUI code.
</details>

## 8.3 The run: the important pattern

A callback that runs a long loop **blocks** the app, unless the loop gives MATLAB a chance to process other events.
`drawnow limitrate` does exactly that: it updates graphics at most 20 times/s **and processes pending callbacks**.
You call it from the engine's `FrameCallback` (SPEC rule R-5), so the Stop button and the sliders work during a run.

```
ProcessButtonPushed
 ├─ lock the setup controls, IsRunning = true
 ├─ try
 │   ├─ source = createSource(Settings); Sink = createSink(Settings)
 │   ├─ Engine = SSBEngine(Settings); Engine.FrameCallback = @(e, frames, info) onFrame(app, frames, info)
 │   ├─ Monitor = SpectrumMonitor(...); reset the plot lines
 │   └─ stats = run(Engine, source, Sink)          ← the loop lives in the engine, not in the app
 ├─ catch err → uialert(...)
 ├─ show final status (frames, mean/max load), enable Play/Save
 ├─ unlock controls, IsRunning = false
 └─ if CloseRequested → delete(app)

onFrame(app, frames, info)          (called every K frames by the engine)
 ├─ update(Monitor, frames); set SpectrumLines YData
 ├─ progress bar / status text (careful: info.totalFrames may be Inf)
 └─ drawnow limitrate                ← Stop and the sliders run here

StopButtonPushed → requestStop(app.Engine)       (that's all)
```

🧠 **Q2.** Why should `StopButtonPushed` *only* request a stop, and not release or delete anything itself?

<details>
<summary><b>✅ Answer</b></summary>

The Stop callback runs **in the middle of** the engine's loop (during `drawnow`). If it released the System objects, the loop's
next `processFrame` would use released objects. That would either error, or silently run a fresh setup and lose the filter state.
Only one place owns the lifetime: the engine's cleanup at the end of `run`.
</details>

🛠 **Task 8.1** Implement `ProcessButtonPushed`, `onFrame` (a private method) and `StopButtonPushed`.

<details>
<summary><b>✅ Solution — ProcessButtonPushed</b></summary>

```matlab
function ProcessButtonPushed(app, event)
    setRunningState(app, true);
    try
        source  = createSource(app.Settings);
        app.Sink   = createSink(app.Settings);
        app.Engine = SSBEngine(app.Settings);
        app.Engine.CallbackEvery = app.PlotEvery;
        app.Engine.FrameCallback = @(~, frames, info) onFrame(app, frames, info);
        app.Monitor = SpectrumMonitor(app.Settings.frameSize, app.Settings.sampleRate);
        resetPlots(app);
        stats = run(app.Engine, source, app.Sink);
        app.StatusLabel.Text = sprintf('%d frames · load mean %.1f %%, max %.1f %%', ...
            stats.frames, stats.meanLoad, stats.maxLoad);
        if stats.stopped
            app.StatusLabel.Text = [app.StatusLabel.Text ' · stopped'];
        end
    catch err
        uialert(app.UIFigure, err.message, 'Processing failed');
    end
    setRunningState(app, false);
    if app.CloseRequested
        delete(app);
    end
end
```
</details>

## 8.4 Parameter changes, also during a run

Use the slider's **`ValueChangingFcn`** (fires continuously while dragging) as well as `ValueChangedFcn`. Update the
`Settings` field **and**, if an engine exists, the tunable System object property:

```matlab
function CarrierSliderValueChanging(app, event)
    setCarrier(app, event.Value*1e3);    % event.Value is the live value; app.CarrierSlider.Value is not updated yet
end
```

On a long file you'll **see** the spectrum move while it processes. In Phase 2 you'll also **hear** it, with no extra code.

🧠 **Q3.** Should the demodulator carrier follow the modulator carrier slider?

<details>
<summary><b>✅ Answer</b></summary>

In loopback, yes: demod carrier = mod carrier, and the **detune** slider adds the offset. In demodulator-only mode the carrier slider
sets the receiver frequency directly. Write one `setCarrier(app, fc)` method that updates `Settings` and both objects (if they exist).
</details>

🧠 **Q4.** The carrier slider's limits must follow FR-9.1. When do you recompute them, and from what?

<details>
<summary><b>✅ Answer</b></summary>

Recompute them whenever the sample-rate dropdown changes, and once in `startupFcn`:
`Limits = [3400 + 200, fs/2 - 3400 - 200]/1e3` kHz. Then clamp the current value into the new range.
This way the GUI can't produce an invalid carrier, and the `SSB:invalidCarrier` check becomes a safety net, not the main protection.
</details>

## 8.5 Closing the window during a run

If the user closes the window during a run and the figure is deleted immediately, the engine's next callback
touches deleted components and errors out. Pattern:

```matlab
function UIFigureCloseRequest(app, event)
    if app.IsRunning
        app.CloseRequested = true;
        requestStop(app.Engine);      % run returns; ProcessButtonPushed then deletes the app
    else
        delete(app);
    end
end
```

## 8.6 Fast plotting

Recreating plots in every callback (`plot(...)`) is very slow, especially in `uifigure`s.

🛠 **Task 8.2** In `startupFcn` (or `resetPlots`), create the lines **once**:
`app.SpectrumLines = plot(app.SpectrumAxes, f/1e3, nan(numel(f), 3));`. In `onFrame`, only update `YData`.
Fix the axis limits once (`ylim`, `xlim`), so MATLAB doesn't recompute them every time.

🧠 **Q5.** How do you choose `CallbackEvery` (K)?

<details>
<summary><b>✅ Answer</b></summary>

In Phase 1 the engine runs as fast as it can, so the callback mostly costs time. Aim for about 10–20 GUI updates per second of
*wall time*: start with K ≈ 10 and watch the total processing time. 🔶 In Phase 2 the frames arrive in real time
(about 47 frames/s at N = 1024), so K ≈ 3 gives about 15 updates/s, and if underruns appear only when plots are visible, raise K.
</details>

## 8.7 Play and save (FR-6)

🛠 **Task 8.3** **Play:** take the chosen signal from `app.Sink` with `getSignal`, apply the volume, clip to [−1, 1], and play it with `audioplayer`.
**Save…:** `uiputfile` + `audiowrite(file, x, fs, 'BitsPerSample', 32)`.

🧠 **Q6.** You write `player = audioplayer(x, fs); play(player);` inside the callback, and you hear nothing (or only a blip). Why?

<details>
<summary><b>✅ Answer</b></summary>

`player` is a local variable. When the callback returns, it's destroyed and playback stops. Store it in an app property
(`app.Player = audioplayer(...)`) so it lives on. A Stop-playback button can then call `stop(app.Player)`.
</details>

> 🔶 **Optional — Phase 2.** With "Live output" ticked, `createSink` returns a `SpeakerSink` (or a `MultiSink` that plays **and** records),
> and the status line also shows underruns. Nothing else in this chapter changes. See [chapter 10](10-phase2-live-streaming.md).

## 8.8 Testing the app (optional, impressive)

Lecture 7's `test_simple_gui_appdesigner.m` shows `matlab.uitest.TestCase`: it can press buttons and move sliders
from a test. Even one test ("Process with a test tone fills the sink and enables Play") shows good engineering.

---

🎯 **Checkpoint (M5):** all FR-8 controls work. On a long file you can drag carrier and detune during the run and see the spectrum react,
Stop works, and closing the window during a run is clean. Play and Save work (T12). ➡️ [Chapter 9](09-deploy-and-polish.md)
