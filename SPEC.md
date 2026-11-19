# SSB-SC modulátor és demodulátor MATLAB-ban

> Valós idejű MATLAB-alkalmazás, amely hangot **modulál** és **demodulál** egyoldalsávos, elnyomott vivős
> (**SSB-SC**) modulációval. Az oldalsáv szabadon választható: **felső (USB)** vagy **alsó (LSB)**.

---

| | |
|---|---|
| **Bemenet** | audio bemenet vagy fájl |
| **Feldolgozás** | valós idejű (streaming) |
| **Kimenet** | hangszóró vagy fájl |

## Felépítés

| Terület | Megoldás |
|---|---|
| **GUI** | App Designer |
| **Jelfeldolgozás** | DSP System Toolbox és Audio Toolbox |
| **Telepítés** | önálló alkalmazás, Linux rendszerre |

### Saját kódrész

- a modulátor és a demodulátor logikája
- a valós idejű feldolgozási ciklus
- a grafikus felület
