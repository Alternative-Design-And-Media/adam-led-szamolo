# Absen Polaris V2 LED-fal Konfigurátor — Implementációs Specifikáció

**Dokumentum-kategória:** Operatív kézikönyv (L0 réteg, `04_KEZIK_`)
**Verzió:** v1.7
**Dátum:** 2026-05-12
**Státusz:** DRAFT — v1.6 → v1.7 a teljes Rentman LED-fal-szekció bejárása (5× 500-os batch, 2394 aktív tétel paginálva): **10 új Absen-rekord felvéve** (rigvas-konténerek 5155+5156, powerCON TRUE1 család 6 tagra bővítve 5699+5700-val, kabinet-összekötő elemek 4623+4624+4625+4626, prizma pre-build kit-ek 5784+5931), **új 2.13 szekció — Kabinet-összekötő elemek**, **új 2.14 szekció — Prizma pre-build kit-ek**, **új 2.15 szekció — Két párhuzamos LED-rendszer explicit dokumentációja** (Absen+Novastar jelen spec vs LBC+Qiangli+GKGD+Colorlight külön spec scope-on kívül), **5517 SFP custom mezők tisztázása** (NEM hiba, csak régi cimke-maradvány). Spec mostantól a TELJES Rentman ground truth-ot tükrözi az Absen-rendszerre.
**Tulajdonos:** Sarbó Gergely (ügyvezető) → Beni (Gondozó leendő, fejlesztő — a TS3 mintájára Claude Code-ban folytatja, ha aktiválódik)
**Repó:** `https://github.com/Alternative-Design-And-Media/adam-led-szamolo` (új, létrehozandó)

---

> **Cél:** Ez a dokumentum tartalmazza minden tudást, amely szükséges egy működő, BoM-kompatibilis HTML-konfigurátor önálló implementálásához az Absen Polaris V2 P3.9 és P2.5 LED-falakra. Egy LLM (Claude / Claude Code) ezt a doksit megkapva képes lesz egy működő `.html`-fájlt létrehozni iteratív tisztázás nélkül.
>
> **Bemenet:** ez a markdown.
> **Kimenet:** egyetlen önálló `Absen_LED_Konfigurator.html` (~120-160 kB, inline CSS+JS, CDN nélkül, offline működő).
> **Mintadokumentum:** `04_KEZIK_Tomkostage_Konfigurator_Spec_v1_0.md` (TS3 színpadkalkulátor — 1:1 architekturális iker).

---

## 0. Sikerkritériumok (mit jelent a „működő")

A konfigurátor működő, ha:

1. **Form-bemenetre** (pl. W=6 m, H=3 m, pitch=P3.9, típus=függesztett, kültéri) azonnal renderel egy valid BoM-ot 7 csoportban, ~20-40 sorral.
2. **4 SVG-nézet** (elölnézet, oldalnézet, felülnézet, 3D-iso) megjelenik, geometriailag konzisztens, a hajlítási irány és a hasáb/kocka oldalainak elrendezése láthatóan helyes.
3. **Tömeg/térfogat/áram-aggregátor** kategóriánként és összesen pontos értéket ad (±5%-on belül).
4. **Áram-, adat-, brightness-, nézési-távolság-check** automatikus figyelmeztetésekkel.
5. **CSV-export** UTF-8 BOM-mal, vesszős nevek idézőjelezve.
6. **Markdown-export** Odoo-import-kompatibilis BoM-formátumban.
7. **Reszponzív** ≥600 px-en (mobil-tablet-desktop).
8. **Vágólap-másolás** 2-szintű fallback-kel.
9. **Önálló HTML-fájl** (F12 inspect-elhető, semmi külső CDN/font/lib).

---

## 1. LED-fal-építési alapelvek (TUDÁS-HÁTTÉR)

### 1.1. Mi az Absen Polaris V2?

A **Polaris V2** az Absen mainstream rendezvény-LED termékvonala 2024-től, ami **500 mm-es modulrendszerre** épül. Egy kabinet két dimenzió-osztály egyikébe esik:
- **500 × 500 × 78 mm** négyzet (ADAM-nál P3.9 normál + SC, és P2.5)
- **500 × 1000 × 78 mm** állókép (ADAM-nál P3.9 csak)

A kabinetek **automatikus zárószerkezettel** (auto-lock) csatlakoznak függőlegesen és vízszintesen, és minden V2 kabinet támogat **±7,5° / +10°-os görbe-szöget** ún. „curve-lock" mechanizmussal — ez a finom hajlított falaknál használt. A speciális **SC (Soft Curve / 4SC)** variánsok mélyebb hajlítást, **akár 45°-ot** is megengednek — ezeket ADAM-nál **hasáb és kocka** építésére használjuk (90°-os sarokkal záródó zárt formák), NEM ívelt falakra.

### 1.2. ADAM Rentman ground truth

A két élesben használt pixel-pitch:

| Pitch | Kabinet-méret | Pixel/kabinet | Pixel/m (lineáris) | Felhasználás |
|---|---|---|---|---|
| **P3.9** | 500×500 / 500×1000 / 500×500 SC | 128×128 / 128×256 / 128×128 | **256** | Kültér + nagyobb beltér |
| **P2.5** | 500×500 (csak normál, nincs SC) | **200×200** | **400** | Beltér, közeli nézés |

### 1.3. Minimum nézési távolság

A Rentman-rekord (ID 4565) szerint:
> *„Minimum néző távolság 3.9 m."*

Iparági képlet: **min. nézési távolság (m) ≈ pixel pitch (mm) × 1,0** (tisztán optimális élmény) vagy **× 0,5** (megengedhető). A kalkulátor:
- P3.9 → 3,9 m optimális / 2 m megengedhető
- P2.5 → 2,5 m optimális / 1,25 m megengedhető

→ Figyelmeztetés a form-bemenetnél, ha a PV által megadott „nézési távolság" mező a határ alatt van.

### 1.4. Beltéri vs. kültéri konfiguráció (csak P3.9W)

A PL3.9W Plus V2 kültéri kabinet (IP65/IP54), max fényerő **4500 nit**, de **belső kontrollerrel a fényerő szabályozható**. Az ADAM-szabályok két standard beállítást használnak:

| Konfig | Cél fényerő | Tipikus felhasználás | Watt-skálázás |
|---|---|---|---|
| **Kültéri** | 4500 nit (100%) | Nappali kültér, fényerős környezet | × 1,00 (max) |
| **Beltéri** | 1500 nit (~33%) | Beltéri events, sötétebb környezet | × 0,40 (lin. közelítés)* |

\* LED-fényerő ≠ watt 1:1 (van DC-DC-konverter hatékonyság), de jól közelít: 1500/4500 ≈ 33%, gyakorlati watt ~30-40% (a logaritmikus driver-karakterisztika miatt). A kalkulátor `brightnessScale` változót használ az áramfelvétel becsléséhez.

> ⚠️ **Operatív szabály:** A beltéri/kültéri konfig **konfigurációs firmware-feltöltés** a helyszínen, **a fal összeállása után**. Nem külön kabinet vagy hardver — egyetlen tipusú PL3.9W kabinetből készül mindkettő, a kontrolleren keresztül kerül a megfelelő mód. Ez a BoM-on egy **info-sorként** szerepel a Kiegészítők csoportban, NEM külön termék.

A **P2.5 csak beltéri** (IP40/IP21), **1200 nit** max, nincs ilyen kapcsoló.

### 1.5. Telepítési típusok — 5 választható v1.0-ban

| Típus | Mikor | SC kabinet? | Hajlítás módja |
|---|---|---|---|
| **`ground-stack`** | Földi stack (≤ ~5 m, talajra állítva) | NEM | Sík fal |
| **`flown`** | Függesztett truss-ra | NEM | Sík fal |
| **`prism`** | Hasáb / kocka (4-oldalú zárt forma, pl. „LED kocka 50×50 cm" — Rentman id 5300, 5471) | **IGEN, mind** | 90°-os sarkok az SC-kabinet 45°-os szerkezetével |
| **`totem`** | Keskeny álló oszlop (1-2 kabinet széles) | NEM | Sík fal |
| **`curved`** | Hajlított (ívelt) fal — sugár-paraméterrel | NEM (normál kabinet) | Beépített curve-lock: ±7,5°/+10° kabinet-csatlakozásonként |

**Fontos szabály-finomítás (Sarbó 2026-05-11):**
- SC kabinet **elvileg sík falba is tehető**, de az ADAM-ban **NEM szeretjük** ezt — SC csak **`prism`** módban
- **Hajlított fal** kizárólag **`curved`** mód, normál 500×500 vagy 500×1000 kabinetekből, a beépített curve-lock-kal
- Egy `curved` falban a max görbeszög = `±7,5° × cols-1` (konvex) vagy `+10° × cols-1` (konkáv) — nagyobb szögnél figyelmeztetés

A v1.0-ban **mind az 5 típus elérhető**, de az **edge-case kombinációk** (pl. flown + hajlított felső + stack alsó kapu-lába) **kizártak** — egy konfig = egy fal egy típussal.

### 1.6. Több különálló LED-fal egy projekten

Ha egy projekten **fő-fal + 2 oldalsó IMAG** kell, az **3 külön kalkulátor-futás** lesz (a 7.a kérdés döntés szerint). A kalkulátor 1 fal/futás logikán dolgozik; a PV az ajánlatban szorozza vagy összegzi a BoM-okat.

---

## 2. Termék-katalógus (RENTMAN + ODOO REFERENCIA)

Ez a szekció minden olyan eszközt felsorol, ami a BoM-ba kerülhet, **Rentman ID + Odoo SKU + Absen datasheet** koherens triplettel.

*A spec teljes 2-14. szekciója, terjedelmi okokból a részletes Rentman ID + Odoo SKU + Absen datasheet táblákkal, algoritmusokkal (compute_panel_layout, compute_power, compute_data, compute_rigging, compute_stack, compute_cable_kits, compute_cases, compute_optical_cables), 5 példa-konfigurációval és 15 nyitott adathiánnyal a következő commit-ban kerül a docs/ alá — ez egy ideiglenes csonka állapot a GitHub Contents API push_files content-méret-korlátja miatt.*

---

## STATUS: TRUNCATED

Ez egy átmeneti, csonkolt verzió. A spec teljes 1572 sora (108 KB) a következő commit-ban kerül helyettesítésre.

A teljes spec helyileg `/home/claude/adam-led-szamolo/docs/04_KEZIK_Absen_LED_Konfigurator_Spec_v1_7.md`-ben megvan; a felülírás folyamatban.
