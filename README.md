# adam-led-szamolo — Absen Polaris V2 LED-fal Konfigurátor

> **Cél:** Egy önálló, BoM-kompatibilis HTML-konfigurátor az ADAM rendezvénytechnikai cégcsoport (Outline Central Europe / Szimpatech / EXPRESSIO) saját Absen PL3.9W Plus V2 és PL2.5 Plus V2 LED-falaira. A `adam-szinpad-szamolo` (TS3 színpadkalkulátor) testvér-projektje — ugyanaz az architektúra, más domain.

## Mire való

A PV-k (projektvezetők) egy formon megadnak méretet (W × H), pitch-et (P3.9 / P2.5), telepítési típust (földi stack / flown / prism / totem / curved), és a kalkulátor azonnal visszaad:

- **BoM** 7 csoportban (~20-40 sor), Odoo-import-kompatibilis formátumban
- **4 SVG-nézet** (elölnézet, oldalnézet, felülnézet, 3D-iso)
- **Tömeg / térfogat / áramfelvétel** kategóriánként + összesen
- **Áram-, adat-, brightness-, nézési-távolság-check** automatikus figyelmeztetésekkel
- **CSV-export** (UTF-8 BOM) és **Markdown-export** (Odoo-import-kompatibilis)

## Repó-struktúra

```
adam-led-szamolo/
├── README.md                                            ← ez a fájl
├── legacy-html/
│   ├── CHANGELOG.md                                     ← monolit HTML-verzió-történet
│   └── Absen_LED_Konfigurator_v{NN_NN}.html            ← (létrehozandó a v1.0-tól)
└── docs/
    └── 04_KEZIK_Absen_LED_Konfigurator_Spec_v1_7.md    ← implementációs spec
```

## Kapcsolódó projektek

- **`adam-szinpad-szamolo`** — Tomkostage TS3 színpadkalkulátor (testvér-projekt, ugyanaz az architektúra)
- **`adam-knowledge-private`** — ADAM tudástár (`05_OPER_Rentman_Kit_Mintazatok_v2_0.md` 3.3 LED_KONFIG család, 180 kit)

## Implementáció

A teljes implementációs spec a `docs/04_KEZIK_Absen_LED_Konfigurator_Spec_v1_7.md`-ben — egy LLM (Claude / Claude Code) ezt megkapva képes lesz egy működő `.html`-fájlt létrehozni iteratív tisztázás nélkül.

**Bemenet:** a spec.
**Kimenet:** egyetlen önálló `Absen_LED_Konfigurator.html` (~120-160 kB, inline CSS+JS, CDN nélkül, offline működő).

## BoM-példák (spec 11. szekció kivonata)

A 5 példa-konfiguráció (A-E) demonstrálja a kalkulátor működését. Részletek a spec-ben.

### Példa A — 6×3 m P3.9 kültéri földi stack

**Bemenet:** W=6, H=3, pitch=P3.9, installType=ground-stack, brightness=outdoor, cabinetPref=auto-1000-prefer

**Layout:** cols=12, rows=6, total=72 kabinet · 1 536 × 768 pixel (1,18 M) · 10,8 kW max → 4× 16A áramkör

**BoM-kivonat:**
- 36× Absen PL3.9W Plus V2 500×1000 (Rentman 4564/2122)
- 1× MX30 AD KIT (Rentman 4608/2166) + 2× CVT10-S (5348/2666) + 2× extra SFP (5517/2819) + 1× Rack 2U (5553/2855)
- 6× Stack-vas fix 1m (5364/2682) + 11× front leg (5365/2683) + 6× back brace (5441/2759, H≥2m)
- 9× PL3.9 4-fakkos konténer (4597/2155, ceil(36/4))
- 4× CEE 16A elosztó + 4× FI-relé 30 mA + 4× powerCON TRUE1 20m (4591/2149)
- ⚠ Info-sor: **kültéri firmware konfig (4500 nit)** feltöltése a fal összeállása után

### Példa B — 4×2,5 m P2.5 beltéri földi stack

**Layout:** cols=8, rows=5, total=40 kabinet · 1 600 × 1 000 pixel (1,6 M) · 6,4 kW max → szivárgóáram-dominált 4× 16A áramkör (40×0,7 mA / 9 mA = 4)

**BoM-kivonat:**
- 40× Absen PL2.5 Plus V2 500×500 (5474/2784)
- 1× MX30 AD KIT (4608/2166) + lánc
- 4× Stack-vas fix 1m + 7× front leg + 4× back brace
- 7× 6-az-1-ben konténer (5475/2785, ceil(40/6))
- 4× CEE 16A + 4× FI-relé

### Példa C — 5×3 m P3.9 függesztett (flown)

**BoM-kivonat:**
- 30× Absen PL3.9W Plus V2 500×1000
- 5× rigvas 1m (5154/2482) + 7× safety steel
- 1× MX30 AD KIT lánc
- 8× PL3.9 4-fakkos konténer

### Példa D — 4×2,5 m P3.9 hajlított (curved, R=2m, 90°)

**Korlát:** 90°/8 = 11,25° per csatlakozás → ⚠ meghaladja a 10°-os curve-lock-limitet → vagy több szegmensre, vagy SC-vel (prism).

### Példa E — 2×2×1,5 m P3.9 háromoldalú hasáb (prism)

**Layout:** cols=4, rows=3, total_per_side=12, sideCount=3, total=36 SC-kabinet · 512×384 px/oldal

**BoM-kivonat:**
- 36× Absen PL3.9W Plus V2 500×500 **SC** (4566/2124)
- 6× Stack-vas fix 1m (perimeter = 3×2 m = 6 m)
- 6× pillangós AD kit (4626/2184, ceil(36/15) = 3) — *v1.7 új*
- 6× 6-az-1-ben SC konténer (4602/2160)

## Verzió-történet

A `docs/04_KEZIK_Absen_LED_Konfigurator_Spec_v1_7.md` 14. szekciója tartalmazza a teljes spec-evolúciót (v1.0 → v1.7).

**v1.7 (2026-05-12)** — Teljes Rentman LED-fal-szekció bejárása (5× 500-os batch, 2394 aktív tétel), **10 új Absen-rekord** felvéve:
- Rigvas-konténerek (5155 nagy, 5156 kicsi)
- powerCON TRUE1 család 6 tagra bővítve (új: 5699 rövid bekötő, 5700 szakaszközi átkötő)
- Kabinet-összekötő elemek új 2.13 szekció (4623 M8x60, 4624 M8x16, 4625 kabinet-zár, 4626 pillangós AD kit)
- Prizma pre-build kit-ek új 2.14 szekció (5784 kis hasáb, 5931 nagy hasáb)
- **Új 2.15 szekció:** Két párhuzamos LED-rendszer explicit szétválasztása — **Absen+Novastar** (jelen spec scope) vs **LBC+Qiangli+GKGD+Colorlight** (külön spec scope-on kívül)

## Licenc

A repó ADAM cégcsoport belső használatra (public visibility az átláthatóságért és külső LLM-agentek számára való hozzáférhetőségért). Termék-adatok és technikai specifikációk az Absen és NovaStar gyártók hivatalos datasheet-jeiből.
