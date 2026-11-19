# Specification — SSB-SC Modulator / Demodulator (streaming-ready)

**Course:** Mérnöki problémamegoldás MATLAB-ban, BME, 2026 fall
**Status:** Draft v0.2 (2026-11-19). Live audio streaming is moved out of scope, but the architecture is built for it.

---

## 1. Purpose

A MATLAB application that **modulates** and **demodulates** audio files using
**single-sideband suppressed-carrier (SSB-SC)** modulation, with selectable
**upper (USB)** or **lower (LSB)** sideband. It has a graphical user interface and
must be deployable as a standalone application.

**Key design decision:** although this version processes **files**, all signal
processing is **frame-based with internal state**, exactly as a live stream needs.
Files are fed through the chain frame by frame, using the same interfaces a
microphone and speaker will use later. Adding live streaming later must only need
**new source/sink classes plus small GUI additions**, with no change to the DSP,
the processing engine or the tests (see §9).

## 2. Scope

### In scope (this version)
- SSB-SC modulation by the **phasing (Hilbert) method**, USB or LSB
- **Sideband-selective coherent demodulation** (phasing demodulator), USB or LSB
- Frame-based processing of WAV files and generated test signals
- Playback of the results (`audioplayer`), saving them to WAV, spectrum and waveform plots
- GUI built with App Designer, deployable with MATLAB Compiler

### Prepared for, not implemented (Phase 2, see §9)
- Live microphone input and speaker output (real-time streaming)

### Out of scope (possible extensions, see §13)
- Transmitted / pilot carrier, carrier recovery, PLL
- Independent sideband (two different messages on USB and LSB)
- Radio hardware (SDR)

## 3. Terms

| Term | Meaning |
|---|---|
| m(t) | message (baseband audio), band-limited to 300–3400 Hz |
| m̂(t) | Hilbert transform of m(t) |
| f_c | carrier frequency (Hz) |
| USB / LSB | upper / lower sideband; sideband sign `sb = +1` for USB, `-1` for LSB |
| Δf | receiver detuning: receiver local oscillator (LO) runs at f_c + Δf |
| Frame | block of `N` consecutive samples, a real `double` column vector |
| State | everything a block must remember between frames (filter memories, delay lines, oscillator phase) |
| Source | object that delivers frames (`FrameSource`) |
| Sink | object that consumes frames (`FrameSink`) |
| Run | one pass of the engine from a source to a sink |

## 4. Environment

| Item | Requirement |
|---|---|
| MATLAB | Recent release (R2023b or newer recommended). Desktop or MATLAB Online, since this version needs no audio devices. |
| Toolboxes, this version | **Base MATLAB only.** Filters are designed by hand (windowed design, `besseli` for the Kaiser window) and run with `filter`. Playback with `audioplayer`, files with `audioread`/`audiowrite`. |
| Optional | Signal Processing Toolbox (`resample` for better resampling quality; `freqz`, `kaiser`, `hilbert` for cross-checking), Communications Toolbox (`ssbmod`/`ssbdemod` **only for cross-checking in tests**) |
| Phase 2 (streaming) | `audioDeviceWriter` (Audio Toolbox or DSP System Toolbox), `audioDeviceReader` (Audio Toolbox), desktop MATLAB |
| Deployment | MATLAB Compiler |

## 5. Functional requirements

### FR-1 Modulator
- FR-1.1 Implements the phasing method:
  `s[n] = m_d[n]·cos(φ[n]) − sb·m̂[n]·sin(φ[n])`, where `m_d` is `m` delayed to
  match the Hilbert filter's group delay.
- FR-1.2 Sideband selectable: USB or LSB. It is a tunable property, so it can be changed between frames.
- FR-1.3 Carrier is suppressed (no carrier component in the output).
- FR-1.4 Carrier frequency `f_c` is tunable and can be changed between frames
  **without phase jumps** (phase-accumulator oscillator).
- FR-1.5 The message is band-limited to 300–3400 Hz by an input band-pass filter
  before modulation.
- FR-1.6 Frame-based with internal state: the output must not depend on how the
  input is split into frames (NFR-3).

### FR-2 Demodulator
- FR-2.1 Coherent, sideband-selective phasing demodulator:
  `I = LPF(r·cos φ)`, `Q = LPF(−r·sin φ)`, `y = I_d − sb·H{Q}`.
- FR-2.2 Sideband selectable (USB/LSB) and tunable. The opposite sideband is rejected.
- FR-2.3 LO frequency = `f_c + Δf`. Detuning `Δf` is tunable in the range ±500 Hz.
  (A detuned receiver shifts the recovered audio: USB by −Δf, LSB by +Δf.)
- FR-2.4 Frame-based with internal state (same as FR-1.6).

### FR-3 Operating modes
| Mode | Chain |
|---|---|
| Loopback (default) | source → modulator → channel → demodulator → sink |
| Modulator only | source → modulator → sink (e.g. save the SSB signal to a WAV) |
| Demodulator only | source (SSB signal, e.g. a WAV saved in "Modulator only" mode) → demodulator → sink |

### FR-4 Signal sources (this version)
- FR-4.1 Test tone (frequency 300–3400 Hz, chosen duration)
- FR-4.2 Two-tone test signal (e.g. 700 Hz + 1900 Hz, chosen duration)
- FR-4.3 WAV file: converted to mono, resampled to the working sample rate if needed
  (`resample` if available, `interp1` otherwise), and normalized **once when loaded**
  (option `Normalize`, default true; off for files that already contain an SSB signal at a known level).

### FR-5 Channel (loopback mode only)
- FR-5.1 Optional additive white Gaussian noise, level set in dBFS (−80 … −10 dBFS, or off).

### FR-6 Output
- FR-6.1 Each run collects the **message, SSB and demodulated** signals in a `BufferSink`.
- FR-6.2 Play any of the three (selectable) with `audioplayer`, using volume control and limiting to [−1, 1].
  No normalization by the whole signal's maximum: the output level is set by a fixed gain only (see R-3).
- FR-6.3 Save any of the three to a WAV file.

### FR-7 Processing engine
- FR-7.1 `SSBEngine` runs a source through the chain into a sink, frame by frame, until
  the source reports `isDone` **or** a stop is requested.
- FR-7.2 Every K frames it calls an optional `FrameCallback` (progress, plotting, `drawnow`),
  which the GUI uses for a progress bar, live plot updates and a working Stop button.
- FR-7.3 It measures processing time per frame and reports the load (% of frame duration),
  which is evidence of real-time capability (NFR-1).
- FR-7.4 The same engine runs from the command line without a GUI:
  `processFile(inFile, outFile, options)` is a thin wrapper around it.

### FR-8 GUI (App Designer)
- FR-8.1 Controls: source, file chooser, test tone frequency and duration, mode, sample rate,
  frame size, modulator sideband, carrier frequency, noise level, demodulator
  sideband, detuning, signal to play/save, volume, **Process**, **Stop**, **Play**, **Save…**.
- FR-8.2 Spectrum plot (dB vs kHz, 0 … f_s/2), three traces: message, SSB, demodulated.
  The spectrum is computed **incrementally** by a `SpectrumMonitor` (averaged per frame) and refreshed from the frame callback while processing.
- FR-8.3 Waveform plot of a selectable signal.
- FR-8.4 Status line: frames processed / total, mean and max processing load.
- FR-8.5 Setup controls (sample rate, frame size, source, mode) are disabled during a run. Tunable
  controls (sidebands, carrier, detuning, noise) are written into the System objects immediately,
  so they also take effect in the middle of a long run.
- FR-8.6 Stop ends a run early. The GUI stays responsive during a run. Closing the window
  during a run stops it cleanly.

### FR-9 Validation and error handling
- FR-9.1 Parameter validation: `f_c − 3400 Hz > 200 Hz` and `f_c + 3400 Hz < f_s/2 − 200 Hz`.
  The carrier slider's limits follow from the sample rate.
- FR-9.2 Errors use identifiers (`SSB:invalidCarrier`, `SSB:invalidInput`, `SSB:invalidFile`, …).
- FR-9.3 File errors are caught and shown with `uialert`; the app returns to the idle state.

## 6. DSP design (default values)

| Parameter | Value | Note |
|---|---|---|
| Sample rate f_s | 48 kHz (also 44.1 kHz) | |
| Frame size N | 1024 (options 256 … 4096) | 21.3 ms at 48 kHz |
| Message band | 300–3400 Hz | |
| Carrier f_c | 10 kHz default | allowed range from FR-9.1 |
| Input band-pass | 511-tap FIR, Kaiser window β = 8 (difference of two windowed-sinc low-pass filters) | delay 255 samples |
| Hilbert transformer | 511-tap type-III FIR, ideal response `2/(πn)` at odd n, Kaiser β = 8 | delay 255 samples; magnitude error < 1e-4 in 300–3400 Hz |
| Demodulator low-pass | 255-tap windowed sinc, cutoff 4 kHz, Kaiser β = 8 | delay 127 samples |
| Total loopback latency | 255 + 255 + 127 + 255 = **892 samples ≈ 18.6 ms** | |

These values were checked with a numerical reference model (outside MATLAB): sideband suppression ≈ 100 dB for
a 1 kHz tone, about 80 dB at the 300 Hz band edge, and loopback RMS error ≈ 0.2%.
They must be re-checked in MATLAB by the test suite (§10).

## 7. Non-functional requirements

| ID | Requirement |
|---|---|
| NFR-1 | Real-time capable: processing one frame (whole chain, without plotting) takes < 50% of the frame duration. Measured and shown even though this version doesn't play live. |
| NFR-2 | No discontinuities at frame boundaries (follows from NFR-3). |
| NFR-3 | Frame-size invariance: processing the same signal with different frame splits gives outputs that differ by < 1e-9. |
| NFR-4 | Base MATLAB only. |
| NFR-5 | Code style follows the course style guide (`MatlabStyle2 book.pdf`); every public function/class has a help comment. |
| NFR-6 | Deployable as a standalone app with MATLAB Compiler. |
| NFR-7 | **Streaming readiness:** the rules R-1 … R-6 in §9.2 hold. |

## 8. Architecture

```
                 ┌─────────────────────────── SSBEngine ───────────────────────────┐
┌─────────────┐  │  ┌──────────────┐   ┌─────────┐   ┌────────────────┐            │  ┌────────────┐
│ FrameSource │──┼─▶│ SSBModulator │──▶│ channel │──▶│ SSBDemodulator │──┬─────────┼─▶│ FrameSink  │
│ (abstract)  │  │  │ (System obj) │   │ (noise) │   │ (System obj)   │  │         │  │ (abstract) │
└─────────────┘  │  └──────────────┘   └─────────┘   └────────────────┘  │         │  └────────────┘
 ToneSource      │                                                       ▼         │   BufferSink
 FileSource      │   every K frames ──▶ FrameCallback ──▶ SpectrumMonitor, GUI     │  [SpeakerSink]
 [MicSource]     └─────────────────────────────────────────────────────────────────┘
                                         [ ] = Phase 2
```

### 8.1 Components

| File | Kind | Responsibility |
|---|---|---|
| `src/dsp/designKaiserWindow.m` | function | `w = designKaiserWindow(L, beta)` |
| `src/dsp/designHilbertFir.m` | function | `h = designHilbertFir(L, beta)`, odd L |
| `src/dsp/designLowpassFir.m` | function | `h = designLowpassFir(L, fCut, fs, beta)` |
| `src/dsp/designBandpassFir.m` | function | `h = designBandpassFir(L, fLow, fHigh, fs, beta)` |
| `src/dsp/SSBModulator.m` | `matlab.System` | FR-1. Tunable: `CarrierFrequency`, `Sideband`. Nontunable: `SampleRate`, filter lengths, band edges. Method `latency()` |
| `src/dsp/SSBDemodulator.m` | `matlab.System` | FR-2. Tunable: `CarrierFrequency`, `Detune`, `Sideband`. Method `latency()` |
| `src/io/FrameSource.m` | abstract handle class | source interface (§8.2) |
| `src/io/ToneSource.m` | `< FrameSource` | FR-4.1, FR-4.2 |
| `src/io/FileSource.m` | `< FrameSource` | FR-4.3 |
| `src/io/FrameSink.m` | abstract handle class | sink interface (§8.2) |
| `src/io/BufferSink.m` | `< FrameSink` | FR-6: collects frames in a preallocated buffer that grows in chunks |
| `src/engine/SSBEngine.m` | handle class | FR-7: owns modulator, demodulator, channel; `run`, `requestStop`, `processFrame` |
| `src/engine/SpectrumMonitor.m` | handle class | FR-8.2: incremental averaged spectra |
| `src/engine/defaultSettings.m` | function | the settings struct with all default values (shared by GUI, `processFile` and tests) |
| `src/engine/createSource.m`, `createSink.m` | factory functions | build a source/sink from a settings struct, **the single place Phase 2 extends** |
| `src/processFile.m` | function | FR-7.4 command-line wrapper |
| `app/SSBApp.mlapp` | App Designer | FR-8 |
| `tests/` | unit tests | §10 |
| `deploy/` | Compiler project | NFR-6 |

### 8.2 Interfaces (contracts)

All frames are real `double` **column vectors**. The sample rate and frame size are fixed for one run.

```matlab
classdef (Abstract) FrameSource < handle
    properties (SetAccess = protected)      % set by the subclass constructor
        SampleRate      % Hz
        FrameSize       % samples per frame (the last frame of a finite source may be shorter)
    end
    methods (Abstract)
        x = nextFrame(obj)      % N×1 double
        tf = isDone(obj)        % true when a finite source is exhausted; a live source always returns false
        n = totalFrames(obj)    % number of frames, or Inf for a live source (used by the progress bar)
    end
    methods
        function release(obj) %#ok<MANU>  % free resources (files, devices); default: nothing
        end
    end
end
```

```matlab
classdef (Abstract) FrameSink < handle
    methods (Abstract)
        write(obj, frames)      % frames: struct with fields message, ssb, demodulated, monitor (all N×1)
    end
    methods
        function release(obj) %#ok<MANU>
        end
    end
end
```

`SSBEngine`:

| Member | Description |
|---|---|
| `SSBEngine(settings)` | creates modulator, demodulator from a settings struct |
| `Modulator`, `Demodulator` | public, so the GUI can write tunable properties directly |
| `Mode`, `NoiseLevelDb`, `MonitorSignal`, `Volume` | public, tunable at any time |
| `FrameCallback`, `CallbackEvery` | `@(engine, frames, info)` hook, called every K frames; `info` has `frameIndex`, `totalFrames`, `loadPercent` |
| `frames = processFrame(obj, x)` | one frame through the chain according to `Mode`; returns the struct passed to sinks |
| `stats = run(obj, source, sink)` | loop `while ~isDone(source) && ~stopRequested`; at the end (also on error, via `onCleanup`) `release` the System objects, the source and the sink, so every run starts from a fresh setup; return load statistics |
| `requestStop(obj)` | sets the stop flag; safe to call from a callback during `run` |

## 9. Streaming readiness (Phase 2)

### 9.1 What Phase 2 adds — and nothing else

| Work item | Size |
|---|---|
| `src/io/MicSource.m` (`< FrameSource`): wraps `audioDeviceReader`; `isDone` → false; `totalFrames` → Inf; reports overruns | ~30 lines |
| `src/io/SpeakerSink.m` (`< FrameSink`): wraps `audioDeviceWriter`, plays `frames.monitor`; reports underruns | ~25 lines |
| `createSource` / `createSink`: add the cases `'mic'` and `'speaker'`; let `ToneSource`/`FileSource` loop forever when `settings.loop` is true | ~10 lines |
| GUI: add "Microphone" to the source list and a "Live output" checkbox; show under/overrun counts in the status line | ~20 lines |
| `MultiSink` (optional): write to several sinks at once, so you can listen live **and** record | ~20 lines |

**Unchanged in Phase 2:** `SSBModulator`, `SSBDemodulator`, filter design, `SSBEngine`, `SpectrumMonitor`,
all existing tests, Stop/close handling, slider callbacks.
Estimated Phase 2 effort: **8–12 hours**, most of it testing with real audio devices.

### 9.2 Rules this version must follow so §9.1 holds

| Rule | Meaning | Checked by |
|---|---|---|
| R-1 | No operation in the chain may need the whole signal: no `hilbert()`/`fft` over the signal, no normalization by the global maximum, no look-ahead. (One-time preparation inside a source, like normalizing a file at load time, is allowed.) | code review, T7 |
| R-2 | The engine never assumes a finite source: it stops on `isDone` **or** `requestStop`, and handles `totalFrames == Inf`. | T13 |
| R-3 | The output level is set by a fixed gain + limiter, never by looking at the whole result. | code review |
| R-4 | Processing time per frame < 50% of the frame duration (NFR-1). | T11 |
| R-5 | The GUI must never block during a run: all GUI updates and `drawnow limitrate` happen in the `FrameCallback`. | T12 |
| R-6 | Only `createSource`/`createSink` know concrete source/sink classes; the engine and the GUI only use the abstract interfaces. | code review |

## 10. Acceptance tests

| ID | Test | Pass criterion | Covers |
|---|---|---|---|
| T1 | USB, 1 kHz tone, f_c = 10 kHz | power at 11 kHz is ≥ 60 dB above power at 9 kHz | FR-1.1–1.3 |
| T2 | LSB, same | 9 kHz is ≥ 60 dB above 11 kHz | FR-1.2 |
| T3 | Carrier suppression | power at f_c ≥ 60 dB below the wanted tone | FR-1.3 |
| T4 | Loopback USB→USB and LSB→LSB with a multitone in 500–3000 Hz | RMS error vs. message delayed by `latency` ≤ 1% | FR-2.1 |
| T5 | Wrong sideband: USB→LSB and LSB→USB | output power ≥ 60 dB below the matched case | FR-2.2 |
| T6 | Detune +100 Hz, 1 kHz tone | output peak at 900 Hz (USB) / 1100 Hz (LSB) | FR-2.3 |
| T7 | Frame-size invariance (N = 1024 vs. irregular splits) | max difference < 1e-9 | NFR-3, R-1 |
| T8 | Carrier change between frames | no sample-to-sample jump larger than with a constant carrier | FR-1.4 |
| T9 | Invalid carrier (e.g. 23 kHz at 48 kHz) | throws `SSB:invalidCarrier` | FR-9.1 |
| T10 | `reset` clears the state | output after `reset` equals output of a fresh object | FR-1.6 |
| T11 | Timing: mean `processFrame` time at N = 1024 | < 50% of 21.3 ms | NFR-1, R-4 |
| T12 | Manual: long file, move sliders and press Stop during a run, close window during a run | GUI responsive, clean stop | FR-8.5, FR-8.6, R-5 |
| T13 | Engine with an **endless test source** (`isDone` always false, `totalFrames` Inf) and a callback that calls `requestStop` after 50 frames | `run` returns after exactly 50 frames, objects released | FR-7.1, R-2 |
| T14 | `processFile` round trip: modulator-only to file, then demodulator-only from that file (noise off, `FileSource` normalization off, 32-bit float WAV) | equals loopback output within 1e-6 | FR-3, FR-7.4 |

T13 is the test that simulates a live stream: it proves the engine is ready for `MicSource` without having one.

## 11. Milestones

| # | Milestone | Done when |
|---|---|---|
| M1 | Theory and offline prototype (whole signal) | USB/LSB spectra plotted correctly |
| M2 | Filter design functions | frequency responses plotted, Hilbert error < 1e-3 in band |
| M3 | Frame-based modulator and demodulator as System objects | T1–T10 pass |
| M4 | Sources, sinks, engine, `processFile`, `SpectrumMonitor` | T11, T13, T14 pass |
| M5 | App Designer GUI | FR-8 complete, T12 passes |
| M6 | Deployment | standalone app runs on a machine without MATLAB |
| *M7* | *Phase 2: live streaming (§9.1)* | *live loopback with mic and speaker, 0 underruns in steady state* |

## 12. Open questions (ask the instructor)
1. Is App Designer (`.mlapp`) required, or is a programmatic `uifigure` app also accepted?
2. May the Communications Toolbox `ssbmod`/`ssbdemod` be used in tests as a reference?
3. Which deployment target is expected: standalone desktop app or web app?
4. Is file-based processing with a streaming-ready architecture enough, or is live audio expected?

## 13. Possible extensions
- Phase 2 live streaming (§9)
- Transmitted carrier + envelope detector, to show why SSB-SC needs coherent detection
- Carrier recovery (pilot tone + PLL)
- Independent sideband: two files on USB and LSB at the same time
- Weaver modulator as an alternative to the phasing method, compared side by side
