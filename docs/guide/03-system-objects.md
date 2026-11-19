# 3 — Your own System objects (2 h)

**Goal:** wrap the chapter 2 code into two classes, `SSBModulator` and `SSBDemodulator` (in `src/`), that work exactly
like the toolbox objects:

```matlab
modulator = SSBModulator('SampleRate', 48e3, 'CarrierFrequency', 10e3, 'Sideband', 'USB');
s = modulator(x);                 % one frame in, one frame out
modulator.Sideband = 'LSB';       % allowed while running: tunable
release(modulator);
```

Then the streaming loop (chapter 4) and the GUI (chapter 5) stay short and readable. This is milestone **M2**.

**Course link:** Lecture 5, slide "You can also define your own System objects"; `Custom/myFrac.m` for `classdef` basics.
**Docs:** search the MATLAB help for *"Define Basic System Objects"*.

---

## 3.1 What MATLAB calls, and when

| You do | MATLAB calls (methods you write, all `protected`) | Use it for |
|---|---|---|
| first `obj(x)` | `validatePropertiesImpl`, `setupImpl`, then `stepImpl` | create the internal toolbox objects; the object becomes **locked** |
| `obj(x)` | `stepImpl(obj, x)` | process one frame |
| change a tunable property while locked | `validatePropertiesImpl` before the next step | check the carrier range |
| `reset(obj)` | `resetImpl` | clear the state |
| `release(obj)` | `releaseImpl` | release the internal objects; unlocks |

Properties are **tunable** by default. Put the ones that can't change while running in a `properties (Nontunable)` block.

🧠 **Q1.** Which of these must be non-tunable: `SampleRate`, `CarrierFrequency`, `Sideband`, `HilbertOrder`, `Detune`? Why?

<details>
<summary><b>✅ Answer</b></summary>

Non-tunable: `SampleRate` and `HilbertOrder`, because they're used to **create** the internal filters in `setupImpl`.
Tunable: `CarrierFrequency`, `Sideband`, `Detune`, which are only read in `stepImpl`, so they can change between any two frames.
That's exactly what the GUI sliders need.
</details>

🧠 **Q2.** Why create the internal `dsp.LowpassFilter` and `dsp.AnalyticSignal` in `setupImpl`, and not in the constructor?

<details>
<summary><b>✅ Answer</b></summary>

With Name-Value construction, `SampleRate` may be set **after** the constructor's first line, or changed later before the first call.
`setupImpl` runs at the first `obj(x)`, when all properties are final. It also runs again after `release`, so a new sample rate
takes effect the next time.
</details>

## 3.2 A string property with fixed values

```matlab
properties
    Sideband = 'USB'
end
properties (Constant, Hidden)
    SidebandSet = matlab.system.StringSet({'USB', 'LSB'})
end
```

The name must be the property name + `Set`. MATLAB then rejects anything other than `'USB'` or `'LSB'`.

## 3.3 Skeleton for `SSBModulator`

This is the **structure only**. Filling in the `TODO`s is your job, using your code from chapter 2.

```matlab
classdef SSBModulator < matlab.System
    %SSBMODULATOR Streaming SSB-SC modulator (phasing method).
    %   modulator = SSBModulator('Name', value, ...)
    %   s = modulator(x) modulates one frame x (real column vector).
    %
    %   See also SSBDemodulator, dsp.AnalyticSignal.

    properties                                  % tunable
        CarrierFrequency (1,1) double {mustBePositive} = 10e3
        Sideband = 'USB'
    end

    properties (Nontunable)
        SampleRate   (1,1) double {mustBePositive} = 48e3
        HilbertOrder (1,1) double {mustBeInteger, mustBePositive} = 500
    end

    properties (Constant, Hidden)
        SidebandSet = matlab.system.StringSet({'USB', 'LSB'})
    end

    properties (Access = private)
        % TODO: the internal toolbox objects and the carrier phase
    end

    methods
        function obj = SSBModulator(varargin)
            setProperties(obj, nargin, varargin{:});   % enables Name,Value construction
        end
    end

    methods (Access = protected)
        function validatePropertiesImpl(obj)
            % TODO: carrier range 3.6 kHz … fs/2 - 3.6 kHz (chapter 1, Q6), error id 'SSB:invalidCarrier'
        end
        function setupImpl(obj, ~)
            % TODO: create the low-pass filter and dsp.AnalyticSignal, phase = 0
        end
        function s = stepImpl(obj, x)
            % TODO: chapter 2, Task 2.4
        end
        function resetImpl(obj)
            % TODO: reset the internal objects, phase = 0
        end
        function releaseImpl(obj)
            % TODO: release the internal objects
        end
    end
end
```

⚠️ Don't name variables `mod` (you need the built-in `mod` for the phase) or `i`/`j` (imaginary units).

<details>
<summary><b>💡 Hint — validatePropertiesImpl</b></summary>

```matlab
margin = 3600;    % 3400 Hz message + 200 Hz safety
if obj.CarrierFrequency <= margin || obj.CarrierFrequency >= obj.SampleRate/2 - margin
    error('SSB:invalidCarrier', 'Carrier %.0f Hz must be between %.0f and %.0f Hz.', ...
        obj.CarrierFrequency, margin, obj.SampleRate/2 - margin);
end
```
Write a helpful message: the GUI will show it to the user with `uialert`.
</details>

<details>
<summary><b>✅ Solution — private block, setup, step, reset, release</b></summary>

```matlab
    properties (Access = private)
        Lowpass
        Hilbert
        Phase = 0
    end

    methods (Access = protected)
        function setupImpl(obj, ~)
            obj.Lowpass = dsp.LowpassFilter('SampleRate', obj.SampleRate, ...
                'PassbandFrequency', 3400, 'StopbandFrequency', 4000);
            obj.Hilbert = dsp.AnalyticSignal('FilterOrder', obj.HilbertOrder);
            obj.Phase = 0;
        end

        function s = stepImpl(obj, x)
            n = size(x, 1);
            a = obj.Hilbert(obj.Lowpass(x));           % m + j*mHat (aligned, see chapter 2)
            if strcmp(obj.Sideband, 'LSB')
                a = conj(a);
            end
            dPhi = 2*pi*obj.CarrierFrequency/obj.SampleRate;
            phi = obj.Phase + dPhi*(0:n-1)';
            obj.Phase = mod(obj.Phase + dPhi*n, 2*pi);
            s = real(a .* exp(1j*phi));
        end

        function resetImpl(obj)
            if ~isempty(obj.Lowpass)
                reset(obj.Lowpass);
                reset(obj.Hilbert);
            end
            obj.Phase = 0;
        end

        function releaseImpl(obj)
            release(obj.Lowpass);
            release(obj.Hilbert);
        end
    end
```
If your chapter 2 test showed that the real part is **not** aligned, add a `dsp.Delay(obj.HilbertOrder/2)` and build `a` from the delayed real part and `imag(a)`.
</details>

🛠 **Task 3.1** Write `SSBModulator`. Test it in the chapter 2 script: replace the inline modulator with `s = modulator(x)`. The spectrum must look the same.

## 3.4 `SSBDemodulator`: your turn

🛠 **Task 3.2** Write `SSBDemodulator` with the same pattern:
- tunable: `CarrierFrequency`, `Detune` (default 0; the LO runs at `CarrierFrequency + Detune`)
- non-tunable: `SampleRate`
- internals: one `dsp.LowpassFilter` and a phase
- step: `y = 2 * lowpass(r .* cos(phi))` (chapter 1, Q4)
- validation: same carrier range; `abs(Detune) <= 500`

<details>
<summary><b>✅ Solution — stepImpl only</b></summary>

```matlab
        function y = stepImpl(obj, r)
            n = size(r, 1);
            dPhi = 2*pi*(obj.CarrierFrequency + obj.Detune)/obj.SampleRate;
            phi = obj.Phase + dPhi*(0:n-1)';
            obj.Phase = mod(obj.Phase + dPhi*n, 2*pi);
            y = 2*obj.Lowpass(r .* cos(phi));
        end
```
</details>

🛠 **Task 3.3** Loopback in the scratch script: `y = demodulator(modulator(x))`, and show `[x, s, y]` in the spectrum analyzer.
The `y` spectrum should look like the low-passed `x`. Then try `Detune = 100` and see the spectrum shift (down for USB, up for LSB).

## 3.5 ⭐ Optional: sideband-selective demodulator (+1 h)

The simple demodulator can't tell USB from LSB (chapter 1, Q4). This one can. Add a tunable `Sideband` property and a
`'Selective'` mode (e.g. a logical `SelectSideband` property, default `false`):

$$I = \operatorname{LPF}\{r\cos\varphi\},\qquad Q = \operatorname{LPF}\{-r\sin\varphi\},\qquad y = I_d - sb\cdot\mathcal H\{Q\}$$

🧠 **Q3.** For a received USB signal, $I = m/2$ and $Q = \hat m/2$. Show that $y = m$ if the receiver is set to USB and $y = 0$ if it's set to LSB.

<details>
<summary><b>✅ Answer</b></summary>

$\mathcal H\{Q\} = \mathcal H\{\hat m\}/2 = -m/2$. Receiver USB ($sb=+1$): $y = m/2 + m/2 = m$. Receiver LSB ($sb=-1$): $y = m/2 - m/2 = 0$.
Note: no ×2 here, unlike the simple demodulator.
</details>

<details>
<summary><b>💡 Hint — getting $I_d$ and $\mathcal H\{Q\}$ with the right delay</b></summary>

Use **two** `dsp.AnalyticSignal` objects (one for I, one for Q) and two low-pass filters (I and Q need separate state!).
`real(hilbI(I))` is $I$ delayed by the same amount as `imag(hilbQ(Q))` = $\mathcal H\{Q\}$. Computing a Hilbert transform you don't use
wastes a little CPU but saves you from writing a delay line.
</details>

**Demo idea:** add a USB signal carrying one file and an LSB signal carrying another file, then switch the receiver's sideband.
You hear one or the other. MATLAB's `ssbdemod` can't do that.

---

🎯 **Checkpoint (M2):** both classes work with `obj(x)`, `reset` and `release`. Changing `Sideband` or `Detune` while running works,
and changing `SampleRate` while running throws an error (as it should). Loopback gives back the low-passed input. ➡️ [Chapter 4](04-streaming.md)
