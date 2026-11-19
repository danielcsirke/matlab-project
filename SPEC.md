# Specifikáció — Valós idejű SSB-SC modulátor / demodulátor

**Tárgy:** Mérnöki problémamegoldás MATLAB-ban, BME, 2026 ősz
**Állapot:** v0.3 (2026-11-19). Időkeret: **kb. 10 óra**. A toolboxok használata előny.

---

## 1. Cél

Egy MATLAB-alkalmazás, amely hangot **modulál** és **demodulál** **egyoldalsávos, elnyomott vivős
(SSB-SC)** modulációval, választható **felső (USB)** vagy **alsó (LSB)** oldalsávval,
**valós időben** (streaming): a bemenet hangfájl vagy mikrofon, a kimenet a hangszóró.
App Designerrel készült grafikus felülete van, és MATLAB Compilerrel önálló alkalmazásként telepíthető.

A jelfeldolgozás a **DSP System Toolbox** és az **Audio Toolbox** streamingre tervezett System objectjeire épül.
A saját kód a modulátor és a demodulátor logikája (két kis saját System object), a valós idejű ciklus és a GUI.

## 2. Hatókör

### Kötelező
- SSB-SC moduláció fázismódszerrel (`dsp.AnalyticSignal`), USB vagy LSB
- Koherens demoduláció (szorzás a vivővel + aluláteresztő szűrő), állítható elhangolással
- Streaming: fájl vagy mikrofon → modulátor → demodulátor → hangszóró
- Élő spektrum (`spectrumAnalyzer`)
- App Designer GUI, futás közben állítható paraméterekkel
- Néhány automatikus teszt, telepítés MATLAB Compilerrel

### Opcionális (ha marad idő, +1–2 óra)
- Oldalsáv-szelektív (fázismódszeres) demodulátor, amely elnyomja a rossz oldalsávot
- Additív zaj a modulátor és a demodulátor között, állítható szinttel

### Hatókörön kívül
- Saját szűrőtervezés, vivővisszaállítás (PLL), független oldalsáv, rádiós hardver

## 3. Környezet

| Elem | Követelmény |
|---|---|
| MATLAB | Asztali MATLAB, friss kiadás (R2023b vagy újabb ajánlott). Élő hangeszközök MATLAB Online-ban valószínűleg nem érhetők el. |
| DSP System Toolbox | `dsp.AnalyticSignal`, `dsp.LowpassFilter`, `dsp.AudioFileReader`, `spectrumAnalyzer` |
| Audio Toolbox | `audioDeviceReader` (mikrofon); `audioDeviceWriter` (ez a DSP System Toolboxban is megvan) |
| MATLAB Compiler | telepítés |

A licenceket a munka elején a `ver` paranccsal ellenőrizni kell.

## 4. Funkcionális követelmények

### FR-1 Modulátor (`SSBModulator`, saját System object)
- FR-1.1 Fázismódszer: `s = Re{m_a · e^{jφ}}` USB-nél, illetve `s = Re{conj(m_a) · e^{jφ}}` LSB-nél,
  ahol `m_a = m + j·m̂` az analitikus jel (`dsp.AnalyticSignal`). Kifejtve: `s = m·cos φ − sb·m̂·sin φ`.
- FR-1.2 A bemenetet moduláció előtt egy aluláteresztő szűrő 3,4 kHz-re korlátozza (`dsp.LowpassFilter`).
- FR-1.3 Az oldalsáv (USB/LSB) és a vivőfrekvencia futás közben állítható (hangolható tulajdonság).
- FR-1.4 A vivő fázisakkumulátorral készül, így a vivőfrekvencia változtatásakor nincs fázisugrás, és a kerethatárokon nincs kattogás.

### FR-2 Demodulátor (`SSBDemodulator`, saját System object)
- FR-2.1 Koherens demoduláció: `y = 2 · LPF{ r · cos φ_LO }`, ahol az LO frekvenciája `f_c + Δf`.
- FR-2.2 A `Δf` elhangolás ±500 Hz tartományban, futás közben állítható. (Elhangolt vevőnél
  a hang eltolódik: USB-nél −Δf, LSB-nél +Δf értékkel. Ez a „Donald kacsa” hatás.)
- FR-2.3 *(Opcionális)* Oldalsáv-szelektív mód: `y = I_d − sb · H{Q}`, ahol `Q = LPF{−r · sin φ}`.

### FR-3 Források és kimenet
- FR-3.1 Forrás: WAV-fájl (`dsp.AudioFileReader`, ismételve lejátszva) vagy mikrofon (`audioDeviceReader`).
  Sztereó bemenetet monóvá kell alakítani.
- FR-3.2 Kimenet: hangszóró (`audioDeviceWriter`). A hallgatott jel választható: eredeti üzenet, SSB-jel vagy demodulált jel.
  Hangerő-szabályozás, a kimenet [−1, 1]-re korlátozva.
- FR-3.3 Élő spektrum `spectrumAnalyzer`-rel, három csatornával: üzenet, SSB-jel, demodulált jel.

### FR-4 GUI (App Designer)
- FR-4.1 Kezelőszervek: forrás (fájl/mikrofon) és fájlválasztó, modulátor oldalsávja (USB/LSB),
  vivőfrekvencia-csúszka, elhangolás-csúszka, hallgatott jel, hangerő, **Start**, **Stop**, állapotsor.
- FR-4.2 Az oldalsáv, a vivő, az elhangolás, a hallgatott jel és a hangerő **futás közben** azonnal hat.
- FR-4.3 A forrás futás közben nem változtatható (a hozzá tartozó kezelőszervek le vannak tiltva).
- FR-4.4 A GUI futás közben is reagál; a Stop és az ablak bezárása szabályosan leállít, és felszabadítja a hangeszközöket.
- FR-4.5 Állapotsor: fut / áll, underrunok száma.

### FR-5 Ellenőrzés és hibakezelés
- FR-5.1 A vivőfrekvencia érvényes tartománya: `3,6 kHz < f_c < f_s/2 − 3,6 kHz`. A csúszka határai ebből adódnak;
  a `SSBModulator` érvénytelen értéknél `SSB:invalidCarrier` hibát dob.
- FR-5.2 A 44,1 kHz-nél kisebb mintavételi frekvenciájú fájlokat a program érthető hibaüzenettel elutasítja
  (`SSB:unsupportedSampleRate`), mert ott nincs hely a vivőnek.
- FR-5.3 Eszköz- és fájlhibák esetén `uialert` jelenik meg, és az alkalmazás alapállapotba kerül.

## 5. Alapértékek

| Paraméter | Érték |
|---|---|
| Mintavételi frekvencia | 48 kHz (vagy a fájlé, ha 44,1 kHz) |
| Keretméret | 1024 minta (48 kHz-en 21,3 ms) |
| Vivőfrekvencia | 10 kHz |
| Üzenet-aluláteresztő | áteresztősáv 3,4 kHz, zárósáv 4 kHz |
| `dsp.AnalyticSignal` szűrőfokszám | 500 (az alapértelmezett 100 túl rövid a mély hangokhoz) |
| Demodulátor-aluláteresztő | áteresztősáv 3,4 kHz, zárósáv 4 kHz |

## 6. Architektúra

```
 forrás                 SSBModulator                    SSBDemodulator               kimenet
┌──────────────────┐   ┌────────────────────────────┐   ┌───────────────────────┐   ┌──────────────────┐
│ dsp.AudioFile-   │──▶│ LPF → dsp.AnalyticSignal → │──▶│ × cos(φ_LO) → LPF × 2 │──▶│ audioDeviceWriter│
│ Reader / audio-  │   │ × e^{jφ} → valós rész      │   │                       │   └──────────────────┘
│ DeviceReader     │   └────────────────────────────┘   └───────────────────────┘
└──────────────────┘                 │ üzenet, SSB, demodulált
                                     ▼
                              spectrumAnalyzer
```

| Fájl | Feladat |
|---|---|
| `src/SSBModulator.m` | FR-1, `matlab.System`. Hangolható: `CarrierFrequency`, `Sideband`. Nem hangolható: `SampleRate`, `HilbertOrder`. |
| `src/SSBDemodulator.m` | FR-2, `matlab.System`. Hangolható: `CarrierFrequency`, `Detune` (opcionálisan `Sideband`). |
| `scratch/liveDemo.m` | streaming függvény GUI nélkül (fejlesztéshez és tartalék demóhoz) |
| `app/SSBApp.mlapp` | FR-3, FR-4 |
| `tests/ssbTests.m` | §8 |
| `deploy/` | Compiler-kimenet |

## 7. Nem funkcionális követelmények

| Azonosító | Követelmény |
|---|---|
| NFR-1 | 1024-es keretmérettel állandósult állapotban nincs underrun (nincs kattogás). |
| NFR-2 | A GUI streaming közben is reagál. |
| NFR-3 | A kód követi a tárgy stílusútmutatóját (`MatlabStyle2 book.pdf`); a saját osztályoknak van help-kommentje. |
| NFR-4 | MATLAB Compilerrel önálló alkalmazásként telepíthető. |

## 8. Elfogadási tesztek

| Azonosító | Teszt | Feltétel |
|---|---|---|
| T1 | USB, 1 kHz-es szinusz, f_c = 10 kHz | a kimeneti csúcs 11 kHz-en van, és ≥ 40 dB-lel erősebb, mint 9 kHz-en |
| T2 | LSB, ugyanígy | a csúcs 9 kHz-en, ≥ 40 dB-lel erősebb, mint 11 kHz-en |
| T3 | Loopback (modulátor → demodulátor), 1 kHz-es szinusz | a kimenet csúcsa 1 kHz-en, amplitúdója ±1 dB-en belül egyezik a bemenetével |
| T4 | Érvénytelen vivő (pl. 23 kHz 48 kHz-en) | `SSB:invalidCarrier` hibát dob |
| T5 | Kézi: GUI, mikrofon vagy fájl, csúszkák mozgatása futás közben, Stop, ablakbezárás futás közben | hallhatóan működik, nincs kattogás, szabályos leállás |

## 9. Időterv

| # | Lépés | Óra | Kész, ha |
|---|---|---|---|
| M1 | Elmélet + toolbox-építőelemek kipróbálása szkriptben | 2 | egy fájl USB/LSB spektruma helyes a `spectrumAnalyzer`-ben |
| M2 | `SSBModulator` és `SSBDemodulator` | 2 | a loopback hallhatóan visszaadja a hangot |
| M3 | Streaming szkript (`liveDemo.m`) | 1 | fájl és mikrofon is szól, nincs underrun |
| M4 | App Designer GUI | 3 | FR-4 teljesül |
| M5 | Tesztek | 1 | T1–T4 sikeres |
| M6 | Telepítés + README | 1 | az önálló alkalmazás fut |
| | **Összesen** | **≈ 10** | |

## 10. Nyitott kérdések (az oktatótól megkérdezendő)
1. Kötelező az App Designer (`.mlapp`), vagy programozottan felépített `uifigure`-alkalmazás is elfogadható?
2. Milyen telepítési cél az elvárt: önálló asztali alkalmazás vagy webalkalmazás?

## 11. Lehetséges bővítések
- Oldalsáv-szelektív demodulátor (FR-2.3) és demó: két különböző hang egyszerre az USB-n és az LSB-n
- Állítható zaj a csatornában
- Két gépes demó: az egyik hangszórón sugározza az SSB-jelet, a másik mikrofonnal veszi és demodulálja
- Kimenet rögzítése fájlba (`dsp.AudioFileWriter`)
