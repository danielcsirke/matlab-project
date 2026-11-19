# 1 — SSB in one page of math (45 min)

**Goal:** understand the two equations you'll implement and the three demo effects you'll show.
No filter design, just enough theory to explain your app at the exam.

---

## 1.1 Why SSB

Multiplying a message by a carrier, $m(t)\cos\omega_c t$, copies the message spectrum to both sides of $f_c$:
an **upper** and a **lower sideband**, carrying the same information. That's DSB-SC, and it wastes half the bandwidth.
**SSB keeps only one sideband.**

🧠 **Q1.** Put in $m = \cos\omega_m t$. Which frequencies come out?

<details>
<summary><b>✅ Answer</b></summary>

$\tfrac12\cos((\omega_c-\omega_m)t) + \tfrac12\cos((\omega_c+\omega_m)t)$: the LSB at $f_c-f_m$ and the USB at $f_c+f_m$.
</details>

## 1.2 Hilbert transform and analytic signal

The **Hilbert transform** shifts every frequency component by −90°: $\mathcal H\{\cos\} = \sin$, and $\mathcal H\{\sin\} = -\cos$.
Write $\hat m = \mathcal H\{m\}$. The **analytic signal** $m_a = m + j\hat m$ has **only positive frequencies**.

In MATLAB, `dsp.AnalyticSignal` returns exactly $m_a$, frame by frame (chapter 2).

## 1.3 The phasing modulator

Shift the one-sided spectrum up to the carrier and take the real part:

$$s = \operatorname{Re}\{m_a e^{j\omega_c t}\} = m\cos\omega_c t - \hat m\sin\omega_c t \qquad \text{(USB)}$$

🧠 **Q2.** For the LSB you use $\operatorname{conj}(m_a)$, which has only negative frequencies. What changes in the expanded formula?

<details>
<summary><b>✅ Answer</b></summary>

The sign of the second term: $s_\text{LSB} = m\cos\omega_c t + \hat m\sin\omega_c t$. In one formula:
$$\boxed{s = m\cos\omega_c t - sb\cdot\hat m\sin\omega_c t,\qquad sb = +1\ (\text{USB}),\ -1\ (\text{LSB})}$$
**USB ↔ LSB is a single sign.**
</details>

🧠 **Q3.** Check with a tone $m = \cos\omega_m t$ in the USB formula. Where does it land?

<details>
<summary><b>✅ Answer</b></summary>

$\cos\omega_m t\cos\omega_c t - \sin\omega_m t\sin\omega_c t = \cos((\omega_c+\omega_m)t)$: one line at $f_c+f_m$, with no carrier and no LSB.
</details>

## 1.4 The coherent demodulator

Multiply by the carrier again and low-pass filter:

$$y = 2\cdot\operatorname{LPF}\{\,s\cdot\cos\omega_c t\,\}$$

🧠 **Q4.** Show that $y = m$. Why the factor 2? Does it matter whether $s$ is USB or LSB?

<details>
<summary><b>✅ Answer</b></summary>

$m\cos^2\omega_c t = \tfrac m2 + \tfrac m2\cos 2\omega_c t$, and $\hat m\sin\omega_c t\cos\omega_c t = \tfrac{\hat m}2\sin 2\omega_c t$.
The low-pass filter removes everything near $2f_c$, leaving $m/2$, so ×2 gives $m$. This works the same for USB and LSB:
the simple demodulator doesn't need to know the sideband, but it also **can't reject** the other one. (That's what the ⭐ optional
sideband-selective demodulator in chapter 3 adds. MATLAB's `ssbdemod` works the same simple way.)
</details>

## 1.5 Detuning: the "Donald Duck" effect

If the receiver's oscillator runs at $f_c + \Delta f$ instead of $f_c$:

🧠 **Q5.** With $m = \cos\omega_m t$ transmitted as USB, what frequency comes out? And for LSB?

<details>
<summary><b>💡 Hint</b></summary>

The received USB tone is at $f_c + f_m$. Mixing with $f_c+\Delta f$ and low-pass filtering leaves the difference frequency.
</details>

<details>
<summary><b>✅ Answer</b></summary>

USB: $(f_c+f_m) - (f_c+\Delta f) = f_m - \Delta f$. LSB: $(f_c+\Delta f) - (f_c-f_m) = f_m + \Delta f$.
**Every** frequency shifts by the same number of Hz, so the harmonics of a voice stop being harmonic. That's the "Donald Duck"
sound of a mistuned SSB radio, and your detune slider will demonstrate it live.
</details>

## 1.6 Where can the carrier go?

Everything must fit between 0 and $f_s/2$. With a 3.4 kHz message and 200 Hz safety margin on both sides:

🧠 **Q6.** At $f_s$ = 48 kHz, what carrier range works for both sidebands? Why is an 8 kHz file useless?

<details>
<summary><b>✅ Answer</b></summary>

$f_c - 3.4 > 0.2$ and $f_c + 3.4 < 24 - 0.2$ give **3.6 kHz < f_c < 20.4 kHz**. In general: $3.6\text{ kHz} < f_c < f_s/2 - 3.6\text{ kHz}$.
Your modulator will enforce this range, and the GUI's carrier slider will use it as its limits.
At 8 kHz, $f_s/2$ = 4 kHz, so there's no room for a carrier at all. The app must reject such files with a clear message
(anything below 44.1 kHz is a safe rule).
Also: the message must be **low-pass filtered to 3.4 kHz** before modulation. A 48 kHz file can contain sound up to 24 kHz,
and shifted up by $f_c$ that would fold back (alias) into the band.
</details>

---

🎯 **Checkpoint:** you can write the boxed formula and the demodulator from memory, and explain the detune effect in one sentence.
➡️ [Chapter 2](02-building-blocks.md)
