# 1 — SSB theory

**Goal:** understand *why* the modulator and demodulator equations look the way they
do. You'll implement exactly these equations later, so it's worth deriving them yourself once.

---

## 1.1 From DSB-SC to SSB

Multiply a message $m(t)$ by a carrier:

$$s_\text{DSB}(t) = m(t)\cos(\omega_c t)$$

This is **double-sideband suppressed-carrier** (DSB-SC). By the modulation theorem its
spectrum is $\tfrac12[M(f-f_c) + M(f+f_c)]$: the message spectrum is copied around
$\pm f_c$. Around $+f_c$ there are **two sidebands**: the upper one (the copy of the
positive frequencies of $m$) and the lower one (the mirror image). Both carry the
same information, so DSB wastes half the bandwidth.

🧠 **Q1.** Let $m(t) = \cos(\omega_m t)$. Write $s_\text{DSB}$ as a sum of cosines. Which term is the USB and which is the LSB?

<details>
<summary><b>✅ Answer</b></summary>

$\cos\omega_m t\cos\omega_c t = \tfrac12\cos((\omega_c-\omega_m)t) + \tfrac12\cos((\omega_c+\omega_m)t)$.
The term at $f_c+f_m$ is the USB and the one at $f_c-f_m$ is the LSB.
**SSB = keep only one of these two.**
</details>

There are two classic ways to remove one sideband:

- **Filter method:** make DSB, then use a very sharp band-pass filter. Hard, because the two sidebands are only $2\times300$ Hz apart.
- **Phasing method:** build the signal so the unwanted sideband **cancels out**. That's what we'll do, and it needs the Hilbert transform.

## 1.2 The Hilbert transform and the analytic signal

The Hilbert transformer is an LTI system with frequency response

$$H(f) = -j\,\operatorname{sgn}(f)$$

Its gain is 1 everywhere. Positive frequencies get a −90° phase shift and negative frequencies get +90°.
We write $\hat m(t) = \mathcal H\{m(t)\}$.

🧠 **Q2.** What is $\mathcal H\{\cos\omega_m t\}$? And $\mathcal H\{\sin\omega_m t\}$?

<details>
<summary><b>✅ Answer</b></summary>

$\mathcal H\{\cos\omega_m t\} = \sin\omega_m t$, and $\mathcal H\{\sin\omega_m t\} = -\cos\omega_m t$.
(Delay each by a quarter period.)
</details>

The **analytic signal** is $m_a(t) = m(t) + j\hat m(t)$.

🧠 **Q3.** Show that $M_a(f) = 2M(f)$ for $f>0$ and $0$ for $f<0$.

<details>
<summary><b>💡 Hint</b></summary>

$M_a(f) = M(f) + j\cdot(-j\operatorname{sgn} f)M(f) = (1+\operatorname{sgn} f)\,M(f)$.
</details>

So the analytic signal has **only positive frequencies**. That's the key idea behind the phasing method.

## 1.3 The phasing modulator

Shift the analytic signal up to the carrier and take the real part:

$$s(t) = \operatorname{Re}\{m_a(t)\,e^{j\omega_c t}\}$$

Shifting a one-sided spectrum up by $f_c$ puts it entirely **above** $f_c$, so that's the USB.

🧠 **Q4.** Expand $\operatorname{Re}\{(m + j\hat m)(\cos\omega_c t + j\sin\omega_c t)\}$.

<details>
<summary><b>✅ Answer</b></summary>

$$s_\text{USB}(t) = m(t)\cos\omega_c t - \hat m(t)\sin\omega_c t$$
</details>

🧠 **Q5.** Which change gives the LSB? (Think: which spectrum has only negative frequencies?)

<details>
<summary><b>✅ Answer</b></summary>

Use the conjugate $m - j\hat m$ (only negative frequencies), which flips the sign:
$$s_\text{LSB}(t) = m(t)\cos\omega_c t + \hat m(t)\sin\omega_c t$$
Both cases in one formula, with $sb = +1$ (USB) or $-1$ (LSB):
$$\boxed{s(t) = m(t)\cos\omega_c t - sb\cdot\hat m(t)\sin\omega_c t}$$
**Switching between USB and LSB is a single sign.**
</details>

🧠 **Q6.** Check it with a tone: put $m=\cos\omega_m t$ into the USB formula. Where does the output lie in frequency?

<details>
<summary><b>✅ Answer</b></summary>

$\cos\omega_m t\cos\omega_c t - \sin\omega_m t\sin\omega_c t = \cos((\omega_c+\omega_m)t)$.
That's a single tone at $f_c+f_m$, with no LSB and no carrier.
</details>

🧠 **Q7.** Suppose the Hilbert branch has a small gain error: $(1+\varepsilon)\hat m$. How big is the leftover unwanted sideband relative to the wanted one? (You'll need this in chapter 2 to pick filter lengths.)

<details>
<summary><b>✅ Answer</b></summary>

Redo Q6 with $(1+\varepsilon)\sin\omega_m t$. The extra term $-\varepsilon\sin\omega_m t\sin\omega_c t$
puts amplitude $\varepsilon/2$ at the unwanted sideband. So the suppression is about
$20\log_{10}(\varepsilon/2)$ dB. A phase error $\delta$ (radians) gives about $\delta/2$ the same way.
**Example:** $\varepsilon = 10^{-3}$ → about −66 dB.
</details>

## 1.4 Coherent (product) demodulation

The receiver multiplies by its own local oscillator (LO) and low-pass filters:

$$I(t) = \operatorname{LPF}\{\,r(t)\cos(\omega_c t + \theta)\,\}$$

🧠 **Q8.** With $r = m\cos\omega_c t - sb\,\hat m\sin\omega_c t$ and $\theta = 0$, show that $I = m/2$.
Does it matter whether $r$ is USB or LSB?

<details>
<summary><b>✅ Answer</b></summary>

$m\cos^2 = \tfrac m2 + \tfrac m2\cos 2\omega_c t$, and $\hat m\sin\cos = \tfrac{\hat m}{2}\sin 2\omega_c t$.
The low-pass filter removes everything at $2\omega_c$, so $I = m/2$ for **both** sidebands.
The simple product detector can't tell USB from LSB, and if both are present it adds them together.
</details>

🧠 **Q9.** Now the LO is off by a constant phase $\theta$. Show that
$I = \tfrac12\left(m\cos\theta + sb\,\hat m\sin\theta\right)$. Why is that acceptable for speech?

<details>
<summary><b>✅ Answer</b></summary>

$\sin\omega_c t\cos(\omega_c t+\theta)$ has the low-frequency part $-\tfrac12\sin\theta$, and the
$\cos$ term gives $\tfrac12\cos\theta$. The result is a mix of $m$ and $\hat m$, which means every
frequency component gets the same phase shift $\theta$. Its amplitude spectrum doesn't change,
and the ear is mostly insensitive to phase. (For data signals this would be a problem.)
</details>

🧠 **Q10.** Now the LO is off in **frequency** by $\Delta f$, so $\theta = 2\pi\Delta f\,t$. Put in a tone
$m=\cos\omega_m t$. Which frequency do you hear for USB, and which for LSB?

<details>
<summary><b>✅ Answer</b></summary>

$I = \tfrac12\cos(\omega_m t - sb\cdot 2\pi\Delta f\,t)$. For USB you hear $f_m - \Delta f$, for LSB $f_m + \Delta f$.
Every component shifts by the same amount in Hz, so harmonics are no longer harmonic.
That's the "Donald Duck" sound of a mistuned SSB radio, and it's a great demo with the detune slider (🔶 live in Phase 2).
</details>

## 1.5 The phasing (sideband-selective) demodulator

To **choose** a sideband at the receiver, take two branches:

$$I = \operatorname{LPF}\{r\cos\omega_c t\},\qquad Q = \operatorname{LPF}\{-r\sin\omega_c t\}$$

and combine them:

$$y = I - sb_\text{rx}\cdot\mathcal H\{Q\}$$

🧠 **Q11.** For $r$ with transmit sideband $sb_\text{tx}$, compute $I$ and $Q$. Then show that $y = m$ if
$sb_\text{rx} = sb_\text{tx}$, and $y = 0$ otherwise.

<details>
<summary><b>💡 Hint</b></summary>

You already have $I = m/2$ from Q8. For $Q$: $-\cos\omega_c t\sin\omega_c t$ has no low-frequency part,
and $\sin^2\omega_c t$ has $\tfrac12$. Then use $\mathcal H\{\hat m\} = -m$.
</details>

<details>
<summary><b>✅ Answer</b></summary>

$Q = sb_\text{tx}\,\hat m/2$, so $\mathcal H\{Q\} = -sb_\text{tx}\,m/2$ and
$$y = \frac m2 + sb_\text{rx}\,sb_\text{tx}\,\frac m2 = \begin{cases} m & sb_\text{rx}=sb_\text{tx}\\ 0 & \text{otherwise}\end{cases}$$
The gain is exactly 1 with no extra scaling, and the other sideband cancels.
**The demodulator has the same structure as the modulator** (delay branch + Hilbert branch + sign), which means less new code to write.
</details>

## 1.6 Discrete time: where the carrier can go

With sample rate $f_s$, everything must fit in $(0, f_s/2)$. The SSB signal covers
$[f_c+300, f_c+3400]$ (USB) or $[f_c-3400, f_c-300]$ (LSB).

🧠 **Q12.** With $f_s = 48$ kHz, a 3400 Hz message, and 200 Hz safety margins, which carrier frequencies work for **both** sidebands?
What about $f_s = 8$ kHz (the course's `speech_dft_8kHz.wav`)?

<details>
<summary><b>✅ Answer</b></summary>

$f_c - 3400 > 200$ and $f_c + 3400 < 24000-200$ give $3.6\text{ kHz} < f_c < 20.4\text{ kHz}$.
At 8 kHz you'd need $f_c > 3.6$ k and $f_c < 0.4$ k, which is impossible. So files must be
**resampled** to the working rate first (FR-4.3).
</details>

🧠 **Q13.** Why does the demodulator need the low-pass filter *before* the Hilbert transformer, and what cutoff should it have?

<details>
<summary><b>✅ Answer</b></summary>

Mixing creates components at about $2f_c$ (around 20 kHz). The derivation in Q11 assumed they're gone.
The cutoff must pass 3400 Hz and block $2f_c - 3400$. 4 kHz is comfortable for $f_c = 10$ kHz.
</details>

---

🎯 **Checkpoint:** you can write the modulator and demodulator equations from memory and explain each sign. Then go to [chapter 2](02-filters.md).
