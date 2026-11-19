# Specifikáció — SSB-SC modulátor / demodulátor (streamingre felkészítve)

**Tárgy:** Mérnöki problémamegoldás MATLAB-ban, BME, 2026 ősz
**Állapot:** Tervezet v0.2 (2026-11-19). Az élő hangstreaming kikerült a jelen verzió hatóköréből, de az architektúra fel van rá készítve.

---

## 1. Cél

Egy MATLAB-alkalmazás, amely hangfájlokat **modulál** és **demodulál**
**egyoldalsávos, elnyomott vivős (SSB-SC)** modulációval. A **felső (USB)** vagy
**alsó (LSB)** oldalsáv választható. Az alkalmazásnak grafikus felhasználói felülete
van, és önálló (standalone) alkalmazásként telepíthetőnek kell lennie.

**Kulcsfontosságú tervezési döntés:** bár ez a verzió **fájlokat** dolgoz fel, a teljes
jelfeldolgozás **keretalapú, belső állapottal**, pontosan úgy, ahogy egy élő stream
megkövetelné. A fájlok keretenként haladnak át a láncon, ugyanazokon az interfészeken,
amelyeket később a mikrofon és a hangszóró is használni fog. Az élő streaming későbbi
hozzáadásához **csak új forrás/nyelő osztályokra és kisebb GUI-kiegészítésekre** lehet szükség;
a DSP-kód, a feldolgozó motor (engine) és a tesztek nem változhatnak (lásd §9).

## 2. Hatókör

### A jelen verzió része
- SSB-SC moduláció **fázismódszerrel (Hilbert-transzformátorral)**, USB vagy LSB
- **Oldalsáv-szelektív koherens demoduláció** (fázismódszeres demodulátor), USB vagy LSB
- WAV-fájlok és generált tesztjelek keretalapú feldolgozása
- Az eredmények lejátszása (`audioplayer`), mentése WAV-ba, spektrum- és jelalak-ábrák
- App Designerrel készült GUI, MATLAB Compilerrel telepíthető

### Felkészítve, de nem megvalósítva (2. fázis, lásd §9)
- Élő mikrofonbemenet és hangszórókimenet (valós idejű streaming)

### Hatókörön kívül (lehetséges bővítések, lásd §13)
- Kisugárzott vivő / pilotjel, vivővisszaállítás, PLL
- Független oldalsáv (két különböző üzenet az USB-n és az LSB-n)
- Rádiós hardver (SDR)

## 3. Fogalmak

| Fogalom | Jelentés |
|---|---|
| m(t) | üzenet (alapsávi hang), 300–3400 Hz-re sávkorlátozva |
| m̂(t) | m(t) Hilbert-transzformáltja |
| f_c | vivőfrekvencia (Hz) |
| USB / LSB | felső / alsó oldalsáv; oldalsáv-előjel: `sb = +1` USB-nél, `-1` LSB-nél |
| Δf | a vevő elhangolása: a vevő lokáloszcillátora (LO) f_c + Δf frekvencián fut |
| Keret (frame) | `N` egymást követő mintából álló blokk, valós `double` oszlopvektor |
| Állapot (state) | minden, amit egy blokknak meg kell jegyeznie a keretek között (szűrőmemóriák, késleltetővonalak, oszcillátorfázis) |
| Forrás (source) | kereteket szolgáltató objektum (`FrameSource`) |
| Nyelő (sink) | kereteket fogadó objektum (`FrameSink`) |
| Futás (run) | a motor egy teljes menete a forrástól a nyelőig |

## 4. Környezet

| Elem | Követelmény |
|---|---|
| MATLAB | Friss kiadás (R2023b vagy újabb ajánlott). Asztali MATLAB vagy MATLAB Online, mert ez a verzió nem használ hangeszközöket. |
| Toolboxok (jelen verzió) | **Csak alap MATLAB.** A szűrőket kézzel tervezzük (ablakozásos tervezés, a Kaiser-ablakhoz `besseli`), és `filter`-rel futtatjuk. Lejátszás `audioplayer`-rel, fájlkezelés `audioread`/`audiowrite`-tal. |
| Opcionális | Signal Processing Toolbox (`resample` a jobb minőségű újramintavételezéshez; `freqz`, `kaiser`, `hilbert` ellenőrzéshez), Communications Toolbox (`ssbmod`/`ssbdemod` **kizárólag tesztekben, ellenőrzésre**) |
| 2. fázis (streaming) | `audioDeviceWriter` (Audio Toolbox vagy DSP System Toolbox), `audioDeviceReader` (Audio Toolbox), asztali MATLAB |
| Telepítés | MATLAB Compiler |

## 5. Funkcionális követelmények

### FR-1 Modulátor
- FR-1.1 A fázismódszert valósítja meg:
  `s[n] = m_d[n]·cos(φ[n]) − sb·m̂[n]·sin(φ[n])`, ahol `m_d` az `m` jel, a Hilbert-szűrő
  csoportkésleltetésével megegyező mértékben késleltetve.
- FR-1.2 Az oldalsáv választható: USB vagy LSB. Hangolható (tunable) tulajdonság, tehát keretek között módosítható.
- FR-1.3 A vivő el van nyomva (a kimenetben nincs vivőkomponens).
- FR-1.4 Az `f_c` vivőfrekvencia hangolható, és keretek között **fázisugrás nélkül**
  módosítható (fázisakkumulátoros oszcillátor).
- FR-1.5 Moduláció előtt egy bemeneti sáváteresztő szűrő 300–3400 Hz-re korlátozza az üzenetet.
- FR-1.6 Keretalapú, belső állapottal: a kimenet nem függhet attól, hogyan bontjuk keretekre
  a bemenetet (NFR-3).

### FR-2 Demodulátor
- FR-2.1 Koherens, oldalsáv-szelektív, fázismódszeres demodulátor:
  `I = LPF(r·cos φ)`, `Q = LPF(−r·sin φ)`, `y = I_d − sb·H{Q}`.
- FR-2.2 Az oldalsáv választható (USB/LSB) és hangolható. Az ellentétes oldalsávot elnyomja.
- FR-2.3 Az LO frekvenciája `f_c + Δf`. A `Δf` elhangolás ±500 Hz tartományban hangolható.
  (Elhangolt vevőnél a visszaállított hang eltolódik: USB-nél −Δf, LSB-nél +Δf értékkel.)
- FR-2.4 Keretalapú, belső állapottal (mint FR-1.6).

### FR-3 Üzemmódok
| Üzemmód | Lánc |
|---|---|
| Loopback (alapértelmezett) | forrás → modulátor → csatorna → demodulátor → nyelő |
| Csak modulátor | forrás → modulátor → nyelő (pl. az SSB-jel mentése WAV-ba) |
| Csak demodulátor | forrás (SSB-jel, pl. "Csak modulátor" módban mentett WAV) → demodulátor → nyelő |

### FR-4 Jelforrások (jelen verzió)
- FR-4.1 Tesztjel: szinusz (frekvencia 300–3400 Hz, választható hossz)
- FR-4.2 Kéthangú tesztjel (pl. 700 Hz + 1900 Hz, választható hossz)
- FR-4.3 WAV-fájl: monóvá alakítva, szükség esetén a munka-mintavételi frekvenciára újramintavételezve
  (`resample`, ha elérhető, egyébként `interp1`), és **betöltéskor egyszer** normálva
  (`Normalize` opció, alapértéke igaz; kikapcsolandó olyan fájloknál, amelyek már ismert szintű SSB-jelet tartalmaznak).

### FR-5 Csatorna (csak loopback módban)
- FR-5.1 Opcionális additív fehér Gauss-zaj, szintje dBFS-ben állítható (−80 … −10 dBFS, vagy kikapcsolva).

### FR-6 Kimenet
- FR-6.1 Minden futás egy `BufferSink`-be gyűjti az **üzenetet, az SSB-jelet és a demodulált** jelet.
- FR-6.2 A három jel bármelyike (választhatóan) lejátszható `audioplayer`-rel, hangerő-szabályozással és [−1, 1]-re korlátozással.
  A teljes jel maximumára normálás tilos: a kimeneti szintet kizárólag rögzített erősítés határozza meg (lásd R-3).
- FR-6.3 A három jel bármelyike WAV-fájlba menthető.

### FR-7 Feldolgozó motor (engine)
- FR-7.1 Az `SSBEngine` keretenként végigvezeti a forrás jelét a láncon a nyelőig, amíg
  a forrás `isDone`-t nem jelez, **vagy** leállítást nem kérnek.
- FR-7.2 Minden K-adik keret után meghív egy opcionális `FrameCallback`-et (folyamatjelzés, ábrázolás, `drawnow`).
  Ezt használja a GUI a folyamatjelzőhöz, az ábrák menet közbeni frissítéséhez és a működő Stop gombhoz.
- FR-7.3 Keretenként méri a feldolgozási időt, és jelenti a terhelést (a keretidő %-ában).
  Ez bizonyítja a valós idejű alkalmasságot (NFR-1).
- FR-7.4 Ugyanaz a motor GUI nélkül, parancssorból is futtatható:
  a `processFile(inFile, outFile, options)` egy vékony burkoló (wrapper) körülötte.

### FR-8 GUI (App Designer)
- FR-8.1 Kezelőszervek: forrás, fájlválasztó, tesztjel frekvenciája és hossza, üzemmód, mintavételi frekvencia,
  keretméret, modulátor oldalsávja, vivőfrekvencia, zajszint, demodulátor oldalsávja,
  elhangolás, lejátszandó/mentendő jel, hangerő, **Process**, **Stop**, **Play**, **Save…**.
- FR-8.2 Spektrumábra (dB a kHz függvényében, 0 … f_s/2), három görbével: üzenet, SSB, demodulált.
  A spektrumot egy `SpectrumMonitor` **inkrementálisan** számolja (keretenkénti átlagolással), és feldolgozás közben
  a keret-callbackből frissül.
- FR-8.3 Jelalak-ábra egy választható jelről.
- FR-8.4 Állapotsor: feldolgozott / összes keret, átlagos és maximális feldolgozási terhelés.
- FR-8.5 A beállító kezelőszervek (mintavételi frekvencia, keretméret, forrás, üzemmód) futás közben le vannak tiltva.
  A hangolható kezelőszervek (oldalsávok, vivő, elhangolás, zaj) azonnal beíródnak a System objectekbe,
  így egy hosszú futás közepén is hatnak.
- FR-8.6 A Stop idő előtt befejezi a futást. A GUI futás közben is reagál. Ha futás közben
  bezárják az ablakot, a futás szabályosan leáll.

### FR-9 Ellenőrzés és hibakezelés
- FR-9.1 Paraméterellenőrzés: `f_c − 3400 Hz > 200 Hz` és `f_c + 3400 Hz < f_s/2 − 200 Hz`.
  A vivőfrekvencia-csúszka határai a mintavételi frekvenciából adódnak.
- FR-9.2 A hibák azonosítóval rendelkeznek (`SSB:invalidCarrier`, `SSB:invalidInput`, `SSB:invalidFile`, …).
- FR-9.3 A fájlhibákat el kell kapni és `uialert`-tel megjeleníteni; az alkalmazás visszatér az alapállapotba.

## 6. DSP-tervezés (alapértékek)

| Paraméter | Érték | Megjegyzés |
|---|---|---|
| Mintavételi frekvencia f_s | 48 kHz (44,1 kHz is) | |
| Keretméret N | 1024 (választható: 256 … 4096) | 48 kHz-en 21,3 ms |
| Üzenetsáv | 300–3400 Hz | |
| Vivő f_c | alapértéke 10 kHz | megengedett tartomány: FR-9.1 |
| Bemeneti sáváteresztő | 511 együtthatós FIR, Kaiser-ablak β = 8 (két ablakozott sinc aluláteresztő különbsége) | késleltetés 255 minta |
| Hilbert-transzformátor | 511 együtthatós III. típusú FIR, ideális válasz páratlan n-re `2/(πn)`, Kaiser β = 8 | késleltetés 255 minta; amplitúdóhiba < 1e-4 a 300–3400 Hz sávban |
| Demodulátor aluláteresztő | 255 együtthatós ablakozott sinc, vágási frekvencia 4 kHz, Kaiser β = 8 | késleltetés 127 minta |
| Teljes loopback-késleltetés | 255 + 255 + 127 + 255 = **892 minta ≈ 18,6 ms** | |

Ezeket az értékeket egy (MATLAB-on kívüli) numerikus referenciamodell ellenőrizte: az oldalsáv-elnyomás
1 kHz-es szinusznál ≈ 100 dB, a 300 Hz-es sávszélen kb. 80 dB, a loopback RMS-hibája ≈ 0,2%.
MATLAB-ban a tesztkészletnek (§10) újra ellenőriznie kell őket.

## 7. Nem funkcionális követelmények

| Azonosító | Követelmény |
|---|---|
| NFR-1 | Valós idejű alkalmasság: egy keret feldolgozása (a teljes lánc, ábrázolás nélkül) a keretidő < 50%-a. Akkor is mérjük és kijelezzük, ha ez a verzió nem játszik le élőben. |
| NFR-2 | Nincs szakadás a kerethatárokon (NFR-3-ból következik). |
| NFR-3 | Keretméret-függetlenség: ugyanazt a jelet különböző keretbontással feldolgozva a kimenetek eltérése < 1e-9. |
| NFR-4 | Csak alap MATLAB. |
| NFR-5 | A kódstílus a tárgy stílusútmutatóját követi (`MatlabStyle2 book.pdf`); minden publikus függvénynek/osztálynak van help-kommentje. |
| NFR-6 | MATLAB Compilerrel önálló alkalmazásként telepíthető. |
| NFR-7 | **Streamingre való felkészítettség:** a §9.2 R-1 … R-6 szabályai teljesülnek. |

## 8. Architektúra

```
                 ┌─────────────────────────── SSBEngine ───────────────────────────┐
┌─────────────┐  │  ┌──────────────┐   ┌─────────┐   ┌────────────────┐            │  ┌────────────┐
│ FrameSource │──┼─▶│ SSBModulator │──▶│ csatorna│──▶│ SSBDemodulator │──┬─────────┼─▶│ FrameSink  │
│ (absztrakt) │  │  │ (System obj) │   │  (zaj)  │   │ (System obj)   │  │         │  │ (absztrakt)│
└─────────────┘  │  └──────────────┘   └─────────┘   └────────────────┘  │         │  └────────────┘
 ToneSource      │                                                       ▼         │   BufferSink
 FileSource      │   minden K-adik keret ──▶ FrameCallback ──▶ SpectrumMonitor, GUI │  [SpeakerSink]
 [MicSource]     └─────────────────────────────────────────────────────────────────┘
                                         [ ] = 2. fázis
```

### 8.1 Komponensek

| Fájl | Típus | Feladat |
|---|---|---|
| `src/dsp/designKaiserWindow.m` | függvény | `w = designKaiserWindow(L, beta)` |
| `src/dsp/designHilbertFir.m` | függvény | `h = designHilbertFir(L, beta)`, páratlan L |
| `src/dsp/designLowpassFir.m` | függvény | `h = designLowpassFir(L, fCut, fs, beta)` |
| `src/dsp/designBandpassFir.m` | függvény | `h = designBandpassFir(L, fLow, fHigh, fs, beta)` |
| `src/dsp/SSBModulator.m` | `matlab.System` | FR-1. Hangolható: `CarrierFrequency`, `Sideband`. Nem hangolható: `SampleRate`, szűrőhosszak, sávhatárok. Metódus: `latency()` |
| `src/dsp/SSBDemodulator.m` | `matlab.System` | FR-2. Hangolható: `CarrierFrequency`, `Detune`, `Sideband`. Metódus: `latency()` |
| `src/io/FrameSource.m` | absztrakt handle osztály | forrás-interfész (§8.2) |
| `src/io/ToneSource.m` | `< FrameSource` | FR-4.1, FR-4.2 |
| `src/io/FileSource.m` | `< FrameSource` | FR-4.3 |
| `src/io/FrameSink.m` | absztrakt handle osztály | nyelő-interfész (§8.2) |
| `src/io/BufferSink.m` | `< FrameSink` | FR-6: előre lefoglalt, darabonként bővülő pufferbe gyűjti a kereteket |
| `src/engine/SSBEngine.m` | handle osztály | FR-7: birtokolja a modulátort, a demodulátort és a csatornát; `run`, `requestStop`, `processFrame` |
| `src/engine/SpectrumMonitor.m` | handle osztály | FR-8.2: inkrementálisan átlagolt spektrumok |
| `src/engine/defaultSettings.m` | függvény | a beállítás-struct az összes alapértékkel (közös a GUI, a `processFile` és a tesztek számára) |
| `src/engine/createSource.m`, `createSink.m` | gyártófüggvények | forrás/nyelő létrehozása a beállítás-structból; **ez az egyetlen hely, amelyet a 2. fázis bővít** |
| `src/processFile.m` | függvény | FR-7.4 parancssori burkoló |
| `app/SSBApp.mlapp` | App Designer | FR-8 |
| `tests/` | egységtesztek | §10 |
| `deploy/` | Compiler-projekt | NFR-6 |

### 8.2 Interfészek (szerződések)

Minden keret valós `double` **oszlopvektor**. A mintavételi frekvencia és a keretméret egy futáson belül állandó.

```matlab
classdef (Abstract) FrameSource < handle
    properties (SetAccess = protected)      % az alosztály konstruktora állítja be
        SampleRate      % Hz
        FrameSize       % minta/keret (véges forrás utolsó kerete rövidebb lehet)
    end
    methods (Abstract)
        x = nextFrame(obj)      % N×1 double
        tf = isDone(obj)        % igaz, ha a véges forrás kifogyott; élő forrás mindig hamisat ad
        n = totalFrames(obj)    % keretek száma, élő forrásnál Inf (a folyamatjelző használja)
    end
    methods
        function release(obj) %#ok<MANU>  % erőforrások (fájlok, eszközök) felszabadítása; alapból semmi
        end
    end
end
```

```matlab
classdef (Abstract) FrameSink < handle
    methods (Abstract)
        write(obj, frames)      % frames: struct, mezői: message, ssb, demodulated, monitor (mind N×1)
    end
    methods
        function release(obj) %#ok<MANU>
        end
    end
end
```

`SSBEngine`:

| Tag | Leírás |
|---|---|
| `SSBEngine(settings)` | a beállítás-structból létrehozza a modulátort és a demodulátort |
| `Modulator`, `Demodulator` | publikusak, hogy a GUI közvetlenül írhassa a hangolható tulajdonságaikat |
| `Mode`, `NoiseLevelDb`, `MonitorSignal`, `Volume` | publikusak, bármikor hangolhatók |
| `FrameCallback`, `CallbackEvery` | `@(engine, frames, info)` hook, minden K-adik keret után hívódik; az `info` mezői: `frameIndex`, `totalFrames`, `loadPercent` |
| `frames = processFrame(obj, x)` | egy keret végigvezetése a láncon a `Mode` szerint; a nyelőknek átadott structot adja vissza |
| `stats = run(obj, source, sink)` | ciklus: `while ~isDone(source) && ~stopRequested`; a végén (hiba esetén is, `onCleanup`-pal) `release`-eli a System objecteket, a forrást és a nyelőt, így minden futás friss setuppal indul; terhelési statisztikát ad vissza |
| `requestStop(obj)` | beállítja a leállítási jelzőt; futás közben callbackből is biztonságosan hívható |

## 9. Streamingre való felkészítettség (2. fázis)

### 9.1 Amit a 2. fázis hozzáad, és semmi más

| Feladat | Méret |
|---|---|
| `src/io/MicSource.m` (`< FrameSource`): az `audioDeviceReader` burkolója; `isDone` → false; `totalFrames` → Inf; jelenti az overrunokat | ~30 sor |
| `src/io/SpeakerSink.m` (`< FrameSink`): az `audioDeviceWriter` burkolója, lejátssza a `frames.monitor`-t; jelenti az underrunokat | ~25 sor |
| `createSource` / `createSink`: a `'mic'` és `'speaker'` esetek hozzáadása; a `ToneSource`/`FileSource` végtelenítése, ha `settings.loop` igaz | ~10 sor |
| GUI: "Microphone" a forráslistában és egy "Live output" jelölőnégyzet; under/overrun számlálók az állapotsorban | ~20 sor |
| `MultiSink` (opcionális): több nyelőbe ír egyszerre, így egyszerre lehet élőben hallgatni **és** rögzíteni | ~20 sor |

**A 2. fázisban változatlan marad:** `SSBModulator`, `SSBDemodulator`, a szűrőtervezés, az `SSBEngine`, a `SpectrumMonitor`,
az összes meglévő teszt, a Stop és az ablakbezárás kezelése, a csúszkák callbackjei.
A 2. fázis becsült ráfordítása: **8–12 óra**, ennek nagy része a valódi hangeszközökkel való tesztelés.

### 9.2 Szabályok, amelyeket a jelen verziónak be kell tartania, hogy a §9.1 teljesüljön

| Szabály | Jelentés | Ellenőrzés |
|---|---|---|
| R-1 | A láncban egyetlen művelet sem igényelheti a teljes jelet: nincs `hilbert()`/`fft` a teljes jelen, nincs normálás a globális maximumra, nincs előretekintés. (A forráson belüli egyszeri előkészítés, például a fájl normálása betöltéskor, megengedett.) | kódátnézés, T7 |
| R-2 | A motor sosem feltételez véges forrást: `isDone` **vagy** `requestStop` hatására áll le, és kezeli a `totalFrames == Inf` esetet. | T13 |
| R-3 | A kimeneti szintet rögzített erősítés + korlátozó határozza meg, sosem a teljes eredmény vizsgálata. | kódátnézés |
| R-4 | Egy keret feldolgozási ideje < a keretidő 50%-a (NFR-1). | T11 |
| R-5 | A GUI futás közben sosem blokkolhat: minden GUI-frissítés és a `drawnow limitrate` a `FrameCallback`-ben történik. | T12 |
| R-6 | Csak a `createSource`/`createSink` ismeri a konkrét forrás/nyelő osztályokat; a motor és a GUI csak az absztrakt interfészeket használja. | kódátnézés |

## 10. Elfogadási tesztek

| Azonosító | Teszt | Elfogadási feltétel | Lefedi |
|---|---|---|---|
| T1 | USB, 1 kHz-es szinusz, f_c = 10 kHz | a teljesítmény 11 kHz-en ≥ 60 dB-lel nagyobb, mint 9 kHz-en | FR-1.1–1.3 |
| T2 | LSB, ugyanígy | 9 kHz-en ≥ 60 dB-lel nagyobb, mint 11 kHz-en | FR-1.2 |
| T3 | Vivőelnyomás | a teljesítmény f_c-n ≥ 60 dB-lel kisebb a hasznos komponensnél | FR-1.3 |
| T4 | Loopback USB→USB és LSB→LSB, 500–3000 Hz közötti többhangú jellel | RMS-hiba a `latency`-vel késleltetett üzenethez képest ≤ 1% | FR-2.1 |
| T5 | Rossz oldalsáv: USB→LSB és LSB→USB | a kimeneti teljesítmény ≥ 60 dB-lel kisebb, mint az illeszkedő esetben | FR-2.2 |
| T6 | Elhangolás +100 Hz, 1 kHz-es szinusz | kimeneti csúcs 900 Hz-en (USB) / 1100 Hz-en (LSB) | FR-2.3 |
| T7 | Keretméret-függetlenség (N = 1024 kontra szabálytalan bontás) | maximális eltérés < 1e-9 | NFR-3, R-1 |
| T8 | Vivőváltás keretek között | nincs nagyobb mintáról mintára ugrás, mint állandó vivőnél | FR-1.4 |
| T9 | Érvénytelen vivő (pl. 23 kHz 48 kHz-en) | `SSB:invalidCarrier` hibát dob | FR-9.1 |
| T10 | A `reset` törli az állapotot | a `reset` utáni kimenet megegyezik egy friss objektum kimenetével | FR-1.6 |
| T11 | Időzítés: a `processFrame` átlagos ideje N = 1024 esetén | < a 21,3 ms 50%-a | NFR-1, R-4 |
| T12 | Kézi teszt: hosszú fájl; futás közben csúszkák mozgatása és Stop; futás közben ablakbezárás | a GUI reagál, szabályos leállás | FR-8.5, FR-8.6, R-5 |
| T13 | A motor egy **végtelen tesztforrással** (`isDone` mindig hamis, `totalFrames` Inf) és egy callbackkel, amely 50 keret után `requestStop`-ot hív | a `run` pontosan 50 keret után visszatér, az objektumok fel vannak szabadítva | FR-7.1, R-2 |
| T14 | `processFile` oda-vissza: csak modulátor fájlba, majd csak demodulátor abból a fájlból (zaj ki, `FileSource`-normálás ki, 32 bites lebegőpontos WAV) | 1e-6-on belül megegyezik a loopback kimenettel | FR-3, FR-7.4 |

A T13 az a teszt, amely élő streamet szimulál: bizonyítja, hogy a motor készen áll a `MicSource`-ra, anélkül hogy az létezne.

## 11. Mérföldkövek

| # | Mérföldkő | Kész, ha |
|---|---|---|
| M1 | Elmélet és offline prototípus (teljes jel egyben) | az USB/LSB spektrumok helyesen kirajzolva |
| M2 | Szűrőtervező függvények | a frekvenciaválaszok kirajzolva, a Hilbert-hiba < 1e-3 a sávban |
| M3 | Keretalapú modulátor és demodulátor System objectként | T1–T10 sikeres |
| M4 | Források, nyelők, motor, `processFile`, `SpectrumMonitor` | T11, T13, T14 sikeres |
| M5 | App Designer GUI | FR-8 teljes, T12 sikeres |
| M6 | Telepítés | az önálló alkalmazás MATLAB nélküli gépen is fut |
| *M7* | *2. fázis: élő streaming (§9.1)* | *élő loopback mikrofonnal és hangszóróval, állandósult állapotban 0 underrun* |

## 12. Nyitott kérdések (az oktatótól megkérdezendő)
1. Kötelező az App Designer (`.mlapp`), vagy programozottan felépített `uifigure`-alkalmazás is elfogadható?
2. Használható-e a Communications Toolbox `ssbmod`/`ssbdemod` függvénye referenciaként a tesztekben?
3. Milyen telepítési cél az elvárt: önálló asztali alkalmazás vagy webalkalmazás?
4. Elegendő-e a fájlalapú feldolgozás streamingre felkészített architektúrával, vagy élő hang az elvárás?

## 13. Lehetséges bővítések
- 2. fázis: élő streaming (§9)
- Kisugárzott vivő + burkolódetektor, annak bemutatására, miért igényel az SSB-SC koherens demodulációt
- Vivővisszaállítás (pilotjel + PLL)
- Független oldalsáv: két fájl egyszerre az USB-n és az LSB-n
- Weaver-modulátor a fázismódszer alternatívájaként, egymás melletti összehasonlítással
