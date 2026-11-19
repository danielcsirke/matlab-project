# Study guide — build the SSB modem yourself

This guide takes you from the theory to a deployed app. **It doesn't hand you the program.**
Each chapter explains the math and the MATLAB tools, gives you a task and a checkpoint, and
hides the answers in collapsible blocks:

<details>
<summary><b>💡 Hint</b> (click to open)</summary>

A small push in the right direction. Open it only after you've been stuck for a while.
</details>

<details>
<summary><b>✅ Solution</b></summary>

A reference answer. Open it **after** you've written your own version, to compare.
If you open it first, you lose most of the learning.
</details>

> The hidden solutions are a reference that was **not run in MATLAB** while writing
> this guide (only the algorithm was checked numerically). If something in them is
> wrong, finding the bug is part of the exercise, and your tests (chapter 6) will catch it.

## Two phases

The project follows [`SPEC.md`](../../SPEC.md):

- **Phase 1 (required):** file-based processing with a **streaming-ready** architecture. Everything runs
  frame by frame, exactly as a live stream would, but the input comes from files and test signals,
  and the output is played back after processing. Base MATLAB only.
- **Phase 2 (optional):** live microphone → speaker streaming. If Phase 1 follows the rules in
  SPEC §9.2, this means **adding** a few small classes, with no rewriting.

Parts that belong to Phase 2 are marked like this:

> 🔶 **Optional — Phase 2 (live streaming).** You can skip these in Phase 1.

## How to work through it

1. Read the chapter, including the "why" parts.
2. Answer the 🧠 questions on paper **before** opening the answers.
3. Write the code for the 🛠 task. Use the hint only if you're stuck for more than 20–30 minutes.
4. Pass the 🎯 checkpoint before moving on. Each checkpoint matches a milestone in [`SPEC.md`](../../SPEC.md#11-mérföldkövek).

## Chapters

| # | Chapter | Milestone | Course lecture to review |
|---|---|---|---|
| 1 | [SSB theory](01-theory.md) | — | (signals & systems background) |
| 2 | [Designing the filters](02-filters.md) | M2 | L3 Functions, L4 Graphics |
| 3 | [Offline prototype](03-offline-prototype.md) | M1 | L4 Data import (`audioread`, `sound`) |
| 4 | [Frames and state](04-frames-and-state.md) | — | L5 Streaming (`stream_function.m`), L3 `persistent` |
| 5 | [System objects](05-system-objects.md) | M3 | L5 Objects & Streaming |
| 6 | [Testing](06-testing.md) | M3, M4 | L7 Validation & testing, L8 `timeit` |
| 7 | [Sources, sinks and the engine](07-engine.md) | M4 | L5 Objects (classes), L7 `onCleanup`, L8 Preallocation |
| 8 | [GUI with App Designer](08-gui.md) | M5 | L6 GUI, L3 callbacks, L5 `audioplayer` |
| 9 | [Deployment and polish](09-deploy-and-polish.md) | M6 | Deployment lecture, style guide |
| 10 | 🔶 [Phase 2: live streaming](10-phase2-live-streaming.md) | *M7* | L5 `realtime_audio.m`, `acousticRealTimeEcho.m` |

Chapters 2 and 3 can be done in either order. Chapter 3 is more motivating; chapter 2 is the base for everything after it.

## Folder layout you will build

```
matlab-project/
├── SPEC.md
├── src/
│   ├── dsp/      designKaiserWindow.m, designHilbertFir.m, designLowpassFir.m,
│   │             designBandpassFir.m, SSBModulator.m, SSBDemodulator.m
│   ├── io/       FrameSource.m, ToneSource.m, FileSource.m, FrameSink.m, BufferSink.m
│   │             🔶 MicSource.m, SpeakerSink.m, MultiSink.m   (Phase 2)
│   ├── engine/   SSBEngine.m, SpectrumMonitor.m, createSource.m, createSink.m
│   └── processFile.m
├── app/          SSBApp.mlapp
├── tests/        ssbTests.m, engineTests.m, EndlessSource.m (test helper)
├── scratch/      your experiments (prototype_offline.m, plotFilters.m, …)
└── deploy/
```

Add `src` to the path while working: `addpath(genpath('src'))`, or better, make a
MATLAB Project (`.prj`) that does it for you.

## Toolboxes

Phase 1 needs **base MATLAB only**, so it works in MATLAB Online too. The Signal Processing Toolbox is
useful for cross-checking (`freqz`, `kaiser`, `hilbert`) and better resampling (`resample`).
🔶 Phase 2 needs `audioDeviceWriter`/`audioDeviceReader` (Audio Toolbox / DSP System Toolbox) and desktop MATLAB.
