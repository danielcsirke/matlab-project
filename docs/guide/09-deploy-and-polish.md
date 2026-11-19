# 9 — Deployment and polish

**Goal:** a standalone app that runs on a computer **without MATLAB** (milestone **M6**), plus the finishing touches
that make the project look professional at the exam.

The course's deployment lecture covers the details. Treat this chapter as a checklist to go through alongside it.

---

## 9.1 MATLAB Compiler basics

- In App Designer: **Share → Standalone Desktop App**. Or from code (R2020b+):
  ```matlab
  compiler.build.standaloneApplication('app/SSBApp.mlapp', 'OutputDir', 'deploy/build');
  ```
- The target machine needs the free **MATLAB Runtime** of the **same release** as your MATLAB.
- The compiler finds dependencies by static analysis. Your classes and functions in `src/` are found because the app
  calls them by name. Anything called through a string (`feval('name')`, `str2func`) is **not** found, so add it
  manually or use a `%#function name` pragma.

🧠 **Q1.** Which parts of your project might behave differently once compiled? Make a list before you build.

<details>
<summary><b>✅ Answer (Phase 1)</b></summary>

- **File paths:** don't rely on `pwd` or files next to the `.m` files. Use `uigetfile`/`uiputfile`, or `ctfroot` for bundled files (e.g. a demo WAV).
- **Toolbox functions:** if you use the optional `resample`, check that it's supported by MATLAB Compiler (each doc page has an "Extended Capabilities" section) and that the fallback still works.
- **Anything that writes into the install folder** fails without admin rights, so save to a user-chosen location.
- **Playback:** `audioplayer` uses the default output device. Test that sound works on the target machine.
</details>

> 🔶 **Optional — Phase 2.** Live audio adds more to the list: another machine has another default device, maybe no mic,
> and maybe different supported sample rates (your `try/catch` + `uialert` must handle that). macOS asks for microphone
> permission the first time; on Windows check the privacy settings. Check that `audioDeviceReader`/`audioDeviceWriter` are
> supported by MATLAB Compiler in your release.

🛠 **Task 9.1** Build **early** (as soon as Process works in the GUI), not on the last evening. Test on a different computer.

## 9.2 Code quality checklist

The course style guide is `MatlabStyle2 book.pdf` (in the Teams files).

- [ ] Names: `camelCase` variables and functions, `PascalCase` classes, meaningful names (`carrierFrequency`, not `f2`)
- [ ] Every file has an H1 help line; `help SSBModulator` gives something useful
- [ ] No magic numbers in the code: 3400, 200, 511 … are named properties, constants or `defaultSettings` fields
- [ ] Inputs validated; errors have IDs (`SSB:...`)
- [ ] No warnings in the Code Analyzer (the orange marks on the right edge)
- [ ] `runtests('tests')` passes
- [ ] The streaming-readiness rules (SPEC §9.2) hold. Do a review pass with that table next to your code.
- [ ] README: what it is, requirements (MATLAB release, toolboxes), how to run, how to test, screenshot, and a "Phase 2" section listing what's prepared
- [ ] Commits along the way with meaningful messages, not one big commit at the end

## 9.3 Presentation demo script

A rehearsed 5-minute demo you could use:

1. **Spectrum view, test tone, USB, Process:** show the single line at $f_c + f_m$ with no carrier and no LSB. Switch to **LSB**, Process again, and the line jumps to $f_c - f_m$.
2. **Long speech file:** during Process, **drag the carrier slider** and the SSB trace moves while it's processing. Point out the load figure: at e.g. 5% load the chain runs 20× faster than real time.
3. **Play** the SSB signal ("what the radio signal sounds like"), then **Play** the demodulated one: clean speech back.
4. **Detune +150 Hz**, Process, Play: the Donald Duck effect. Explain Q10 from chapter 1 in one sentence.
5. **Wrong sideband at the receiver:** silence. The phasing demodulator really selects the sideband.
6. **Noise slider:** SNR versus intelligibility.
7. **Architecture slide:** source → engine → sink, the T13 "endless source" test, and "live streaming is two small classes away". This shows you thought about extending it.

> 🔶 **Optional — Phase 2 demo:** the same steps live with a microphone and headphones: drag carrier and detune and hear it immediately.
> Have a **recorded fallback** in case the presentation PC's audio doesn't work.

## 9.4 If you finish early (SPEC §13)

| Extension | New concept you'd learn |
|---|---|
| 🔶 Phase 2: live streaming ([chapter 10](10-phase2-live-streaming.md)) | real-time audio I/O, underruns, latency |
| Transmitted carrier + envelope detector | why SSB-SC needs coherent detection; envelope distortion |
| Pilot tone + PLL carrier recovery | feedback loops, per-sample processing vs. MATLAB speed |
| Independent sideband: two files on USB + LSB | the phasing demodulator separating two programs |
| Weaver modulator | an alternative to the Hilbert filter (two mixers + low-pass filters) |

---

🎯 **Done (M6).** Standalone app runs on another computer, tests pass, README written, demo rehearsed.
Want more? ➡️ 🔶 [Chapter 10](10-phase2-live-streaming.md)
