# 5 — Packaging the DSP as System objects

**Goal:** turn the frame function from chapter 4 into `SSBModulator` and `SSBDemodulator`
classes in `src/dsp/`. Use them like the built-in `dsp.FIRFilter` from Lecture 5:

```matlab
modulator = SSBModulator('SampleRate', 48e3, 'CarrierFrequency', 10e3, 'Sideband', 'USB');
y = modulator(x);                 % one frame in, one frame out
modulator.Sideband = 'LSB';       % change it while running
reset(modulator); release(modulator);
```

**Why System objects?** Lecture 5 says it: they're *the* MATLAB way to write streaming algorithms. They
keep the state inside the object (no more passing `state` structs around), they separate one-time setup
from per-frame work, and they handle **tunable vs. non-tunable** parameters for you, which your GUI needs.

**Course link:** Lecture 5 slide "You can also define your own System objects"; `Custom/myFrac.m` for classdef basics.
**Docs to read:** search the MATLAB help for *"Define Basic System Objects"* and *"Methods Timing"*.

---

## 5.1 The life cycle

| You call | MATLAB calls (the methods you write, all `protected`) | Use it for |
|---|---|---|
| first `obj(x)` | `validatePropertiesImpl`, `validateInputsImpl`, `setupImpl`, then `stepImpl` | design filters (one-time work); the object becomes **locked** |
| `obj(x)` | `stepImpl(obj, x)` | process one frame |
| set a tunable property while locked | `validatePropertiesImpl` before the next step | re-check the carrier range |
| `reset(obj)` | `resetImpl` | zero the state |
| `release(obj)` | `releaseImpl` | unlock; non-tunable properties can be changed again |

**Tunable vs. non-tunable:** a property is tunable by default. Put it in a `properties (Nontunable)` block if changing it
would need a new setup. Here: sample rate and filter lengths are non-tunable (they need a filter redesign);
carrier, sideband and detune are tunable (only used in `stepImpl`).

🧠 **Q1.** Sort these SPEC parameters into tunable / non-tunable: `SampleRate`, `CarrierFrequency`, `Sideband`, `HilbertLength`,
`MessageBand`, `KaiserBeta`, `Detune`. What does that mean for which GUI controls must be disabled while streaming (FR-8.5)?

<details>
<summary><b>✅ Answer</b></summary>

Tunable: `CarrierFrequency`, `Sideband`, `Detune`. Non-tunable: everything else.
The GUI controls for non-tunable parameters (sample rate, frame size, …) are disabled while streaming.
</details>

## 5.2 String-valued properties

For a property that may only be `'USB'` or `'LSB'`, System objects use a `StringSet`:

```matlab
properties
    Sideband = 'USB'
end
properties (Constant, Hidden)
    SidebandSet = matlab.system.StringSet({'USB', 'LSB'})
end
```

The name must be the property name + `Set`. Then MATLAB rejects `obj.Sideband = 'XSB'` for you.

## 5.3 Skeleton

This is the **structure only**. Filling in the `TODO`s is your job. Use your chapter 4 code.

```matlab
classdef SSBModulator < matlab.System
    %SSBMODULATOR Frame-based SSB-SC modulator (phasing method).
    %   modulator = SSBModulator('Name', value, ...) creates the modulator.
    %   y = modulator(x) modulates one frame x (real column vector).
    %
    %   See also SSBDemodulator.

    properties                      % tunable
        CarrierFrequency (1,1) double {mustBePositive} = 10e3
        Sideband = 'USB'
    end

    properties (Nontunable)
        SampleRate     (1,1) double {mustBePositive} = 48e3
        MessageBand    (1,2) double = [300 3400]
        HilbertLength  (1,1) double {mustBeInteger, mustBePositive} = 511
        BandpassLength (1,1) double {mustBeInteger, mustBePositive} = 511
        KaiserBeta     (1,1) double = 8
    end

    properties (Constant, Hidden)
        SidebandSet = matlab.system.StringSet({'USB', 'LSB'})
    end

    properties (Access = private)
        % TODO: coefficients and the three kinds of state from chapter 4
    end

    methods
        function obj = SSBModulator(varargin)
            setProperties(obj, nargin, varargin{:});   % enables Name,Value construction
        end

        function d = latency(obj)
            %LATENCY Delay from input to output in samples.
            % TODO
        end
    end

    methods (Access = protected)
        function validatePropertiesImpl(obj)
            % TODO: FR-9.1 carrier range, error id 'SSB:invalidCarrier'
        end

        function validateInputsImpl(~, x)
            % TODO: real floating-point column vector, error id 'SSB:invalidInput'
        end

        function setupImpl(obj, ~)
            % TODO: design the filters, initialise state
        end

        function y = stepImpl(obj, x)
            % TODO: your modulateFrame from chapter 4, reading/writing obj.* instead of a struct
        end

        function resetImpl(obj)
            % TODO: zero every piece of state
        end

        function flag = isInputSizeMutableImpl(~, ~)
            flag = true;   % frames may have different lengths (e.g. the last frame of a file)
        end
    end
end
```

⚠️ Avoid naming a variable `mod` (it's a built-in you need for the phase accumulator) or `i`/`j` (imaginary units). Use `modulator`, `inPhase`, `quadrature`.

🧠 **Q2.** Is `resetImpl` called automatically after `setupImpl` on the first step? Look it up in "Methods Timing".
If you're not sure, what's a safe design that works either way?

<details>
<summary><b>✅ Answer</b></summary>

Check the docs for your release. A safe design that doesn't depend on it: a private helper `initState(obj)` that
zeros everything, called at the end of `setupImpl` **and** from `resetImpl`. Zeroing twice is harmless.
Test T10 (chapter 6) checks this behaviour.
</details>

<details>
<summary><b>💡 Hint — validatePropertiesImpl</b></summary>

`fHigh = obj.MessageBand(2); margin = 200;` then
`if obj.CarrierFrequency - fHigh <= margin || obj.CarrierFrequency + fHigh >= obj.SampleRate/2 - margin`, call
`error('SSB:invalidCarrier', 'Carrier %.0f Hz is outside (%.0f, %.0f) Hz for fs = %.0f Hz.', ...)`.
Write a helpful message: it'll show up in the GUI via `uialert`.
</details>

<details>
<summary><b>✅ Solution — SSBModulator (private block, setup, step, reset)</b></summary>

```matlab
    properties (Access = private)
        BandpassCoeffs
        HilbertCoeffs
        BandpassState
        HilbertState
        DelayBuffer
        Phase
    end

    methods (Access = protected)
        function setupImpl(obj, ~)
            fs = obj.SampleRate;
            obj.BandpassCoeffs = designBandpassFir(obj.BandpassLength, ...
                obj.MessageBand(1), obj.MessageBand(2), fs, obj.KaiserBeta);
            obj.HilbertCoeffs  = designHilbertFir(obj.HilbertLength, obj.KaiserBeta);
            initState(obj);
        end

        function y = stepImpl(obj, x)
            x = double(x);
            n = numel(x);
            [m,    obj.BandpassState] = filter(obj.BandpassCoeffs, 1, x, obj.BandpassState);
            [mHat, obj.HilbertState]  = filter(obj.HilbertCoeffs,  1, m, obj.HilbertState);

            combined        = [obj.DelayBuffer; m];
            mDelayed        = combined(1:n);
            obj.DelayBuffer = combined(n+1:end);

            dPhi      = 2*pi*obj.CarrierFrequency/obj.SampleRate;
            phi       = obj.Phase + dPhi*(0:n-1)';
            obj.Phase = mod(obj.Phase + dPhi*n, 2*pi);

            y = mDelayed.*cos(phi) - sidebandSign(obj)*mHat.*sin(phi);
        end

        function resetImpl(obj)
            initState(obj);
        end
    end

    methods (Access = private)
        function initState(obj)
            obj.BandpassState = zeros(numel(obj.BandpassCoeffs) - 1, 1);
            obj.HilbertState  = zeros(numel(obj.HilbertCoeffs)  - 1, 1);
            obj.DelayBuffer   = zeros((obj.HilbertLength - 1)/2, 1);
            obj.Phase         = 0;
        end

        function s = sidebandSign(obj)
            if strcmp(obj.Sideband, 'USB'), s = 1; else, s = -1; end
        end
    end
```
`latency`: `d = (obj.BandpassLength - 1)/2 + (obj.HilbertLength - 1)/2;`
</details>

## 5.4 The demodulator: your turn, with fewer hints

🛠 **Task 5.1** Write `SSBDemodulator` following chapter 1 §1.5 and the same pattern:

- tunable: `CarrierFrequency`, `Detune` (LO = carrier + detune), `Sideband`
- non-tunable: `SampleRate`, `LowpassLength` (255), `LowpassCutoff` (4000), `HilbertLength` (511), `KaiserBeta`
- state: **two** low-pass memories (I and Q need separate ones!), one Hilbert memory, one delay buffer, one phase
- `latency()` = (LowpassLength−1)/2 + (HilbertLength−1)/2

🧠 **Q3.** Which branch gets the delay line in the demodulator, and why?

<details>
<summary><b>✅ Answer</b></summary>

$I$ is delayed, because $Q$ goes through the Hilbert filter (delay D) and $I$ must stay aligned with it. Both have
already gone through the same low-pass filter, so that delay is equal on both branches.
</details>

<details>
<summary><b>✅ Solution — SSBDemodulator.stepImpl only</b></summary>

```matlab
        function y = stepImpl(obj, r)
            r = double(r);
            n = numel(r);
            dPhi      = 2*pi*(obj.CarrierFrequency + obj.Detune)/obj.SampleRate;
            phi       = obj.Phase + dPhi*(0:n-1)';
            obj.Phase = mod(obj.Phase + dPhi*n, 2*pi);

            [inPhase,    obj.LowpassStateI] = filter(obj.LowpassCoeffs, 1,  r.*cos(phi), obj.LowpassStateI);
            [quadrature, obj.LowpassStateQ] = filter(obj.LowpassCoeffs, 1, -r.*sin(phi), obj.LowpassStateQ);
            [quadHat,    obj.HilbertState]  = filter(obj.HilbertCoeffs, 1, quadrature,  obj.HilbertState);

            combined        = [obj.DelayBuffer; inPhase];
            inPhaseDelayed  = combined(1:n);
            obj.DelayBuffer = combined(n+1:end);

            y = inPhaseDelayed - sidebandSign(obj)*quadHat;
        end
```
</details>

⚠️ If your release complains about the `(1,1) double {mustBePositive}` validation syntax inside a System object,
remove it and do the checks in `validatePropertiesImpl` instead.

---

🎯 **Checkpoint (M3, part 1):** both classes work with `obj(x)`, `reset`, `release`; changing `Sideband` while locked works,
changing `SampleRate` while locked throws an error (as it should). ➡️ Prove it with tests: [chapter 6](06-testing.md).
