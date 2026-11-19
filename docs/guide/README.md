# Study guide — build the SSB modem yourself (~10 hours)

This guide walks you through building the app described in [`SPEC.md`](../../SPEC.md): a real-time SSB
modulator/demodulator (microphone or file in, speaker or file out) built from **toolbox System objects**, wrapped in two
small classes of your own, with an App Designer GUI, deployed as a standalone Linux app.
**It doesn't hand you the program.** Each chapter explains the idea and the MATLAB tools, gives you a task
and a checkpoint, and hides the answers in collapsible blocks:

<details>
<summary><b>💡 Hint</b> (click to open)</summary>

A small push in the right direction. Open it only after you've been stuck for a while.
</details>

<details>
<summary><b>✅ Solution</b></summary>

A reference answer. Open it **after** you've written your own version, to compare.
</details>

> The hidden solutions were **not run in MATLAB** while writing this guide. If one is wrong, your tests
> (chapter 6) will catch it, and fixing it is part of the exercise.

## How to work through it

1. Read the chapter.
2. Answer the 🧠 questions **before** opening the answers. They're short.
3. Write the code for the 🛠 tasks. Use the hint after ~15 minutes of being stuck.
4. Pass the 🎯 checkpoint before moving on. Each one is a milestone (M1–M6) in the table below.

⏱ The time budget is tight on purpose. If a chapter takes much longer than planned, open the hints sooner.

## Chapters

| # | Chapter | Time | Milestone (done when…) | Course lecture |
|---|---|---|---|---|
| 1 | [SSB in one page of math](01-theory.md) | 45 min | — | — |
| 2 | [Toolbox building blocks](02-building-blocks.md) | 1 h 15 | **M1** the spectrum analyzer shows a correct USB and LSB | L5 Objects & Streaming |
| 3 | [Your own System objects](03-system-objects.md) | 2 h | **M2** loopback gives back the input | L5 (custom System objects), L7 validation |
| 4 | [The streaming loop](04-streaming.md) | 1 h | **M3** file/mic → speaker/file works with no underruns | L5 `realtime_audio.m`, `stream_function.m` |
| 5 | [GUI with App Designer](05-gui.md) | 3 h | **M4** the app streams, sliders act live, Stop is clean | L6 GUI, L3 callbacks |
| 6 | [Tests, deployment, demo](06-tests-deploy.md) | 2 h | **M5** T1–T4 pass · **M6** the Linux build runs | L7 testing, deployment lecture |
| | **Total** | **≈ 10 h** | | |

Optional parts are marked ⭐ **Optional**. Do them only if you're ahead of schedule.

## Before you start (10 minutes)

1. Run `ver` and check that you have **DSP System Toolbox**, **Audio Toolbox** and **MATLAB Compiler**.
2. Use **desktop MATLAB** on your Linux machine. Live audio devices probably won't work in MATLAB Online,
   and the standalone build has to be made on Linux anyway (chapter 6).
3. Get a test WAV at **44.1 or 48 kHz**. The course's `speech_dft_8kHz.wav` is too low (chapter 1 explains why).
   `valami_amerika.wav` from Exercise 5 is a good candidate; check it with `audioinfo`.
4. Have **headphones**. Mic → speaker loopback without them gives acoustic feedback (howling).

## Folder layout

```
matlab-project/
├── SPEC.md
├── src/       SSBModulator.m, SSBDemodulator.m
├── app/       SSBApp.mlapp
├── tests/     ssbTests.m
├── scratch/   experiments, liveDemo.m
└── deploy/
```

Add `src` to the path while working: `addpath('src')`.
