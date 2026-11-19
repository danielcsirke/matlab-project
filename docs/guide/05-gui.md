# 5 — GUI with App Designer (3 h)

**Goal:** `app/SSBApp.mlapp`: the chapter 4 loop inside an app, with controls that act **while it streams**.
Milestone **M4**.

**Course link:** Lecture 6 (App Designer vs GUIDE; use App Designer), Exercise 6 `appdesigner/Solution/App.mlapp`
(a good template for layout and callbacks), Lecture 3 `try_callback.m`.

---

## 5.1 Layout (20 min)

```
┌─────────────────────────────────────────────┐
│ SOURCE   (•File ○Microphone)  [Browse…] name│
│ MODULATOR                                   │
│   Sideband   (•USB ○LSB)                    │
│   Carrier    ────●──────────  10.0 kHz      │
│ DEMODULATOR                                 │
│   Detune     ──────●────────  +0 Hz         │
│ OUTPUT                                      │
│   Listen to  [Output      ▼]                │
│   Volume     ──────●───                     │
│   ☑ Speaker   ☐ Save to file [Browse…] name │
│ [ Start ]  [ Stop ]                         │
│ Status: running · underruns 0               │
└─────────────────────────────────────────────┘
        (the spectrumAnalyzer opens in its own window)
```

Name the components in the Component Browser (`CarrierSlider`, `StartButton`, …), not `Slider_2`.

🧠 **Q1.** Carrier slider limits: what are they at 48 kHz, and when must you recompute them?

<details>
<summary><b>✅ Answer</b></summary>

3.6 … 20.4 kHz at 48 kHz (chapter 1, Q6). They depend on the sample rate, which comes from the file (or 48 kHz for the mic),
so recompute them when a file is chosen, and clamp the current value into the new range. Then the GUI can't even produce an invalid carrier,
and `SSB:invalidCarrier` is only a safety net.
</details>

## 5.2 App properties

In Code View, add **private properties**: `Source`, `Modulator`, `Demodulator`, `Speaker`, `FileWriter`, `Scope`, `IsRunning` (logical),
`CloseRequested` (logical), `InputFile`, `OutputFile`, `SampleRate`.

## 5.3 The loop inside the app: the key pattern (1 h)

A callback with a `while` loop **blocks** the app, unless the loop lets MATLAB process other events.
`drawnow limitrate` does that: it updates graphics at most 20×/s **and runs pending callbacks**, so Stop and the sliders work while the loop runs.

```
StartButtonPushed
 ├─ disable source controls, IsRunning = true
 ├─ try
 │   ├─ create source, Modulator, Demodulator, Speaker, Scope  (from the current control values)
 │   └─ while IsRunning
 │        ├─ x → Modulator → Demodulator → pick signal → volume → clip → Speaker; Scope
 │        ├─ update the underrun count in the status label (every ~20 frames is enough)
 │        └─ drawnow limitrate              ← Stop and slider callbacks run here
 ├─ catch err → uialert(app.UIFigure, err.message, 'Streaming stopped')
 ├─ release everything
 ├─ enable controls, IsRunning = false
 └─ if CloseRequested → delete(app)

StopButtonPushed → app.IsRunning = false     (that's all!)
```

🧠 **Q2.** Why must `StopButtonPushed` *only* set the flag, and not call `release` itself?

<details>
<summary><b>✅ Answer</b></summary>

The Stop callback runs **in the middle of** the Start callback's loop (during `drawnow`). If it released the speaker, the very next line of
the loop would use a released object. Only one place should create and release things: the Start callback.
</details>

🛠 **Task 5.1** Implement `StartButtonPushed` and `StopButtonPushed`. Reuse your chapter 4 code, and put the
"one frame" part in a private method `processFrame(app)` so `StartButtonPushed` stays readable.

<details>
<summary><b>✅ Solution — StartButtonPushed</b></summary>

```matlab
function StartButtonPushed(app, event)
    setRunning(app, true);                 % your helper: enables/disables controls, sets IsRunning
    try
        createObjects(app);                % your helper: Source, Modulator, Demodulator, Speaker, Scope
        underruns = 0;
        k = 0;
        while app.IsRunning
            underruns = underruns + processFrame(app);   % returns this frame's underrun count
            k = k + 1;
            if mod(k, 20) == 0
                app.StatusLabel.Text = sprintf('running · underruns %d', underruns);
            end
            drawnow limitrate
        end
    catch err
        uialert(app.UIFigure, err.message, 'Streaming stopped');
    end
    releaseObjects(app);
    setRunning(app, false);
    app.StatusLabel.Text = 'stopped';
    if app.CloseRequested
        delete(app);
    end
end
```
</details>

## 5.4 Controls that act while streaming (45 min)

Use the slider's **`ValueChangingFcn`** (fires continuously while you drag) and write directly into the tunable property:

```matlab
function CarrierSliderValueChanging(app, event)
    fc = event.Value*1e3;            % event.Value is the live value while dragging
    if ~isempty(app.Modulator)
        app.Modulator.CarrierFrequency   = fc;
        app.Demodulator.CarrierFrequency = fc;
    end
end
```

🛠 **Task 5.2** Do the same for the sideband buttons, the detune slider, the "listen to" dropdown and the volume.
The dropdown and the volume aren't System object properties, so store them in app properties and read them in `processFrame`.

🧠 **Q3.** Why must the demodulator's carrier follow the modulator's carrier slider, and which slider adds the receiver error?

<details>
<summary><b>✅ Answer</b></summary>

In this app both sides are in one program, so "the receiver is tuned to the transmitter" means the same `CarrierFrequency`.
The deliberate error comes only from the **detune** slider (`Demodulator.Detune`). That keeps the demo clear: move detune, hear Donald Duck.
</details>

🛠 **Task 5.3** Output choice: a "Speaker" and a "Save to file" checkbox (at least one must be on), with `uiputfile` for the file name.
`createObjects` creates `audioDeviceWriter` and/or `dsp.AudioFileWriter` accordingly; `processFrame` writes to whichever exists.
These choices are locked while running, like the source.

🧠 **Q4.** File source, "Save to file" only, no speaker: the loop never waits (chapter 4, Q2). What should happen when the file ends,
and what does that mean for `drawnow limitrate`?

<details>
<summary><b>✅ Answer</b></summary>

Read the file once and stop the loop when `isDone(app.Source)`, as if Stop had been pressed, then show "saved to …" in the status.
`drawnow limitrate` is still needed: without it, Stop and the window's close button don't respond during a long file.
Because the loop runs much faster than real time here, the progress is worth showing (e.g. seconds processed in the status label).
</details>

## 5.5 Closing the window while streaming (15 min)

If the window is deleted while the loop runs, the loop's next line touches deleted components and errors. Pattern:

```matlab
function UIFigureCloseRequest(app, event)
    if app.IsRunning
        app.CloseRequested = true;
        app.IsRunning = false;       % the loop ends; StartButtonPushed deletes the app
    else
        delete(app);
    end
end
```

## 5.6 Errors the user can cause (15 min)

🛠 **Task 5.4** Try to break your app, and make each case end in a clear `uialert` and a usable app:
- choose an 8 kHz WAV file
- choose an output file in a folder you can't write to
- choose "Microphone" with no microphone connected (or with mic access denied)
- press Start twice quickly
- press Start with no file chosen

<details>
<summary><b>💡 Hint</b></summary>

Most of these are covered by the `try/catch` around the loop, if `createObjects` throws errors with clear messages and IDs.
"Start twice" is solved by disabling the Start button while running (`setRunning`).
</details>

---

🎯 **Checkpoint (M4):** you can stream from a file and from the mic, drag carrier and detune and **hear** it change immediately,
switch USB/LSB and **see** the mirrored spectrum, save the output to a playable WAV, and Stop and closing the window both leave
no blocked audio device. ➡️ [Chapter 6](06-tests-deploy.md)
