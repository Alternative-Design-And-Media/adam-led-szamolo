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

### 2.1. LED-kabinetek

| Termék | Rentman ID | Rentman code | Odoo ID | Odoo SKU | Méret (mm) | Súly (kg) | Pixel/kabinet | Max W | Avg W |
|---|---|---|---|---|---|---|---|---|---|
| **Absen PL3.9W Plus V2 500×500** | 4565 | 2123 | *(nincs még)* | *(létrehozandó: `ABSEN-PL3.9W-PLUSV2-500x500-STANDARD`)* | 500×500×78 | 8,0 | 128×128 | 150 | 50 |
| **Absen PL3.9W Plus V2 500×500 SC** | 4566 | 2124 | *(nincs még)* | *(létrehozandó: `ABSEN-PL3.9W-PLUSV2-500x500-CORNER`)* | 500×500×78 | 8,0 | 128×128 | 150 | 50 |
| **Absen PL3.9W Plus V2 500×1000** | 4564 | 2122 | 920 | `ABSEN-DISPLAY-P3.9-500x1000-OUTDOOR-7680HZ` | 500×1000×78 | 13,0 | 128×256 | 300 | 100 |
| **Absen PL2.5 Plus V2 500×500** | 5474 | 2784 | 1596 | `ABSEN-PL2.5-PLUS-V2-500x500-STANDARD` | 500×500×78 | 7,8 | 200×200 | 160 | 54 |

**Datasheet-ek (Absen hivatalos):**
- **PL3.9W Plus V2** (kültéri, IP65/IP54): max fényerő 4500 nit @ 100% / 1500 nit @ ~33%; max W/m² 600, avg 200; refresh 7680 Hz; op temp -20 → +50°C; built-in curve-lock ±7,5°/+10°
- **PL2.5 Plus V2** (beltéri, IP40/IP21): max fényerő 1200 nit; max W/m² 640, avg 215; refresh 3840 Hz; op temp -10 → +40°C; built-in curve-lock ±7,5°/+10°

> ⚠️ **Datasheet-Rentman diszkrepancia (W):** A PL3.9W Plus V2 500×500 Rentman `power=180` (Rentman ID 4565), míg a datasheet 150 W max. A kalkulátor a **datasheet-értékeket** használja (ez az Absen hivatalos hatékonyság-spec), a Rentman `power` mező egy ADAM-belső konzervatív becslés. Watt-számolásnál **csak datasheet**.

> ⚠️ **Adathiány:** A PL2.5 Plus V2 (Rentman 5474) `power=0`, `current=0`, `volume=0`. **Pótolandó adatok:**
> - Volume: ~0,02 m³ (a PL3.9 500×500 alapján, ami fizikailag azonos)
> - Max power: 160 W (datasheet 640 W/m² × 0,25 m²)
> - Current: 0,75 A @ 230 V (P=UI, 160/230 ≈ 0,70 A; +10% biztonsági)
> - **L1 napló-bejegyzés:** `06_NAPLO_2026-W19.md`-ben felvenni mint Rentman adatminőség-feladat, Beninek továbbítható

### 2.2. SC szemantika és hajlítás

| Variáns | Maximális szög | Mire való ADAM-nál |
|---|---|---|
| **Standard kabinet (nem SC)** | ±7,5° konvex / +10° konkáv (per kabinet-csatlakozás) | Sík falra és **enyhe ívelt fal** (`curved` mód) |
| **SC (Soft Curve / 4SC) kabinet** | **±45°** (per kabinet-csatlakozás) | **Hasáb / kocka** (`prism` mód) — 4-oldalú zárt formák |

**ADAM operatív szabály (Sarbó 2026-05-11):**
- SC kabinet elvileg sík falba is tehető, de **NEM szokás**
- Hajlított falat a standard 500×500 és 500×1000-ből építünk a beépített curve-lock-kal
- SC kizárólag hasáb / kocka építésére (90°-os sarkok 2 SC-kabinet 45°-os hajlítással)

### 2.3. Processzor (NovaStar COEX-széria — két lépcső pixel-szám alapján)

**Önálló processzor-tételek Rentmanben:**

| Termék | Rentman ID | Rentman code | Súly | Power | Pixel-kapacitás | Portok | Méret |
|---|---|---|---|---|---|---|---|
| **Novastar KU20** | 4605 | 2163 | 2,1 kg | 25 W, 0,1 A | 3,9 M pixel | 6× GigaEth + 1× 10G optika | Half-1U (29×25×5 cm) |
| **Novastar MX30** | 4606 | 2164 | 7,2 kg | 55 W, 0,02 A | 6,5 M pixel | 10× GigaEth + 2× 10G optika | 1U (48×47×9,4 cm), HDR10, 4K-input |

**AD KIT (önálló processzor flight case-ben):**

| Termék | Rentman ID | Rentman code | Súly | Tartalma | Megjegyzés |
|---|---|---|---|---|---|
| **Novastar KU20 \| 6E + 1S \| AD KIT** | 4607 | 2165 | 8,82 kg | KU20 + 1× SFP modul + flight case (empty_weight 6,3 kg) | A standalone KU20 hordtáskában |
| **Novastar MX30 \| 10E + 2S \| AD KIT** | 4608 | 2166 | 20,67 kg | MX30 + 2× SFP modul + flight case | A standalone MX30 hordtáskában |

**AD konfiguráció (Virtual Package — `tags: "elveszett"` állapotban):**

| Termék | Rentman ID | Rentman code | Tartalma | Státusz |
|---|---|---|---|---|
| Novastar KU20 processzor + optika \| 6 port \| AD kit | 5360 | 2678 | KU20 + CVT10 + 6 port | 🚫 `tags: elveszett` — hibás import (NEM fizikai elvesztés). Beni átdolgozza fizikai AD-konfiguráció-rekorddá |
| Novastar MX30 processzor + optika \| 10 port \| AD kit | 5350 | 2668 | MX30 + 1× CVT10 + 10 port | 🚫 `tags: elveszett` — ugyanúgy |
| Novastar MX30 processzor + optika \| 10 port + 10 port redundáns (20 port) \| AD konfiguráció | 5555 | 2857 | MX30 + 2× CVT10 + 20 port | (státusz: aktív, de még Virtual Package) |

> **Sarbó-szabály v1.5+v1.6:** A Virtual Package-eket **NEM** használjuk default-ban (a Rentman rendszer átdolgozása alatt áll — virtuálisban nem lehet virtuális kit, csak fizikai). A v1.6-os default a **fizikai AD KIT (4608 — MX30 10E+2S) + külön CVT10-S + Rack 2U** kombináció.

**Processzor-választás v1.6 (Sarbó-szabály 5+6 — fizikai komponensek, redundáns 50Hz/8bit):**

| Fal pixel-szám | Default BoM | Megjegyzés |
|---|---|---|
| ≤ 6,5 M pixel | **1× MX30 AD KIT (4608)** + **2× CVT10-S (5348)** + **2× extra SFP (5517)** + **1× Rack 2U (5553)** | MX30 AD KIT-ben már 2 SFP van, redundánsan; +2 SFP a 2 CVT10-hez |
| 6,5 M – 13 M pixel | **2× MX30 AD KIT (4608)** + **4× CVT10-S (5348)** + **4× extra SFP (5517)** + **2× Rack 2U (5553)** | 2 processzor kaszkádolva |
| > 13 M pixel | 🔴 Beni-konzultáció | kívül a v1.0 scope-ján |

> **Kit-tartalom megjegyzés:**
> - **MX30 AD KIT (4608) tartalma:** 1× MX30 (4606) + 2× 10G SFP modul (5517) + hordtáska. `empty_weight: 1 kg`, total 20,67 kg → kb 7,2 kg MX30 + 0,1 kg SFP + 13 kg hordtáska. A táska beleszámolódik a szállítási súlyba.
> - **KU20 AD KIT (4607) tartalma:** 1× KU20 + 1× SFP + hordtáska. NEM default-ban használt (csak 1 fiber out, nem alkalmas redundáns konfigra).
> - **CVT10 KIT (5349)** alternatíva: ha 3 CVT10 kell egyben (multi-feed setup, > 6,5 M pixel), 1× CVT10 KIT olcsóbb mint 3× standalone CVT10-S. 14,39 kg.

A kalkulátor PV-checkbox-szal kínálhat alternatívákat: "CVT10 KIT (3 db, 1 táskában)" vs "3× standalone CVT10".

### 2.4. Optikai konverter (CVT10) + SFP modul + Rack 2U

**Optikai konverter Rentmanben (CVT10-S — single-mode 10 Gbit/s):**

| Termék | Rentman ID | Rentman code | Súly | Méret | Megjegyzés |
|---|---|---|---|---|---|
| **Novastar CVT10-S** | 5348 | 2666 | 2,5 kg | 25×10×34 cm | Önálló konverter, half-1U, OPT1 master / OPT2 backup |
| **NOVA CVT10 \| optika ethernet átalakító \| KIT** | 5349 | 2667 | 14,39 kg | 28×12×33 cm | Hordtáskás konténer (3 db CVT10 + SFP-k + kábelek) |

A CVT10-S egy 10G optikai bemenetből **akár 10 db Gigabit Ethernet kimenetet** ad — egy CVT10 fal-oldalon kifejti a fal-link-eket. Egy processzor 1 optikai out-ja → 1 db CVT10.

**SFP modul (a 10G optikai port hardver-modulja):**

| Termék | Rentman ID | Rentman code | Súly | Megjegyzés |
|---|---|---|---|---|
| **Novastar 10G SFP module (SM)** | 5517 | 2819 | 0,05 kg | Single-mode 10G SFP+, hot-swap. Egy CVT10-be 2 db kell (master + backup), egy processzorba 1 db (KU20) vagy 2 db (MX30) |

**SFP-darabszám szabály:**
- Processzor oldalon: `procModel === 'KU20' ? 1 : 2` (KU20 = 1× optikai, MX30 = 2× optikai)
- CVT10 oldalon: `2 × cvtCount` (master + backup minden CVT10-en)
- Összes: `procSFP + 2 × cvtCount`

**Rack 2U Double Door (processzor + CVT10 együtt egy rack-konténerbe):**

| Termék | Rentman ID | Rentman code | Súly | Méret | Megjegyzés |
|---|---|---|---|---|---|
| **Novastar Rack 2U Double Door Rack** | 5553 | 2855 | 8 kg | 63×16×55 cm | Dupla ajtós, 2U-s rack — egy KU20+CVT10 vagy 1× MX30+CVT10 fér el együtt egy rackbe. PV-választás: `1× per processor + CVT10 pár` |

> **AD kit redundancia (v1.6 szabály):** A Virtual Package AD kit-eket (`2678/2668/2857`) **NEM** használjuk default-ban — `tags: "elveszett"` (hibás import, Beni átdolgozza). A v1.6 a **fizikai komponens-listát** ajánlja (4608 MX30 AD KIT + standalone CVT10 + Rack 2U). A 4607 (KU20 AD KIT) és 4608 (MX30 AD KIT) `type: case` fizikai rekordok — ezek a stand-alone processzor flight case-be foglalva (KU20 esetén + 1 SFP, MX30 esetén + 2 SFP).

### 2.5. Rigging — flown módra (Absen-specifikus rigvas család)

**Rigvas-tételek:**

| Termék | Rentman ID | Rentman code | Súly (kg) | Méret (mm) | Bérleti díj | Funkció |
|---|---|---|---|---|---|---|
| **Absen 0,5 m rigvas (LED függesztő)** | 4622 | 2180 | 3,8 | 500×100×100 | 1 650 Ft | Keskeny falakhoz, 0,5 m moduláris hossz |
| **Absen 1,0 m rigvas (LED függesztő)** | 5154 | 2482 | 7,3 | 1000×100×100 | 2 800 Ft | Standard 1 m hosszú rigvas |
| **Absen 1,0 rigvas \| AD kit (konténer 10×)** | 5157 | 2485 | 113 (10 db / konténer) | 700×1140×530 | 30 900 Ft | 10× 1 m rigvas hordtáskában |

**Rigvas-konténerek (külön rekord, a fenti AD kit-ek belső konténere):**

| Termék | Rentman ID | Code | Súly (kg) | Méret (mm) | Megjegyzés |
|---|---|---|---|---|---|
| **Absen rigvas konténer \| nagy** | **5155** | **2483** | 40 | 700×1140×530 | A `5157` AD kit (10× 1m rigvas) belső konténere. Önállóan is rendelhető extra-tárolásra. |
| **Absen rigvas konténer \| kicsi** | **5156** | **2484** | 42 | 620×1140×560 | A 0,5m rigvasokhoz. Stand-alone tárolódoboz. |

**Rigvas-darabszám szabály (default v1.0):**
- A fal teljes szélességét le kell fedni rigvassal: greedy `[1,0 m → 0,5 m]` sorrendben
- `rigvas_1m = floor(W_eff)`
- `rigvas_05m = ceil((W_eff - rigvas_1m) / 0,5)`
- Példa: W = 6,0 m → 6× 1,0 m rigvas + 0× 0,5 m
- Példa: W = 3,5 m → 3× 1,0 m rigvas + 1× 0,5 m rigvas
- Példa: W = 4,5 m → 4× 1,0 m rigvas + 1× 0,5 m rigvas

**AD kit (konténer) választás:**
- Ha `rigvas_1m ≥ 8` → 1× kit konténer (5157, gazdaságos szállítás — a 5155 nagy konténer benne van)
- Egyébként → szabad rigvas-tételek + opcionálisan 5156 (kicsi konténer) ha 4+ db 0,5m rigvas van

> ⚠️ **Korábbi téves spec (v1.0 → v1.1 javítás):** A v1.0 az „Absen PL single/double/90° hanging bar" Odoo-SKU-kat írta — ezek az Odoo-katalógusban vannak (id 921-923), de a **Rentmanben NEM** ezek a használt rigvasok. A használatban lévő tételek a 2180/2482/2485 kódokon vannak.

### 2.6. Stack — földi szerkezet (Absen-specifikus stack család)

| Termék | Rentman ID | Rentman code | Súly (kg) | Méret (mm) | Bérleti díj | Funkció |
|---|---|---|---|---|---|---|
| **Absen alsó sztekkvas fix 1 m** | 5364 | 2682 | 11,4 | 1000×550×50 | 2 000 Ft | Alapvonal-gerenda **egyenes** szakaszra, 1 m / db |
| **Absen alsó sztekkvas fokolható 0,5 m** | **5473** | **2783** | 8,6 | 500×580×50 | 1 800 Ft | **Szöges** ("fokolható") alapvonal-gerenda — sarkokhoz / fractional width-hez |
| **Absen alsó sztekkvas első kitámasztó** | 5365 | 2683 | 0,5 | 205×40×50 | 110 Ft | Első (front) támasztó láb — minden kabinet-csatlakozási ponthoz előre |
| **Absen kabinet kitámasztó** | 5441 | 2759 | 5,0 | (nincs adat) | 850 Ft | Hátsó merevítő — minden **második** csatlakozási ponthoz |

**Sztekkvas-szabály (Sarbó-finomítás, v1.3 megerősítve v1.4-ben):**
- **1 m sztekkvas (`5364/2682`)** csak **egyenes** szakaszra (merev test, nem hajlik)
- **0,5 m sztekkvas (`5473/2783`)** **fokolható** = szögbe rakható — sarkokhoz (prism mód) és fractional width-hez (W mod 1m ≠ 0)
- A kettő **kombinálható** — egyenes szakaszra 1 m, sarokba 0,5 m

**Front-leg és back-brace szabály (Sarbó 2026-05-11 v1.5):**

- **N kabinet** egy sorban → **N−1 belső csatlakozási pont** (kabinet-kabinet illesztések, **a két szélső véget NEM számolva** — ott a sztekkvas-rúd vége tartja)
- **Első kitámasztó (`5365/2683`) — front leg:** **MINDEN belső csatlakozási ponthoz** előre, **N−1 db**
- **Hátsó kitámasztó (`5441/2759`) — back brace:** **H ≥ 2,0 m**-tól, **minden második** belső ponthoz, `ceil((N−1) / 2)` db

> **FONTOS — Sarbó 2026-05-11 v1.5 finomítás:**
> - A back brace `H ≥ 2,0 m`-tól megy be (NEM 1,5 m). Az alatti magasságon a kabinet kitámasztó **fizikailag nem fér el** — nem opcionális hiány, hanem építési korlát.
> - A front leg **N−1** (csak belső kabinet-kabinet illesztések, NEM `N+1`). A v1.3-v1.4 spec a két szélső véget is számolta (`N+1`) — ez **helytelen volt**. A fal végeit a sztekkvas-rúd vége tartja, nem kell külön láb.

**Sztekkvas-darabszám szabály (v1.3, kombinatorikus 0,5 + 1,0 m):**

```
Bemenet: N (cabinets in row), installType (ground-stack vagy prism)

Ha installType === 'ground-stack' VAGY 'totem':
  Egyenes szakasz, 1 m sztekkvas preferált
  Ha N páros:
    stackvas_1m = N/2
    stackvas_05m = 0
  Ha N páratlan:
    stackvas_1m = floor(N/2)
    stackvas_05m = 1

Ha installType === 'prism':
  Per oldal: sarkokra 0,5 m, közepére 1 m
  Per oldal N kabinet:
    Ha 4-side prism: minden oldal 2 sarokkal jár (de 2 oldal osztja meg, így 4 sarok össz)
    stackvas_05m_per_side = 1 (1 sarok / oldal a saját oldalához tartozik)
    stackvas_1m_per_side = ceil((N - 1) / 2)
  Halmozva: szorozd sideCount-tal
  (Részletes prism-logika v1.4-ben — Beninél tisztázandó)
```

**Példák (ground-stack, v1.5 szabály — N-1 front leg, H≥2m back brace):**

| W (m) | N (cabinets) | H (m) | Szabály | 1 m sztekkvas | 0,5 m sztekkvas | Belső pont (N−1) | Front legs (N−1) | Back braces (ha H≥2m) |
|---|---|---|---|---|---|---|---|---|
| 2,0 | 4 | 1,0 | páros, H<2 | 2 | 0 | 3 | 3 | 0 |
| 2,0 | 4 | 3,0 | páros, H≥2 | 2 | 0 | 3 | 3 | 2 |
| 2,5 | 5 | 2,5 | páratlan, H≥2 | 2 | 1 | 4 | 4 | 2 |
| 3,0 | 6 | 2,0 | páros, H=2 | 3 | 0 | 5 | 5 | 3 |
| 6,0 | 12 | 3,0 | páros, H≥2 | 6 | 0 | 11 | 11 | 6 |

> ⚠️ **v1.4 → v1.5 nagy javítás:** A v1.4-es spec az N+1 értelmezést használta (a két szélső véget is láb-helyzetnek vette). Egy 6 m × 3 m falra a v1.4 13 db front leget + 7 db back brace-t adott. **A helyes v1.5 érték: 11 db front leg + 6 db back brace.**

### 2.7. Hordládák (cabinet-tárolás)

**Üres konténer-rekordok (a kabineteket külön rakják bele):**

| Termék | Rentman ID | Code | Tartalom | Méret | Súly |
|---|---|---|---|---|---|
| **PL3.9W Plus V2 500×1000 konténer (4 fakkos)** | 4597 | 2155 | 4 db 500×1000 cabinet | 75×110×50 cm | 24,2 kg |
| **Absen konténer PL2.5 Plus V2 6-az-1-ben - műanyag** | 5475 | 2785 | 6 db 500×500 cabinet (P2.5 vagy P3.9 normál) | 83×59×74 cm | 24,3 kg |
| **Absen konténer PL2.5 Plus V2 SC 6-az-1-ben - műanyag** | **4602** | **2160** | 6 db 500×500 SC cabinet (csak SC variáns) | 83×58×60 cm | 24,3 kg |
| **PL2.5 Plus V2 8-az-1-ben fa** | *(Rentmanben nincs)* | *(létrehozandó)* | 8 db 500×500 cabinet | 92×61×78,5 cm | TBD |

> ⚠️ **4602 (SC konténer) leírás-hiba:** A Rentman description-je "Absen PL3.9W (0,5x0,5) üres konténer"-t mond, de a name szerint **PL2.5 Plus V2 SC**-hez tartozik. Beni-feladat: a description és internal_remark javítása. Mindenesetre az SC variánsú hordláda külön rekord (4602/2160).

**Kabinet-szintű AD KIT-ek (alternatíva — 6 / 4 kabinet + hordláda egy rekordba foglalva):**

| Termék | Rentman ID | Code | Tartalma | Súly (össz) | Power | Megjegyzés |
|---|---|---|---|---|---|---|
| **Absen PL3.9W kabinet \| 0,5×0,5 \| AD kit** | **4601** | **2159** | 6× PL3.9W 500×500 + konténer (5475-höz hasonló) | 74,3 kg | 2880 W, 12 A | empty_weight: 3,2 kg (csak konténer) |
| **Absen PL3.9W kabinet \| 0,5×1 \| AD kit** | **4596** | **2154** | 4× PL3.9W 500×1000 + 4-fakkos konténer (4597) | 74,2 kg | 1440 W, 6 A | volume: 0,4125 m³ |
| **Absen PL3.9W SC kabinet \| 0,5×0,5 \| AD kit** | **4603** | **2161** | 6× PL3.9W SC 500×500 + SC konténer (4602) | 74,3 kg | 2880 W, 12 A | empty_weight: 3,2 kg |
| **Absen PL2.5 Plus V2 kabinet \| 0,5×0,5 \| AD kit** | **5476** | **2786** | 6× PL2.5 500×500 + konténer (5475) | 74,3 kg | 2880 W, 12 A | empty_weight: 3,2 kg |

> **Sarbó-szabály v1.6 (kit-szintű BoM alternatíva):** A kalkulátor a default-ban **kabinet-szintű AD KIT-eket** is használhat (kit-szintű 1-soros BoM) a komponens-szintű részletezés helyett. Ha a fal pontosan többszöröse a 6-nak (vagy 4-nek a 500×1000 esetén), 1 kit = 1 sor a BoM-on. Az egyenlőtlen maradékot stand-alone kabinetekkel egészítjük ki.

**Default csomagolási szabály v1.6 (Sarbó 2026-05-11 megerősítés):**
- **P3.9 500×500 (normál + SC) + P2.5 500×500**: ugyanaz a hordláda (PL2.5 Plus V2 6-az-1-ben műanyag — mindkettő 500×500×78 mm fizikailag azonos). `ceil((p39_500x500 + p25_500x500) / 6)` → konténer; SC esetén külön (`ceil(pSC / 6)` → SC konténer).
- **P3.9 500×1000**: saját 4-fakkos hordláda. `ceil(p39_500x1000 / 4)` → 4597 (vagy a 4596 AD KIT, ha kabinet + konténer együtt egy rekordban kell).

### 2.8. Áramelosztás (kapcsolódó BoM-tételek)

A LED-fal teljes Watt-számából és panel-számából **két korlátot** számolunk, és a szigorúbb nyer:

| Elem | Szabály |
|---|---|
| **Áramkör — aktív teljesítmény** | `circuits_by_power = ceil(total_W × 1,25 / 3680)` (25% biztonsági ráhagyás) |
| **Áramkör — szivárgóáram** | `circuits_by_leakage = ceil(total_leakage_mA / 9)` — 30 mA RCD × 30% biztonsági limit |
| **Recommended áramkör** | `MAX(circuits_by_power, circuits_by_leakage)` — szigorúbb nyer |
| **CEE 16A elosztó** | 1 db / áramkör |
| **FI-relé (RCD) 30 mA** | 1 db / áramkör (közönség-közeli) |
| **Áramkör (3× 400V/16A/32A — opcionális)** | Manual flag, ha PV nagy fal-nál választja |

**Szivárgóáram-paraméterek (panelenként):**

| Pitch | Max W / panel | Avg W / panel | Earth leakage / panel | Forrás |
|---|---|---|---|---|
| **PL3.9W 500×500** | 150 W | 50 W | 0,7 mA (becslés) | Absen datasheet + analóg PL2.5 |
| **PL3.9W 500×1000** | 300 W | 100 W | 1,4 mA (becslés, 2× nagyobb panel) | Absen datasheet |
| **PL3.9W SC (corner)** | 150 W | 50 W | 0,7 mA (becslés) | Absen datasheet |
| **PL2.5 Plus V2 500×500** | 160 W | 54 W | **0,7 mA** | Sterling Event Group PL2.5 Pro spec (azonos tápdoboz V2-ben) |

> **Sarbó-szabály v1.5:** A 30 mA RCD-re max ~12 panel mehet a szivárgás-korlát miatt (12 × 0,7 = 8,4 mA < 9 mA limit), míg az aktív áram-korlát ~18 panel-t engedne. A **szivárgóáram-korlát dominál** P2.5-nél — ezt a v1.0-v1.4 spec figyelmen kívül hagyta.

**Áramfelvétel-képletek:**
- `total_W_max = panelek × per_cabinet_max_W × brightnessScale` (kültéri 1.0, beltéri 0.4)
- `total_W_avg = panelek × per_cabinet_avg_W × brightnessScale`
- `total_leakage_mA = panelek × per_cabinet_leakage` (NEM skálázódik a brightness-szel)
- `circuits_by_power = ceil(total_W_max × 1,25 / 3680)`
- `circuits_by_leakage = ceil(total_leakage_mA / 9)`
- `recommended_circuits = MAX(by_power, by_leakage)`

#### 2.8.1. Absen betápdoboz (kit-szintű elosztó) — Rentman 5718

Az ADAM raktárban van egy **Absen-specifikus betápdoboz** kit-szinten:

| Termék | Rentman ID | Code | Súly | Méret | Bemenet | Kimenet | Megjegyzés |
|---|---|---|---|---|---|---|---|
| **Absen betápdoboz \| 6 × 16A \| Kit** | **5718** | **3278** | 26,7 kg | 57×56,5×21 cm | **32A 3-fázisú CEE** | **6 × 16A egyfázisú** | A doboz 1 db Schuko-elosztót ad **6 áramkörrel**. Tartalmazza a 6 db RCD-t belső kapcsolószekrényben. Tags: `buza_patrik_2511, sarbo_gergo_202510` |

**Kit-szintű BoM-szabály (v1.6 új):**
```javascript
// Egy Absen betápdoboz 6 áramkört ad — kit-szintű ajánlat
const absenPowerBoxes = Math.ceil(recommended_circuits / 6);

// Komponens-szintű alternatíva (PV explicit kéri):
// - absenPowerBoxes db Absen betápdoboz, VAGY
// - recommended_circuits db CEE 16A elosztó + recommended_circuits db FI-relé 30mA + 3-fázis bevezetés
```

| Áramkör-szám | Kit (5718) | Komponens-bontás (alternatíva) |
|---|---|---|
| 1–6 | 1× Absen betápdoboz | 6× CEE 16A + 6× RCD + 1× 3-fáz bevezetés |
| 7–12 | 2× Absen betápdoboz | 12× CEE 16A + 12× RCD + 2× 3-fáz bevezetés |
| 13–18 | 3× Absen betápdoboz | 18× CEE 16A + 18× RCD + 3× 3-fáz bevezetés |

### 2.9. powerCON TRUE1 bekötő/átkötő kábelek — 6 tagú család

A v1.5-ben tévesen összekevertem a TRUE1 bekötőket. A v1.6 + komplett Rentman folder 314 bejárás (v1.7) feltárta, hogy **6 különböző** TRUE1 kábel létezik:

| Termék | Rentman ID | Code | Hossz | Súly | Szerep |
|---|---|---|---|---|---|
| **powerCON TRUE1 átkötő \| Absen \| 1,2m** | 4588 | 2146 | 1,2 m | 0,27 kg | Kabinet-közi power daisy (500×500-hoz, nagy táskában 12×) |
| **powerCON TRUE1 átkötő \| Absen \| 2m** | 4589 | 2147 | 2,0 m | 0,39 kg | Kabinet-közi power daisy (500×1000-hez, kicsi táskában 8×) |
| **powerCON TRUE1 átkötő \| Absen \| 6m** | **5700** | **3260** | 6 m | n/a | **Szakaszközi multi-feed átkötő** — hosszú daisy-chain átfutásra (pl. ha a 2 fal-szegmens távol van, vagy közbülső szakaszon kell csatlakoztatni) |
| **powerCON TRUE1 bekötő \| Absen \| 2m** | **5699** | **3259** | 2 m | n/a | **Rövid main feed** — ha az elosztódoboz közvetlenül a fal mellett van (a 20m túl hosszú) |
| **powerCON TRUE1 bekötő \| Absen \| 20m** | 4591 | 2149 | 20 m | 3,4 kg | **Standard main feed** (elosztó→1. kabinet, a kábeles táskákban) |
| **powerCON TRUE1 bekötő \| Novastar \| 1m** | 5664 | 3224 | 1 m | 3,4 kg | **CVT10-tetőszerelés** — utolsó kabinet powerCON-out-jából a CVT10-be |

**Telepítési mintázatok:**
- **A) Default — földi elosztó:** A CVT10 a földi rackben (Rack 2U), és a kabinetek a `4591/2149` main feed-jén (20 m) kapcsolódnak. A kábeles táskában már benne van.
- **B) Tetőre szerelt CVT10:** A CVT10 a fal tetejére kerül. Tápja az utolsó kabinet powerCON-out-jából, a **`5664/3224`** 1m TRUE1 bekötő szolgálja.
- **C) Rövid main feed:** Ha az elosztódoboz közvetlenül a fal mellett (≤ 2 m), a `5699/3259` 2m bekötő használható a 20m helyett — kevesebb kábel, takarosabb.
- **D) Szakaszközi multi-feed:** Ha a fal 2+ szegmensből áll (pl. fő- és oldalsó fal), a szegmensek közti átkötés a `5700/3260` 6m átkötővel megy — egy szegmens main feed-jéből oszik szét a másik szegmensre.

**Default BoM-szabály (v1.7):**
```javascript
// Main power feed — minden szegmenshez 1 db (a kábeles táskában már benne van,
// de multi-feed setup esetén kerülhet külön sor)
const mainPowerFeeds_20m = segments;  // 4591/2149

// Rövid main feed extra (PV explicit kérésre, ha elosztó közel)
const shortMainFeeds_2m = userRequestedShortFeed ? segments : 0;  // 5699/3259

// Szakaszközi átkötő (multi-segment setup-ban a szegmensek között)
const interSegmentFeeds_6m = (segments > 1) ? (segments - 1) : 0;  // 5700/3260

// CVT10 tetőszerelés (PV checkbox)
const cvtRoofMountCables_1m = isCvtOnRoof ? cvt_count : 0;  // 5664/3224
```

### 2.10. Absen kábeles család (Rentman folder 314) — a kábeles táska tartalmának komponensei

A teljes Rentman folder 314 bejárás (v1.7) feltárta a komplett Absen-kábel-családot. A két kábeles táska (4594, 4595) ezen tételek **Combination Default Content**-jeként van összeállítva:

| Termék | Rentman ID | Code | Hossz | Súly/db | Felhasználás |
|---|---|---|---|---|---|
| **powerCON TRUE1 átkötő \| Absen \| 1,2m** | 4588 | 2146 | 1,2 m | 0,27 kg | Kabinet-közi power daisy (500×500) |
| **powerCON TRUE1 átkötő \| Absen \| 2m** | 4589 | 2147 | 2,0 m | 0,39 kg | Kabinet-közi power daisy (500×1000) |
| **powerCON TRUE1 átkötő \| Absen \| 6m** | **5700** | **3260** | 6 m | n/a | **Szakaszközi multi-feed** (új, v1.7) |
| **powerCON TRUE1 bekötő \| Absen \| 2m** | **5699** | **3259** | 2 m | n/a | **Rövid main feed extra** (új, v1.7) |
| **powerCON TRUE1 bekötő \| Absen \| 20m** | 4591 | 2149 | 20 m | 3,4 kg | Main power feed |
| **etherCON átkötő \| Absen \| 1,1m** | 4590 | 2148 | 1,1 m | 0,11 kg | Kabinet-közi data daisy |
| **ethernet → etherCON bekötő \| Absen \| 30m** | 4592 | 2150 | 30 m | 1,2 kg | Main data feed |
| **ethernet kábel \| Vention \| 20m \| fekete** | 4593 | 2151 | 20 m | 0,35 kg | Backup data |

### 2.11. Absen kábeles táska (cabinet-csomag = kit + tárolódoboz)

A LED-fal cabinet-jeit szállítási konténerenként **egy kábeles táska** szolgálja ki — a 2.10. szekcióban felsorolt komponensekből összeállított komplett data + power csomag. Két méret, mindkettő `type: case` Rentmanben (a táska maga a kit).

| Termék | Rentman ID | Code | Súly | Méret | Kabinet-szám lefedés | Felhasználás |
|---|---|---|---|---|---|---|
| **Absen kábeles táska \| kicsi \| 8/8/1+1/1 \| AD kit** | **4594** | **2152** | 11,95 kg | 30×59×30 cm | **8 cabinet** (2× 4-az-1-ben) | **500×1000-es** kabinetekhez (P3.9 only) |
| **Absen kábeles táska \| nagy \| 12/12/1+1/1 \| AD kit** | **4595** | **2153** | 13,01 kg | 30×67×30 cm | **12 cabinet** (2× 6-az-1-ben) | **500×500-as** kabinetekhez (P3.9 normál/SC + P2.5) |

#### 2.11.1. Tartalom-decode (screenshot ground truth Sarbó 2026-05-11)

A `8/8/1+1/1` és `12/12/1+1/1` séma decode-ja a 2.10-es kábel-családi tételekkel:

| Komponens | Rentman | Kicsi (4594) | Nagy (4595) | Szerep |
|---|---|---|---|---|
| Stanley Fatmax ezüst táska | (Rentmanben nem rögzített) | 1× (59 cm) | 1× (67 cm) | Tárolás / szállítás |
| etherCON átkötő \| Absen \| 1,1m | **4590/2148** | **8×** | **12×** | Kabinet-közi data daisy-chain |
| powerCON TRUE1 átkötő \| Absen \| 2m | **4589/2147** | **8×** | — | Kabinet-közi power daisy (500×1000-hez) |
| powerCON TRUE1 átkötő \| Absen \| 1,2m | **4588/2146** | — | **12×** | Kabinet-közi power daisy (500×500-hoz) |
| ethernet → etherCON bekötő \| Absen \| 30m | **4592/2150** | 1× | 1× | Main data feed (processzor→1. kabinet) |
| ethernet kábel \| Vention \| 20m \| fekete | **4593/2151** | 1× | 1× | Backup data (redundáns ether) |
| powerCON TRUE1 bekötő \| Absen \| 20m | **4591/2149** | 1× | 1× | Main power feed (elosztó→fal) |

**A `8/8/1+1/1` séma:**
- `8` = power átkötő (kabinet-közi, 2m-es a kicsiben, 1,2m-es a nagyban)
- `8` = data átkötő (kabinet-közi, 1,1m-es mindkét táskában)
- `1+1` = 1 main data (30 m etherCON) + 1 backup data (20 m Vention) — **redundáns ether-pár**
- `1` = main power feed (20 m powerCON TRUE1)

> **Érdekesség:** a kicsi táska 2 m-es powerCON-átkötőkkel megy, a nagy csak 1,2 m-rel. Logikus: a kicsi a **500×1000-es** kabinetekhez (1 m magas → nagyobb port-távolság), a nagy a **500×500-as** kabinetekhez (0,5 m magas → rövidebb port-távolság).

#### 2.11.2. Kábeles táska-darabszám szabály

```javascript
// Nagy táska: 500×500-as kabinetekhez (P3.9 normal + SC + P2.5)
const bigCableKits = Math.ceil(
  (layout.p500x500 + layout.pSC + layout.p25_500x500) / 12
);

// Kis táska: 500×1000-es kabinetekhez (P3.9 only)
const smallCableKits = Math.ceil(layout.p500x1000 / 8);
```

| Konfiguráció | 500×500 | 500×1000 | Nagy (4595) | Kicsi (4594) |
|---|---|---|---|---|
| 6×3 m P3.9 (auto-1000) | 0 | 36 | 0 | ceil(36/8) = **5** |
| 6×3 m P3.9 (only-500) | 72 | 0 | ceil(72/12) = **6** | 0 |
| 4×2,5 m P2.5 | 40 | 0 | ceil(40/12) = **4** | 0 |
| 2×2×1,5 m P3.9 prism (3-side) | 36 (SC) | 0 | ceil(36/12) = **3** | 0 |

### 2.12. Totem-kit-ek (alternatív — előre-konfigurált függőleges fal)

A Rentmanben van **két előre-szerelvényezett** P2.5 totem-kit Virtual Package-ként. Ha PV totem-falat tervez, és a méret pontosan ezeknek a többszöröse, a kit-szintű 1-soros BoM egyszerűbb:

| Termék | Rentman ID | Code | Méret | Pixel-felbontás | Megjegyzés |
|---|---|---|---|---|---|
| **Absen P2.5 100×200 LED totem 400×800 AD kit** | 5548 | 2850 | 400 × 800 mm | 160 × 320 px (P2.5) | Virtual Package, total_stock=0 |
| **Absen P2.5 100×250 LED totem 400×1000 AD kit** | 5745 | 3297 | 400 × 1000 mm | 160 × 400 px (P2.5) | Virtual Package, total_stock=0 |

> ⚠️ **v1.6 megjegyzés:** Ezek Virtual Package-ek (a Beni-átdolgozás hatókörében). A kalkulátor `installType: totem` módban opcionálisan ajánlja ezeket a kit-eket — ha a fal méret pontosan 0,4×0,8 vagy 0,4×1,0 m. Egyébként a totem-módban a standard 500×500 P2.5 / P3.9 kabineteket használjuk.

### 2.13. Kabinet-összekötő elemek (csavarok + zárak + pillangós-csavar kit) — folder 314 (v1.7 új)

Az Absen-kabineteket **mechanikai csatlakozók** tartják össze: lábcsavarok az alsó kabinetekhez, kabinet-zárak a szomszédos kabinetek vertikális/horizontális összefogásához, és pillangós csavarok a fal-teljes-méretű egységen.

| Termék | Rentman ID | Code | Súly | Funkció |
|---|---|---|---|---|
| **M8x16 Absen csavar** | **4624** | **2182** | n/a | **Rövid kabinet-lábcsavar** — minden kabinet 4 lábcsúcsára, alsó kabinetnél a sztekkvashoz |
| **M8x60 csavar Absen** | **4623** | **2181** | n/a | **Hosszú kabinet-lábcsavar** — speciális rögzítéshez (pl. tetőre szerelt CVT10 vagy hasáb-sarkon) |
| **Szimpla Absen LED kabinet összefogó (2 csavarhoz)** | **4625** | **2183** | 0,12 kg | **Kabinet-zár** (LED cabinet lock, single) — szomszédos kabineteket fogja össze 2 csavarral, 10×10×0,3 cm fémlemez |
| **P3.9 Absen pillangós csavaros \| AD kit \| 5×3** | **4626** | **2184** | 6 kg | **Pillangós csavar kit** (LED cabinet lock set, koffer) — 5×3 kabinetes falra elegendő darabszám, 33×8×5 cm koffer |

**Darabszám-szabályok:**

```javascript
// Kabinet-zár (4625) — minden szomszédos kabinet-párhoz 1
// Egy NxM falban: vízszintes szomszédok = (N-1)×M, függőleges = N×(M-1)
const cabinetLocks = (N - 1) * M + N * (M - 1);

// Pillangós-csavar AD kit (4626) — egy 5×3 kit egy 15 kabinet falra elég
// Általánosított: ceil((N×M) / 15) AD kit
const pillangosKits = Math.ceil((N * M) / 15);

// Lábcsavar (4624 M8x16) — minden kabinet 4 lábcsúcsára
const screwsM8x16 = N * M * 4;

// Hosszú lábcsavar (4623 M8x60) — speciális helyzetekre, default 0
const screwsM8x60 = specialMounting ? specialMountingCount : 0;
```

> **PV-mező a UI-ban:** "Lock kit" checkbox — ha `true`, a 4626 (5x3 kit) BoM-on; ha `false`, a 4625 (szimpla) per-darab kerül.

### 2.14. Prizma pre-build kit-ek (alternatív — előre-konfigurált hasáb) — v1.7 új

A Rentmanben **két előre-szerelvényezett** P3.9 SC-prizma kit van Virtual Package-ként. Ha PV `installType: prism` módban a fal méret pontosan ezek többszöröse, a kit-szintű 1-soros BoM egyszerűbb:

| Termék | Rentman ID | Code | Méret | Pixel-felbontás | Megjegyzés |
|---|---|---|---|---|---|
| **P3.9 Absen 3 oldalú hasáb \| 0,5/1/0,5×1 \| 512×256 \| kit** | **5784** | **3336** | 3 oldalú hasáb 1m magas (0,5 + 1 + 0,5 m alaplap) | 512×256 px (P3.9) | Virtual Package, kis hasáb |
| **P3.9 Absen 3 oldalú hasáb \| 0,5/1/0,5×2 \| 512×512 \| AD kit** | **5931** | **3483** | 3 oldalú hasáb 2m magas (0,5 + 1 + 0,5 m alaplap) | 512×512 px (P3.9) | Virtual Package, nagy hasáb |

> ⚠️ **v1.7 megjegyzés:** Ezek **csak P3.9 SC kabinetekkel** mennek (90°-os hasáb-sarok megvalósítható csak SC variánssal — lásd 2.2. SC szemantika). A v1.7 kalkulátor `installType: prism` módban opcionálisan ajánlja ezeket a kit-eket, ha a fal méret pontosan 0,5+1+0,5 m alaplap × 1m vagy 2m magas. Egyébként a prizma-módban dinamikusan számoljuk a kabineteket (3 oldal × N×M × SC variáns).

### 2.15. Két párhuzamos LED-rendszer az ADAM-nál — explicit dokumentáció (v1.7 új)

A teljes Rentman LED-fal-szekció bejárás (v1.7, 2026-05-12) feltárta, hogy az ADAM-nál **két fizikailag is külön LED-rendszer fut párhuzamosan**, eltérő hardver-családdal és kabinet-mérettel. Ezt a spec mostantól explicit dokumentálja, hogy a jövőbeli fejlesztések (külön LBC/Qiangli-konfigurátor) ne tévessze meg a csapatot:

| Aspektus | **Absen rendszer (jelen spec scope-ja)** | **LBC + Qiangli + GKGD rendszer (külön spec scope-on kívül)** |
|---|---|---|
| **Kabinet-méret** | 500×500 / 500×1000 mm (PL3.9W, PL2.5 Plus V2) | 640×640 mm (GKGD/LBC P25), 640×480 mm (LBC Q25, QiangLi P4) |
| **Stack-vas család** | "Absen sztekkvas fix 1m" (5364), "fokolható 0,5m" (5473) | "LED stack vas" 0.48/0.64/1.28/3.2 m + toldó (883-887, 1699-1701), "LED álló stack vas 1.92" (880-882) |
| **Rigvas család** | "Absen 0,5/1m rigvas" (4622, 5154) + AD kit (5157) | "LBC 640 riggvas" (873), "QiangLi 640/1280 rigvas" (875-877), "QiangLi 3.2m" (2039), "GKGD 640" (872) |
| **Kábel-rendszer** | "powerCON TRUE1 Absen" 1,2/2/6/20 m, "etherCON Absen" 1,1/30 m, Vention 20 m backup | "GKGD powercon" 1/10 m (559, 560), "QiangLi CNLINKO LP M20" 0,7/5/10 m (857-859, 931), "Powercon átkötő 0,7 LBC" (927) |
| **Kábeles táska** | Absen 8/8/1+1/1 (4594) vagy 12/12/1+1/1 (4595) | LBC P2.5 **12/12/1/2** (2173/4615), QiangLi P4 **10/10/1/1** (953/3434), GKGD P2.5 **12/1** (967/3448) |
| **Hordláda** | "Absen PL2.5 6-az-1-ben műanyag" (5475) + SC variáns (4602) + PL3.9W 4-fakkos (4597) | "LBC kabinet 6-in-1 konténer" (402), "LBC két fakkos konténer #2" (404), "QiangLi P4 5-in-1" (414), "GKGD P25 6-in-1" (386) |
| **Vezérlés (processzor)** | **Novastar** — KU20 (4605), MX30 (4606), CVT10-S (5348), CVT10 KIT (5349), 10G SFP (5517), AD KIT-ek (4607, 4608), Rack 2U (5553) | **Colorlight** — X16 (3369), X2S (3370), X20 (3371), X4e (3372), A100 (3768) / A200 (3225) / C3 Pro (3348) CloudPlayer-ek, A4K (3347), CS4KPro (3349), IM9 receiver (3350), 5A-75B (3341) / 5A-75E (3342) receivers, H10FIX optika (4409, 4410), HSFP10 SFP (4408), CL-SSR-L16-RJ11 (3482), cloud előfizetés (2514) |
| **Kabinet-zár** | "Szimpla Absen LED kabinet összefogó" (4625), pillangós csavar AD kit (4626) | "Szimpla LBC LED kabinet összefogó" (932/3413), "Szimpla QiangLi LED kabinet összefogó" (878/3359), "Dupla QiangLi LED kabinet összefogó" (879/3360), "Dupla pillangós 18x AD kit" (1986/4428), "P2.5 pillangós csavaros AD kit" (1044/3526), "P4 pillangós csavaros AD kit" (1045/3527), "P25 6x rigvas AD kit" (1023/3505), "P4 4x rigvas AD kit" (1022/3504) |
| **LED pulpitus / takarólap** | nincs (Absen-családban nincs külön pulpitus) | Fehér / fekete / szürke LED pulpitus (3351, 3352, 1883, 2647), C3Pro-val kit (1630, 1631, 2071, 2057), LED takarólap 1.28x64 (3343) és 64x64 (3344) |
| **Konfigurátor scope** | **v1.0 — jelen spec** (`adam-led-szamolo` repó) | **Külön spec lesz** (kívül a v1.x scope-on, jövőbeli `adam-led-szamolo-colorlight` vagy bővítés) |

**Adatfeltöltési mintázat (v1.7 új tisztázás):**

A Rentmanben több rekord olyan **custom mezőt** tartalmaz, ami a **más rendszer** terminusait használja. Ezek **NEM hibák**, csak a Rentman-rekord-felvételénél a régi sablon-szöveg maradványa. Példák:
- **5517 (Novastar 10G SFP module SM)** `custom_16/19` = "Colorlight H10FIX optikai kifejtő" — a Novastar SFP-t a Colorlight-rendszer name-jával címkézték.
- A `defaultgroup: "LED"` és `folder: 64` ugyanaz mindkét rendszernél (közös LED-szekció).

**Konzekvenciák:**
- A v1.7 konfigurátor **csak az Absen-rendszerrel dolgozik**. Az LBC/Qiangli/GKGD-rendszerre később külön kalkulátor készül, vagy ez bővül több-rendszerű opcióval.
- A felhasználói felületen jelölni kell, hogy a kalkulátor **kifejezetten Absen ledfal-konfigurációra** való. Ha a PV nem-Absen kabinettel akar dolgozni, manuális Rentman-beillesztés szükséges.

### 2.16. Egyéb felfedezett tételek (referenciaként, nem default)

| Termék | Rentman ID | Code | Megjegyzés |
|---|---|---|---|
| **Absen P1.58 (Aries)** | 4563 | 2121 | Fine-pitch (1,58 mm), 4,8 kg, 33,75×5,4×60 cm méret. `total_stock = 0` — NEM ADAM-saját, valószínűleg beszerzés-tervezett. Kihagyva a v1.6 default-okból. |

---

## 3. Form-bemenetek (UI)

### 3.1. Alapparaméterek

```html
<form id="cfg-form" onchange="render()">
  <h3>Méret</h3>
  <label>Szélesség W (m): <input type="number" id="W" min="0.5" max="30" step="0.5" value="6"></label>
  <label>Magasság H (m): <input type="number" id="H" min="0.5" max="10" step="0.5" value="3"></label>

  <h3>Pitch</h3>
  <label><input type="radio" name="pitch" value="P3.9" checked> P3.9 (kültér / nagyobb beltér)</label>
  <label><input type="radio" name="pitch" value="P2.5"> P2.5 (közeli beltér)</label>

  <h3>Konfiguráció-típus</h3>
  <select id="installType">
    <option value="ground-stack">Földi stack</option>
    <option value="flown">Flown / függesztett (truss)</option>
    <option value="prism">Hasáb / kocka (SC kabinetekkel, 4-oldalú)</option>
    <option value="totem">Totem (keskeny álló)</option>
    <option value="curved">Hajlított ívelt fal (sugárral, normál kabinet)</option>
  </select>

  <h3>Fényerő (csak P3.9 esetén)</h3>
  <label><input type="radio" name="brightness" value="outdoor" checked> Kültéri 4500 nit (firmware konfig)</label>
  <label><input type="radio" name="brightness" value="indoor"> Beltéri 1500 nit (firmware konfig)</label>

  <h3>Hajlítás (csak curved esetén)</h3>
  <label>Sugár R (m): <input type="number" id="R" min="0.5" max="50" step="0.5" value="3"></label>
  <label>Szög (°): <input type="number" id="curveAngle" min="0" max="180" step="5" value="90"></label>

  <h3>Hasáb / kocka konfiguráció (csak prism esetén)</h3>
  <select id="prismShape">
    <option value="square-4side">Négyszögletes hasáb (4 oldal × W×H)</option>
    <option value="cube">Kocka (6 oldal, mind W×W)</option>
    <option value="triangle-3side">Háromszög hasáb (3 oldal)</option>
  </select>

  <h3>Nézési távolság (m)</h3>
  <input type="number" id="viewingDistance" min="1" max="100" step="0.5" value="10">

  <h3>Optikai kábel hossza (m, port-onként)</h3>
  <input type="number" id="fiberLength" min="5" max="300" step="5" value="50">

  <h3>Kabinet-preferencia (sík fal, csak P3.9)</h3>
  <select id="cabinetPref">
    <option value="auto-1000-prefer">Auto: 500×1000 preferálva (gazdaságos)</option>
    <option value="only-500">Csak 500×500</option>
    <option value="only-1000">Csak 500×1000</option>
  </select>
</form>
```

### 3.2. Automatikus paraméter-meghatározás

- A **W × H × pitch** kombinációból a kalkulátor kiszámolja:
  - kabinet-szám (cols, rows, total)
  - effektív felbontás (px_W × px_H)
  - effektív méret (W_eff = cols × 0,5 m, H_eff = rows × 0,5 m)
- **Felfelé kerekít** (pl. 5,3 × 2,7 m → 5,5 × 3,0 m = 11×6 = 66 kabinet 500×500-zal)
- Megjeleníti a **„kívánt vs. tényleges méret"** különbséget

---

## 4. Algoritmusok

### 4.1. `compute_panel_layout(W, H, pitch, cabinetPref, installType, prismShape)`

```javascript
function compute_panel_layout(W, H, pitch, cabinetPref, installType, prismShape) {
  const W_eff = Math.ceil(W * 2) / 2;
  const H_eff = Math.ceil(H * 2) / 2;
  const cols = W_eff / 0.5;
  const rows = H_eff / 0.5;
  let total = cols * rows;  // one side
  
  // PRISM mode: multiply by number of sides
  let sideCount = 1;
  if (installType === 'prism') {
    sideCount = (prismShape === 'cube') ? 6
              : (prismShape === 'triangle-3side') ? 3
              : 4;  // square-4side default
    total *= sideCount;
  }

  // Cabinet allocation
  let p500x500 = 0;
  let p500x1000 = 0;
  let pSC = 0;  // SC variants — only in P3.9, only in prism mode

  if (installType === 'prism' && pitch === 'P3.9') {
    pSC = total;  // ALL SC in prism mode (P3.9)
  } else if (installType === 'prism' && pitch === 'P2.5') {
    // Szabály 7 (Sarbó 2026-05-11 v1.5): P2.5 + prism KATEGORIKUSAN TILTVA
    // A validate() függvény ezt errs-be teszi — compute_layout itt csak hibás állapotot jelez
    throw new Error('P2.5 + prism TILTVA — nincs SC variáns, fokolás max ±7,5°/+10°');
  } else if (pitch === 'P2.5') {
    p500x500 = total;  // P2.5 only 500x500
  } else if (cabinetPref === 'only-500') {
    p500x500 = total;
  } else if (cabinetPref === 'only-1000') {
    if (rows % 2 !== 0) {
      p500x500 = total;  // fallback
    } else {
      p500x1000 = cols * (rows / 2);
    }
  } else { // auto-1000-prefer
    const rowsAs1000 = Math.floor(rows / 2) * 2;
    const remRows = rows - rowsAs1000;
    p500x1000 = cols * (rowsAs1000 / 2);
    p500x500 = cols * remRows;
    // For prism mode, normal cabinets used too — but per the spec, prism = SC only
  }

  return {
    cols, rows, total, sideCount,
    W_eff, H_eff,
    px_W: cols * (pitch === 'P3.9' ? 128 : 200),
    px_H: rows * (pitch === 'P3.9' ? 128 : 200),
    p500x500, p500x1000, pSC,
    isPrism: installType === 'prism'
  };
}
```

### 4.2. `compute_power(layout, pitch, brightness)`

```javascript
function compute_power(layout, pitch, brightness) {
  const perPanel = {
    'P3.9': {
      '500x500':  {max: 150, avg: 50, leakage: 0.7},
      '500x1000': {max: 300, avg: 100, leakage: 1.4},  // 2× nagyobb panel
      'SC':       {max: 150, avg: 50, leakage: 0.7}
    },
    'P2.5': {
      '500x500':  {max: 160, avg: 54, leakage: 0.7}  // Sterling Event Group ground truth
    }
  };

  const scale = (pitch === 'P3.9' && brightness === 'indoor') ? 0.40 : 1.00;

  const pp = perPanel[pitch];
  let totalMax = 0, totalAvg = 0, totalLeakage = 0;
  let panelCount = 0;

  totalMax += layout.p500x500 * (pp['500x500']?.max || 0);
  totalAvg += layout.p500x500 * (pp['500x500']?.avg || 0);
  totalLeakage += layout.p500x500 * (pp['500x500']?.leakage || 0);
  panelCount += layout.p500x500;
  
  if (pitch === 'P3.9') {
    totalMax += layout.p500x1000 * pp['500x1000'].max;
    totalAvg += layout.p500x1000 * pp['500x1000'].avg;
    totalLeakage += layout.p500x1000 * pp['500x1000'].leakage;
    panelCount += layout.p500x1000;
    
    totalMax += layout.pSC * pp['SC'].max;
    totalAvg += layout.pSC * pp['SC'].avg;
    totalLeakage += layout.pSC * pp['SC'].leakage;
    panelCount += layout.pSC;
  }

  totalMax *= scale;
  totalAvg *= scale;
  // Leakage NEM skálázódik a brightness-szel (a tápdoboz szivárgása konstans)

  // KÉT KORLÁT az áramkör-számoláshoz (Sarbó-szabály v1.5):
  // 1. Aktív teljesítmény: ceil(totalW × 1.25 / 3680)
  const circuits_by_power = Math.ceil(totalMax * 1.25 / 3680);
  
  // 2. Szivárgóáram: max 9 mA / 30 mA RCD (30% biztonsági puffer a trip-pontig)
  const circuits_by_leakage = Math.ceil(totalLeakage / 9);
  
  // A SZIGORÚBB korlát nyer (gyakran a szivárgás dominál P2.5-nél)
  const recommended_circuits = Math.max(circuits_by_power, circuits_by_leakage);

  return {
    totalMax, totalAvg, totalLeakage, panelCount, scale,
    circuits_by_power, circuits_by_leakage,
    circuits_16A: recommended_circuits
  };
}
```

> **Szivárgóáram-szabály részletes magyarázat (v1.5 új):**
> 
> A 30 mA-es FI-relé (RCD) trip-pontja 30 mA, de a **biztonságos üzemi limit 30%** = **9 mA** összes szivárgás. A maradék 21 mA-es puffer szükséges:
> - a tényleges hibaesemény-detektálásra (testzárlat),
> - a hálózati felharmonikusok + EMI-szivárgás-tartalékra,
> - a hőkapacitív szivárgás-emelkedésre indításnál (LED-falaknál különösen jelentős).
> 
> **PL2.5 Plus V2 szivárgóáram:** 0,7 mA / panel (Sterling Event Group spec PL2.5 Pro-ra; a V2 tápdoboz-architektúra azonos). Egy 30 mA-es RCD-re így max **~12 panel** (12 × 0,7 = 8,4 mA < 9 mA limit) — még jóval kevesebb, mint az aktív áram-alapú 18 panel.
> 
> **Példa (40 panel P2.5):**
> | Korlát | Számítás | Áramkör |
> |---|---|---|
> | Aktív (kültéri max) | 40 × 160 × 1,25 / 3680 | 3 |
> | Szivárgás | 40 × 0,7 / 9 | **4** ✅ |
> | **Recommended** | MAX | **4** |
> 
> Ez a P2.5 falaknál a szivárgóáram-korlát DOMINÁL — a v1.4 spec ezt nem vette figyelembe.

### 4.3. `compute_data(layout, pitch)` — Fizikai komponens-lánc (v1.6 — fizikai AD KIT default)

**Sarbó-szabály v1.6:** A v1.5-ben stand-alone MX30 + CVT10 + SFP komponenseket javasoltam. A teljes Rentman-bejárás után kiderült, hogy létezik **fizikai AD KIT** (4608/2166 MX30 10E+2S AD KIT — `type: case`, NEM Virtual Package), ami a stand-alone MX30 + 2× SFP-t **egyetlen hordtáskás rekordban** kínálja. A v1.6 default mostantól ezt használja → **egyszerűbb BoM** (kevesebb sor, magától tartalmazza a flight case-t).

```javascript
function compute_data(layout, pitch) {
  const totalPixels = layout.px_W * layout.px_H * (layout.sideCount || 1);
  
  // v1.6 default: MX30 AD KIT (4608) + standalone CVT10 + extra SFP + Rack 2U
  // - MX30 AD KIT tartalma: 1× MX30 (4606) + 2× SFP (5517) + flight case
  // - + extra SFP modul a CVT10-ek számára (2× CVT10 → 2× SFP a CVT10-eken)
  
  let mx30Kits, cvtCount, extraSfp, rackCount, kuKit;
  
  if (totalPixels <= 6_500_000) {
    // Default: 1× MX30 AD KIT + 2× CVT10-S + 2× extra SFP + 1× Rack 2U
    mx30Kits = {count: 1, rentmanId: 4608, code: '2166', name: 'Novastar MX30 | 10E + 2S | AD KIT'};
    cvtCount = 2;  // master + backup
    extraSfp = 2;  // 1 per CVT10 (a CVT10-be 1 db kell az optikai port-hoz)
    rackCount = 1;
    kuKit = null;
  } else if (totalPixels <= 13_000_000) {
    // Nagyobb fal: 2× MX30 AD KIT + 4× CVT10 + 4× extra SFP + 2× Rack 2U
    mx30Kits = {count: 2, rentmanId: 4608, code: '2166', name: 'Novastar MX30 | 10E + 2S | AD KIT'};
    cvtCount = 4;
    extraSfp = 4;
    rackCount = 2;
    kuKit = null;
  } else {
    // > 13 M pixel: 3+ MX30 AD KIT → Beni-konzultáció
    const n = Math.ceil(totalPixels / 6_500_000);
    mx30Kits = {count: n, rentmanId: 4608, code: '2166', name: 'Novastar MX30 | 10E + 2S | AD KIT'};
    cvtCount = n * 2;
    extraSfp = n * 2;
    rackCount = n;
    kuKit = null;
  }
  
  // CVT10 alternatíva: ha 3 CVT10 kell, 1× CVT10 KIT (5349, 14,39 kg) olcsóbb mint 3× standalone
  // PV-választás — default standalone, csak ha cvtCount >= 3 ajánljunk CVT10 KIT-et
  const cvtKitAlternative = (cvtCount >= 3) ? {
    rentmanId: 5349, code: '2667', count: Math.floor(cvtCount / 3),
    standaloneRemainder: cvtCount % 3
  } : null;
  
  // Pixel-port szabály: 1 ethernet port per 650K pixel (Gigabit Ethernet bandwidth, 50Hz/8bit)
  const fiberPortCapacity = 650_000;
  const ethPortsNeeded = Math.max(1, Math.ceil(totalPixels / fiberPortCapacity));
  const ethPortsAvailable = cvtCount * 10;  // 10 Ethernet / CVT10
  
  return {
    totalPixels, ethPortsNeeded, ethPortsAvailable,
    components: {
      mx30Kit: mx30Kits,
      cvt: {count: cvtCount, rentmanId: 5348, code: '2666', name: 'Novastar CVT10-S'},
      cvtKitAlt: cvtKitAlternative,
      extraSfp: {count: extraSfp, rentmanId: 5517, code: '2819', name: 'Novastar 10G SFP module (SM)'},
      rack: {count: rackCount, rentmanId: 5553, code: '2855', name: 'Novastar Rack 2U Double Door'}
    }
  };
}
```

> **A komponens-lánc magyarázata (v1.6):**
> - **MX30 AD KIT (4608)** tartalma: 1 MX30 + 2 SFP-modul (a 2 optikai port-ra) + flight case (20,67 kg total, 1 kg flight case empty). A 2 SFP-modul a processzor-oldali optikai out-okhoz.
> - **+2 CVT10-S (5348)** standalone: a fal-oldalon kifejtve. Master OPT1 + Backup OPT2 (redundáns).
> - **+2 extra SFP (5517)** a CVT10-ek optikai bemenetéhez (1 db per CVT10).
> - **+1 Rack 2U (5553)** a rack-konténerhez (MX30 + 2 CVT10 együtt fér el).
> - **Total SFP a láncon:** 2 (MX30-on, az AD KIT-ben már benne) + 2 (CVT10-eken, extra) = 4 db.

### 4.4. `compute_rigging(layout, installType)` — Absen rigvas család

```javascript
function compute_rigging(layout, installType) {
  if (installType !== 'flown') {
    return {rigvas_1m: 0, rigvas_05m: 0, rigvasKit: 0, safetySteel: 0};
  }
  const W = layout.W_eff;
  const rigvas_1m_raw = Math.floor(W);
  const rigvas_05m = Math.ceil((W - rigvas_1m_raw) / 0.5);
  
  // AD kit (10x 1m rigvas container) — economical if 8+ needed (Sarbó 2026-05-11 megerősítve)
  let rigvasKit = 0;
  let rigvas_1m = rigvas_1m_raw;
  if (rigvas_1m_raw >= 8) {
    rigvasKit = 1;
    rigvas_1m = rigvas_1m_raw - 10;  // 10× covered by kit
    if (rigvas_1m < 0) rigvas_1m = 0;
  }
  
  // Safety steel: 1 per rigvas point + 2 backup
  const safetySteel = rigvas_1m_raw + rigvas_05m + 2;
  
  return {rigvas_1m, rigvas_05m, rigvasKit, safetySteel};
}
```

### 4.5. `compute_stack(layout, installType)` — Absen stack család (v1.5 N−1 szabály)

```javascript
function compute_stack(layout, installType) {
  if (!['ground-stack', 'totem', 'prism'].includes(installType)) {
    return {stackvas_1m: 0, stackvas_05m: 0, frontSupportLegs: 0, cabinetSupportBraces: 0};
  }
  
  const cols = layout.cols;       // kabinet-szám egy egyenes sorban
  const H = layout.H_eff;
  
  let stackvas_1m, stackvas_05m, internalConnectionPoints;
  
  if (installType === 'prism') {
    // Prism mode: per side sarkok 0,5m, közepén 1m
    const N = cols;
    const sides = layout.sideCount;
    
    const remaining = N - 1;
    let stackvas_1m_perSide, stackvas_05m_perSide;
    if (remaining % 2 === 0) {
      stackvas_1m_perSide = remaining / 2;
      stackvas_05m_perSide = 1;
    } else {
      stackvas_1m_perSide = Math.floor(remaining / 2);
      stackvas_05m_perSide = 2;
    }
    
    stackvas_1m = stackvas_1m_perSide * sides;
    stackvas_05m = stackvas_05m_perSide * sides;
    internalConnectionPoints = (N - 1) * sides;  // belső csatlakozási pontok per side
    // Note: sarkok osztva — v1.6 finomítás
    
  } else {
    // Ground-stack vagy totem: egyenes szakasz
    const N = cols;
    if (N % 2 === 0) {
      stackvas_1m = N / 2;
      stackvas_05m = 0;
    } else {
      stackvas_1m = Math.floor(N / 2);
      stackvas_05m = 1;
    }
    internalConnectionPoints = N - 1;  // N−1 belső kabinet-kabinet illesztés
  }
  
  // Front legs: MINDEN belső csatlakozási ponthoz előre (Sarbó 2026-05-11 v1.5)
  // A két szélső véget a sztekkvas-rúd vége tartja — nem kell külön láb
  const frontSupportLegs = internalConnectionPoints;
  
  // Back braces: MINDEN MÁSODIK belső ponthoz, csak ha H >= 2,0m
  // (H < 2,0 m esetén fizikailag NEM fér el a kabinet kitámasztó)
  const cabinetSupportBraces = (H >= 2.0)
    ? Math.ceil(internalConnectionPoints / 2)
    : 0;
  
  return {
    stackvas_1m, stackvas_05m, internalConnectionPoints,
    frontSupportLegs, cabinetSupportBraces
  };
}
```

> **Példa (W=6 m, H=3 m, ground-stack):** cols=12, internalConnectionPoints=11 → 6× 1m sztekkvas + 0× 0,5m + **11× front leg** + **6× back brace** (H≥2m).

> **Példa (W=6 m, H=1,5 m, ground-stack):** cols=12, internalConnectionPoints=11 → 6× 1m sztekkvas + 0× 0,5m + **11× front leg** + **0× back brace** (H<2m, fizikailag nem fér el).

### 4.5.bis `compute_cable_kits(layout, pitch)` — Absen kábeles készletek (v1.3 új)

```javascript
function compute_cable_kits(layout, pitch) {
  // Big kit (12 cabinet): 500×500-as kabinetekre
  const cab500x500_total = layout.p500x500 + layout.pSC + 
    (pitch === 'P2.5' ? layout.p500x500 : 0);  // P2.5 csak 500x500
  const bigCableKits = Math.ceil(cab500x500_total / 12);
  
  // Small kit (8 cabinet): 500×1000-es kabinetekre
  const smallCableKits = Math.ceil(layout.p500x1000 / 8);
  
  return {bigCableKits, smallCableKits};
}
```

### 4.6. `compute_cases(layout, pitch)`

```javascript
function compute_cases(layout, pitch) {
  const cases = [];
  if (pitch === 'P3.9') {
    if (layout.p500x500 > 0 || layout.pSC > 0) {
      const n500 = Math.ceil((layout.p500x500 + layout.pSC) / 6);
      cases.push({sku: 'ABSEN-FC-PL2.5-PLUS-V2-6IN1-PLASTIC', name: 'Absen 6-az-1-ben hordláda (500×500)', db: n500});
    }
    if (layout.p500x1000 > 0) {
      const n1000 = Math.ceil(layout.p500x1000 / 4);
      cases.push({sku: 'PL3.9-FC-4IN1', name: 'PL3.9W Plus V2 500×1000 konténer (4 fakkos)', db: n1000});
    }
  } else { // P2.5
    const n = Math.ceil(layout.p500x500 / 6);
    cases.push({sku: 'ABSEN-FC-PL2.5-PLUS-V2-6IN1-PLASTIC', name: 'Absen 6-az-1-ben hordláda (500×500)', db: n});
  }
  return cases;
}
```

### 4.7. `compute_optical_cables(data, fiberLength)`

```javascript
function compute_optical_cables(data, fiberLength) {
  // Each fiber port between processor and CVT10 needs 1 LC-LC duplex fiber cable
  // Each Ethernet port between CVT10 and LED panels needs 1 CAT6 SFTP cable (~2-5m typical)
  return {
    fiberCableCount: data.procCount * 2,  // 1 master + 1 backup per processor optical out
    fiberCableLength: fiberLength,
    cat6Cables: data.ethPortsNeeded,
    cat6LengthDefault: 5  // 5m default per cable
  };
}
```

---

## 5. BoM-szerkezet (7 csoport, fix sorrend)

A `bom` egy tömb, amely `{grp: 'csoport-név'}` markereket és `{csop, sku, qty, uom, note}` sorokat tartalmaz. A csoportok sorrendje:

1. **LED-kabinetek** — `p500x500`, `p500x1000`, `pSC` darabszám szerint
2. **Processzor + adatlánc** — NovaStar KU20 vagy MX30 + CVT10 + fiber kábelek + CAT6 kábelek
3. **Rigging** (csak flown) — 1,0 m + 0,5 m rigvas + AD kit konténer + Safety Steel
4. **Stack szerkezet** (csak ground-stack / totem / prism) — alsó sztekkvas + első kitámasztó + kabinet kitámasztó (ha H ≥ 1,5 m)
5. **Áramelosztás** — circuit-darab × CEE 16A elosztó + FI-relé + tápkábel-méter (becsült)
6. **Hordládák** — kabinet-konténerek (Rigvas konténer már a 3. csoportban)
7. **Kiegészítők + info-sorok** — betekintő monitor + ügyeleti laptop + ⚠ FW-konfig sor („Beltéri/kültéri firmware feltöltés a helyszínen a fal összeállása után")

A teljes `build_bom_grouped()` függvény minden csoportot a megfelelő sorrendben felfűz.

---

## 6. Tömeg/térfogat/áram-aggregátor (`massVolMap`)

```javascript
const massVolMap = {
  // LED-kabinetek (Rentman 4564/4565/4566/5474)
  'ABSEN-PL3.9W-PLUSV2-500x500-STANDARD': {kg: 8.0, m3: 0.020, cat: 'LED-kabinetek', rentman_code: '2123', rentman_id: 4565},
  'ABSEN-PL3.9W-PLUSV2-500x500-CORNER':   {kg: 8.0, m3: 0.020, cat: 'LED-kabinetek', rentman_code: '2124', rentman_id: 4566},
  'ABSEN-DISPLAY-P3.9-500x1000-OUTDOOR-7680HZ': {kg: 13.0, m3: 0.040, cat: 'LED-kabinetek', rentman_code: '2122', rentman_id: 4564},
  'ABSEN-PL2.5-PLUS-V2-500x500-STANDARD': {kg: 7.8, m3: 0.020, cat: 'LED-kabinetek', rentman_code: '2784', rentman_id: 5474},

  // Processzor + adatlánc — RENTMAN GROUND TRUTH (12 NovaStar-tétel)
  'NOVASTAR-KU20-OPTIKA-6PORT-AD-KIT': {kg: 8.82, m3: 0.030, cat: 'Processzor + adatlánc', rentman_code: '2678', rentman_id: 5360, note: 'Virtual Package — KU20+CVT10+SFP'},
  'NOVASTAR-MX30-OPTIKA-10PORT-AD-KIT': {kg: 22.0, m3: 0.060, cat: 'Processzor + adatlánc', rentman_code: '2668', rentman_id: 5350, note: 'Virtual Package — MX30+CVT10+SFP'},
  'NOVASTAR-MX30-OPTIKA-20PORT-REDUNDANS': {kg: 35.0, m3: 0.100, cat: 'Processzor + adatlánc', rentman_code: '2857', rentman_id: 5555, note: 'Virtual Package — MX30+2× CVT10 redundáns'},
  
  // Standalone NovaStar tételek (csak ha PV bontást kér)
  'NOVASTAR-KU20':                {kg: 2.1, m3: 0.004, cat: 'Processzor + adatlánc', rentman_code: '2163', rentman_id: 4605, power_w: 25},
  'NOVASTAR-MX30':                {kg: 7.2, m3: 0.020, cat: 'Processzor + adatlánc', rentman_code: '2164', rentman_id: 4606, power_w: 55},
  'NOVASTAR-KU20-6E-1S-AD-KIT':   {kg: 8.82, m3: 0.030, cat: 'Processzor + adatlánc', rentman_code: '2165', rentman_id: 4607, note: 'KU20 + flight case'},
  'NOVASTAR-MX30-10E-2S-AD-KIT':  {kg: 20.67, m3: 0.060, cat: 'Processzor + adatlánc', rentman_code: '2166', rentman_id: 4608, note: 'MX30 + flight case'},
  'NOVASTAR-CVT10-S':             {kg: 2.5, m3: 0.005, cat: 'Processzor + adatlánc', rentman_code: '2666', rentman_id: 5348},
  'NOVASTAR-CVT10-KIT':           {kg: 14.39, m3: 0.030, cat: 'Processzor + adatlánc', rentman_code: '2667', rentman_id: 5349, note: '3× CVT10 + kábelek konténerben'},
  'NOVASTAR-10G-SFP-MODULE-SM':   {kg: 0.05, m3: 0.0001, cat: 'Processzor + adatlánc', rentman_code: '2819', rentman_id: 5517},
  'NOVASTAR-RACK-2U-DOUBLE-DOOR': {kg: 8.0, m3: 0.055, cat: 'Processzor + adatlánc', rentman_code: '2855', rentman_id: 5553, note: '63×16×55 cm, KU20+CVT10 vagy MX30+CVT10 fér el'},

  // Rigging (Rentman 4622/5154/5157)
  'ABSEN-RIGVAS-0.5M':        {kg: 3.8, m3: 0.005, cat: 'Rigging', rentman_code: '2180', rentman_id: 4622},
  'ABSEN-RIGVAS-1.0M':        {kg: 7.3, m3: 0.010, cat: 'Rigging', rentman_code: '2482', rentman_id: 5154},
  'ABSEN-RIGVAS-AD-KIT-10X':  {kg: 113, m3: 0.420, cat: 'Rigging', rentman_code: '2485', rentman_id: 5157, note: '10× 1m rigvas konténerben'},
  'SAFETY-STEEL-1.5M':        {kg: 0.3, m3: 0.0002, cat: 'Rigging'},

  // Stack (Rentman 5364/5365/5441/5473)
  'ABSEN-STACK-VAS-FIX-1M':       {kg: 11.4, m3: 0.028, cat: 'Stack szerkezet', rentman_code: '2682', rentman_id: 5364, note: 'Egyenes szakaszra'},
  'ABSEN-STACK-VAS-FOKOLHATO-05M':{kg: 8.6,  m3: 0.010, cat: 'Stack szerkezet', rentman_code: '2783', rentman_id: 5473, note: 'Fokolható (szögbe rakható) — sarkokhoz'},
  'ABSEN-STACK-FRONT-LEG':        {kg: 0.5,  m3: 0.0004, cat: 'Stack szerkezet', rentman_code: '2683', rentman_id: 5365},
  'ABSEN-CABINET-SUPPORT-BRACE':  {kg: 5.0,  m3: 0.010, cat: 'Stack szerkezet', rentman_code: '2759', rentman_id: 5441},

  // Áramelosztás (powerCON TRUE1 = Rentman 5664)
  'POWERCON-TRUE1-NOVASTAR-20M':  {kg: 3.4, m3: 0.020, cat: 'Áramelosztás', rentman_code: '3224', rentman_id: 5664, note: '20 m hosszú (a Rentman-name "1m" hibás)'},
  'CEE-16A-ELOSZTO-5x':           {kg: 2.5, m3: 0.008, cat: 'Áramelosztás'},
  'FI-RELE-30MA':                 {kg: 0.4, m3: 0.001, cat: 'Áramelosztás'},

  // Kábeles készletek (Rentman 4594/4595 — táska = konténer egyben)
  'ABSEN-CABLE-BAG-SMALL-8CAB':  {kg: 11.95, m3: 0.053, cat: 'Kábeles készletek', rentman_code: '2152', rentman_id: 4594, note: '8 cabinet (2× 4-az-1-ben), 8/8/1+1/1 tartalom'},
  'ABSEN-CABLE-BAG-LARGE-12CAB': {kg: 13.01, m3: 0.060, cat: 'Kábeles készletek', rentman_code: '2153', rentman_id: 4595, note: '12 cabinet (2× 6-az-1-ben), 12/12/1+1/1 tartalom'},

  // Hordládák (Rentman 4597 + Odoo 927)
  'PL3.9-FC-4IN1':                                  {kg: 24.2, m3: 0.410, cat: 'Hordládák', rentman_id: 4597},
  'ABSEN-FC-PL2.5-PLUS-V2-6IN1-PLASTIC':            {kg: 24.3, m3: 0.410, cat: 'Hordládák', rentman_id: 5475, note: 'P2.5 és P3.9 500×500 közös hordlád'},

  // Kiegészítők
  'BETEKINTO-MONITOR-DELL':   {kg: 4.5, m3: 0.025, cat: 'Kiegészítők'},
  'UGYELET-LAPTOP-KIT':       {kg: 3.0, m3: 0.020, cat: 'Kiegészítők'}
};
```

> A Rentman-ground-truth súlyok (NovaStar KU20 = 2,1 kg pontosan a datasheet 4,63 lbs-vel egyezik, NovaStar MX30 = 7,2 kg az Absen 1U rack-súlyához mérve normál). A standalone tételek a v1.2-spec hivatalos forrásai, az AD kit súlya (8,82 / 20,67 kg) a Rentman virtuális-csomag-súlyaival lett egyeztetve.

---

## 7. Vizualizációk — 4 SVG-nézet

### 7.1. Általános koordináta-rendszer

- **Elölnézet:** x = W (jobbra), y = H (fent). Kabinet-rács W × H, minden cellában a pixel-felbontás kis felirattal. Prism módban → mind az N oldal egymás mellett ablakok-szerűen.
- **Oldalnézet:** x = vastagság (78 mm + rigging-távolság), y = H. Stack-magasság, függesztő-rúd vagy stack-vas látható.
- **Felülnézet:** csak `curved` (ívelt) és `prism` módnál releváns. Sík falnál „N/A — sík fal".
- **3D-iso:** kabinett-projekció, `skewX(0.45) + skewY(0.65)`. Színpadhát-szerű perspektíva.

### 7.2. CSS-paletta (LED-specifikus)

```css
/* LED-kabinetek */
.led-pnl-39      { fill: #1a1a1a; stroke: #000; stroke-width: 0.025; }
.led-pnl-39-sc   { fill: #2a2018; stroke: #c89821; stroke-width: 0.025; }   /* arany kontúr SC-hez */
.led-pnl-39-1000 { fill: #1a1a1a; stroke: #000; stroke-width: 0.025; }
.led-pnl-25      { fill: #0a0a0a; stroke: #000; stroke-width: 0.025; }
.led-glow        { fill: #4a90e2; opacity: 0.15; }

/* Stack és rigging */
.stack-vas       { fill: #6e6359; stroke: #1a1612; stroke-width: 0.020; }
.stack-front-leg { fill: #8a8a8a; stroke: #1a1612; stroke-width: 0.015; }
.stack-brace     { fill: #b94e1f; stroke: #1a1612; stroke-width: 0.015; }
.rigvas          { fill: #b94e1f; stroke: #1a1612; stroke-width: 0.015; }
.safety-steel    { stroke: #ff0000; stroke-width: 0.020; stroke-dasharray: 0.05,0.03; fill: none; }

/* Truss-keret (portal) */
.truss-rail      { fill: #ccc; stroke: #666; stroke-width: 0.020; }

/* Hajlítás */
.curve-arc       { stroke: #4a90e2; stroke-width: 0.030; fill: none; stroke-dasharray: 0.10,0.05; }

/* Prism (hasáb) sarkok */
.prism-corner    { stroke: #c89821; stroke-width: 0.025; fill: none; stroke-dasharray: 0.15,0.08; }

/* Feliratok */
.lbl-pitch       { font-family: 'JetBrains Mono', monospace; font-size: 0.18px; fill: #888; }
.lbl-dim         { font-family: 'JetBrains Mono', monospace; font-size: 0.22px; fill: #6e6359; font-weight: 700; }
.lbl-front       { font-family: 'JetBrains Mono', monospace; font-size: 0.18px; fill: #b94e1f; font-weight: 700; }
.lbl-prism-side  { font-family: 'JetBrains Mono', monospace; font-size: 0.20px; fill: #c89821; font-weight: 700; }
```

### 7.3. `draw_frontview(layout, pitch, installType)` — fő nézet

A fő nézet a kabinet-rácsot mutatja:
- Sötét háttér (LED-fal ki van kapcsolva állapotban)
- Minden cella (500×500 vagy 500×1000) **kontúrral elválasztva**
- SC kabinetek **arany kontúrral** (csak prism módban)
- Méret-feliratok W (alul) és H (balra, elforgatva)
- Pixel-felbontás középen nagy felirattal: pl. `1920 × 720`
- Pitch + fényerő-felirat felül: pl. `Absen P3.9 kültéri | 4500 nit (firmware konfig)`
- **Prism módban** mind az N oldal külön kis ábraként, sarkok kiemelve

### 7.4. `draw_sideview(layout, installType)`

- Az **oldalnézet** a vastagságot + a rigvasokat (flown) vagy a stack-vasakat (ground) mutatja
- Magasság-vonal H_eff-fel, kabinet-rétegek (~50 cm)
- Flown módban a rigvas tipikus magasságban (H_eff + 0.3 m felett), Safety Steel piros szaggatott vonallal
- Ground módban: 
  - Alul: `stack-vas` (~5 cm vastag, 1 m vagy 0,5 m szakaszok)
  - Lent elöl: `stack-front-leg` (kis háromszög-támaszok, **N−1 db** belső csatlakozási pontoknál)
  - Hátul: `stack-brace` (csak ha **H ≥ 2,0 m**, fizikailag nem fér el alacsonyabbnál)

### 7.5. `draw_topview(layout, installType, curveR, curveAngle, prismShape)`

- **Sík fal:** rajzol egy **vízszintes vonalat**, `N/A — sík fal` felirattal
- **Curved:** rajzol egy **ívet** a sugár-paraméter szerint, kabinet-pontok az ív mentén; méret-feliratok: szelvény-hossz, sugár, szög
- **Prism:**
  - `square-4side` → négyzet vagy téglalap körvonal, az oldalak hosszával
  - `cube` → négyzet (felülről kocka = négyzet), oldalhossz felirat
  - `triangle-3side` → szabályos háromszög körvonal

### 7.6. `draw_iso3d(layout, installType)`

- 3D iso-projekció a falra
- Háttér: padló-vonal + truss-keret (flown)
- Front-vonal (közönség felé) jelölés
- Prism módban a 3D-iso látványosan mutatja a hasábot (több oldal egyszerre)
- Skálázott, a TS3-kalkulátor 3D nézetéhez hasonló stílusban

---

## 8. Auto-figyelmeztetések

A kalkulátor automatikusan generálja a következő figyelmeztetéseket:

| Trigger | Üzenet (példa) |
|---|---|
| `viewingDistance < pitch × 1.0` | ⚠ A nézési távolság (X m) kisebb az optimális minimumnál (P3.9 → 3,9 m). |
| `viewingDistance < pitch × 0.5` | 🔴 A nézési távolság (X m) a megengedhető minimum alatt (P3.9 → 1,95 m). Erősen ajánlott P2.5-re cserélni. |
| `pitch === 'P2.5' && installType === 'prism'` | 🔴 **TILTVA** — A P2.5 készletben nincs SC kabinet, és a max ív a fokolás-tartomány (±7,5° / +10°). Prism (hasáb/kocka, 90°-os sarok) megvalósíthatatlan P2.5-tel. Válaszd a P3.9-et a prism módhoz, vagy maradj curved módban P2.5-tel. |
| `installType === 'curved' && curveAngle > 10 × cols` | ⚠ A megadott összhajlítás-szög meghaladja az `±7,5° × kabinet-szám` curve-lock-limitet. Csökkents szögön vagy fontold meg több, kisebb szegmensre osztást. |
| `total_W_max × 1.25 > 46 kW (3 × 16A)` | 🔴 A számított max. áramfelvétel 3 fázist (32A vagy 63A) igényel. Kérlek vegyél fel külön áramellátási tervet. |
| `totalPixels > 3.9M && procModel === 'KU20'` | (info) ℹ A pixel-szám > 3,9 M → automatikusan MX30 processzort használ. |
| `totalPixels > 6.5M` | ⚠ A pixel-szám > 6,5 M, 2× MX30 kaszkádolt processzor szükséges. |
| `totalPixels > 13M` | 🔴 A pixel-szám > 13 M, ez kívül esik a v1.0 scope-ján. Beni-konzultáció. |
| `installType === 'flown' && total_kg / cols > 200` | ⚠ Egy oszlop terhe > 200 kg — kérlek ellenőrizd a truss-teherbírást. |
| `pitch === 'P3.9' && brightness === 'outdoor' && installType === 'totem'` | ⚠ Kültéri totem 4500 niten napsütésben is jól látható, de kérlek ellenőrizd a totem szél-stabilitását (ballast). |
| `installType === 'prism' && (W_eff !== H_eff) && prismShape === 'cube'` | 🔴 Kockához W és H egyenlő kell legyen — javasolt méret: max(W,H) × max(W,H). |

---

## 9. Export

### 9.1. CSV-export (UTF-8 BOM)

```
\uFEFF"csop","sku","name","qty","uom","kg","m3","note"
"LED-kabinetek","ABSEN-PL3.9W-PLUSV2-500x500-STANDARD","Absen PL3.9W Plus V2 500x500",36,"db",288.0,0.72,""
"Processzor + adatlánc","NOVASTAR-KU20","NovaStar KU20 LED video controller",1,"db",2.1,0.005,""
"Processzor + adatlánc","NOVASTAR-CVT10-S","NovaStar CVT10-S Fiber Converter",1,"db",2.1,0.005,""
...
```

### 9.2. Markdown-export (Odoo-import-kompatibilis BoM-formátum)

```markdown
# BoM — 6×3 m P3.9 kültéri stack | Generálva 2026-05-11 16:23

## Konfiguráció
- Méret: 6,0 × 3,0 m (12 × 6 = 72 kabinet)
- Felbontás: 1536 × 768 pixel
- Pitch: P3.9 (kültéri firmware konfig, 4500 nit)
- Típus: földi stack

## Tömeg- és térfogat-összesítés
- Össztömeg: 1083 kg
- Össztérfogat: 3,1 m³
- Áramfelvétel max: 10 800 W (kültéri) — 4 db 16A áramkör

## ⚠ Helyszíni operatív teendő
- **Firmware-konfig feltöltés** a fal összeállása után: KÜLTÉRI mód (4500 nit)

## BoM (Odoo-import-kompatibilis)

### LED-kabinetek
| SKU | Név | Db | UoM | kg | m³ |
|---|---|---|---|---|---|
| ABSEN-PL3.9W-PLUSV2-500x500-STANDARD | Absen PL3.9W Plus V2 500×500 | 36 | db | 288,0 | 0,72 |
| ABSEN-DISPLAY-P3.9-500x1000-OUTDOOR-7680HZ | Absen PL3.9W Plus V2 500×1000 | 18 | db | 234,0 | 0,72 |

### Processzor + adatlánc
| SKU | Név | Db | UoM | kg | m³ |
|---|---|---|---|---|---|
| NOVASTAR-KU20 | NovaStar KU20 LED video controller (1,18M pixel ≤ 3,9M) | 1 | db | 2,1 | 0,005 |
| NOVASTAR-CVT10-S | NovaStar CVT10-S Fiber Converter | 1 | db | 2,1 | 0,005 |
| FIBER-LC-LC-DUPLEX-50M | LC-LC duplex fiber kábel 50m | 2 | db | 1,6 | 0,004 |
| CAT6-SFTP-5M | CAT6 SFTP 5m | 2 | db | 0,6 | 0,002 |

### Stack szerkezet
| SKU | Név | Db | UoM | kg | m³ |
|---|---|---|---|---|---|
| ABSEN-STACK-VAS-FIX-1M | Absen alsó sztekkvas fix 1m (id 5364, code 2682) | 6 | db | 68,4 | 0,168 |
| ABSEN-STACK-FRONT-LEG | Absen alsó sztekkvas első kitámasztó (id 5365, code 2683) | 6 | db | 3,0 | 0,002 |
| ABSEN-CABINET-SUPPORT-BRACE | Absen kabinet kitámasztó (id 5441, code 2759) | 6 | db | 30,0 | 0,060 |

...

### Kiegészítők és helyszíni operatív
| SKU | Név | Db | UoM | Megjegyzés |
|---|---|---|---|---|
| (info) | Firmware-konfig: KÜLTÉRI mód (4500 nit) feltöltése a fal összeállása után | 1 | feladat | helyszíni technikus |
| BETEKINTO-MONITOR-DELL | Betekintő monitor 27" Dell | 1 | db | |
```

### 9.3. Vágólap másolás

Két szintű fallback:
1. Modern API: `navigator.clipboard.writeText(text)` — async, https-en működik
2. Régi fallback: `textarea` + `document.execCommand('copy')`

---

## 10. Validáció

### 10.1. `validate(inputs)`

```javascript
function validate(inputs) {
  const errs = [];
  const warn = [];
  if (inputs.W < 0.5 || inputs.W > 30) errs.push('W tartomány: 0,5 - 30 m');
  if (inputs.H < 0.5 || inputs.H > 10) errs.push('H tartomány: 0,5 - 10 m');
  if (inputs.installType === 'curved' && inputs.curveAngle > inputs.cols_estimate * 10) {
    warn.push('A curveAngle meghaladja a curve-lock-limitet (kabinet-szám × 10°)');
  }
  // Szabály 7 (Sarbó 2026-05-11 v1.5): P2.5 prism mód KATEGORIKUSAN TILTVA
  // P2.5-ből nincs SC variáns; max ívet engedi a fokolása (±7,5° / +10°), de prism (90°-os sarok) NEM
  if (inputs.installType === 'prism' && inputs.pitch === 'P2.5') {
    errs.push('P2.5 pitch + prism mód KATEGORIKUSAN TILTVA — P2.5-ből nincs SC variáns Rentmanben, a max ív a fokolás-tartomány (±7,5° / +10°). 90°-os prism-sarok megvalósíthatatlan. Válaszd a P3.9-et a prism módhoz, vagy a P2.5-tel maradj curved módban.');
  }
  if (inputs.installType === 'prism' && inputs.prismShape === 'cube' && inputs.W !== inputs.H) {
    errs.push('Kockához W és H egyenlő kell legyen');
  }
  // ... lásd 8. szekció
  return {errs, warn};
}
```

---

## 11. Példa-konfigurációk

### Példa A — 6×3 m P3.9 kültéri stack

**Bemenet:** W=6, H=3, pitch=P3.9, installType=ground-stack, brightness=outdoor, cabinetPref=auto-1000-prefer, fiberLength=50

**Várt layout:**
- cols=12, rows=6, total=72, sideCount=1
- p500x1000=36 (12 × 3 pár), p500x500=0
- px: 1536 × 768 = 1 179 648 pixel
- Max W (kültéri): 36 × 300 × 1.0 = 10 800 W → 4 × 16A áramkör

**Várt BoM (kivonat):**
- 36× ABSEN-DISPLAY-P3.9-500x1000-OUTDOOR-7680HZ (Rentman 2122)
- 1× NOVASTAR-KU20-OPTIKA-6PORT-AD-KIT (Rentman code 2678) — 1,18 M pixel ≤ 3,9 M → KU20-kit
- 6× ABSEN-STACK-VAS-FIX-1M (Rentman 2682, 6 m W → 6 db 1m)
- 6× ABSEN-STACK-FRONT-LEG (Rentman 2683)
- 6× ABSEN-CABINET-SUPPORT-BRACE (Rentman 2759, H = 3 m ≥ 1,5 m)
- 9× PL3.9-FC-4IN1 (ceil(36/4))
- 4× CEE-16A-ELOSZTO-5x + 4× FI-RELE-30MA
- 4× POWERCON-TRUE1-NOVASTAR-20M (Rentman 3224, 1 db / áramkör)
- ⚠ info-sor: KÜLTÉRI firmware konfig (4500 nit) a fal összeállása után

### Példa B — 4×2,5 m P2.5 beltéri stack

**Bemenet:** W=4, H=2.5, pitch=P2.5, installType=ground-stack

**Várt layout:**
- cols=8, rows=5, total=40
- p500x500=40
- px: 1600 × 1000 = 1 600 000 pixel
- Max W: 40 × 160 × 1.0 = 6 400 W → 2 × 16A áramkör

**Várt BoM (kivonat):**
- 40× ABSEN-PL2.5-PLUS-V2-500x500-STANDARD (Rentman 2784)
- 1× NOVASTAR-KU20-OPTIKA-6PORT-AD-KIT (Rentman 2678, 1,6 M pixel ≤ 3,9 M, de 5 port szükséges 6-ból)
- 4× ABSEN-STACK-VAS-FIX-1M (Rentman 2682, 4 m W → 4 db 1m)
- 4× ABSEN-STACK-FRONT-LEG (Rentman 2683)
- 4× ABSEN-CABINET-SUPPORT-BRACE (Rentman 2759, H = 2,5 m ≥ 1,5 m)
- 7× ABSEN-FC-PL2.5-PLUS-V2-6IN1-PLASTIC (ceil(40/6))
- 2× CEE-16A-ELOSZTO-5x + 2× FI-RELE-30MA
- 2× POWERCON-TRUE1-NOVASTAR-20M

### Példa C — 5×3 m P3.9 függesztett

**Bemenet:** W=5, H=3, pitch=P3.9, installType=flown, brightness=outdoor

**Várt:**
- cols=10, rows=6, total=60
- p500x1000=30
- 5× ABSEN-RIGVAS-1.0M (Rentman 2482, 5 m W → 5 db 1 m)
- 7× SAFETY-STEEL-1.5M (5 rigvas pont + 2 backup)
- 1× NOVASTAR-KU20-OPTIKA-6PORT-AD-KIT (Rentman 2678, 1,18 M pixel ≤ 3,9 M)
- 8× PL3.9-FC-4IN1 (ceil(30/4))
- 4× POWERCON-TRUE1-NOVASTAR-20M

### Példa D — 4×2,5 m P3.9 hajlított (R=2 m, 90°)

**Bemenet:** W=4 (ívhossz), H=2.5, pitch=P3.9, installType=curved, curveR=2, curveAngle=90

**Várt:**
- Ívhossz = π × R × szög/180 = π × 2 × 0,5 ≈ 3,14 m (közelítve W=4-re a felfelé kerekítéssel)
- cols=8, rows=5, total=40
- p500x500=40 vagy mix (NORMÁL kabinetek a curve-lock-kal)
- 90°/8 = 11,25° per csatlakozás → ⚠ figyelmeztetés: meghaladja a 10°-os limitet → vagy több, kisebb szegmensre vagy SC-vel készíteni (prism mód)
- Top view: 90°-os ív, R=2 m

### Példa E — 2×2×1,5 m P3.9 háromoldalas hasáb (prism)

**Bemenet:** W=2, H=1.5, pitch=P3.9, installType=prism, prismShape=triangle-3side, brightness=outdoor

**Várt layout:**
- cols=4, rows=3, total_per_side = 12, sideCount=3, total=36
- pSC=36 (mind SC kabinet)
- px (per oldal): 512 × 384
- Top view: szabályos háromszög 3 oldallal 2 m × 1,5 m

**Várt BoM (kivonat):**
- 36× ABSEN-PL3.9W-PLUSV2-500x500-CORNER (Rentman 2124, SC)
- 1× NOVASTAR-KU20-OPTIKA-6PORT-AD-KIT (Rentman 2678, 36 × 16K = 590K pixel ≤ 3,9 M)
- Stack: perimeter = 3 × 2 m = 6 m → 6× ABSEN-STACK-VAS-FIX-1M (Rentman 2682) + 6× ABSEN-STACK-FRONT-LEG (Rentman 2683) + 6× ABSEN-CABINET-SUPPORT-BRACE (Rentman 2759, H = 1,5 m → kabinet support kötelező)
- 6× ABSEN-FC-PL2.5-PLUS-V2-6IN1-PLASTIC (Rentman 5475, ceil(36/6))
- 1× POWERCON-TRUE1-NOVASTAR-20M

Ez összevethető a Rentman „15nm led hasáb P3.9 kültéri | 1.5 x 2.5 x 4 | 384 x 640 x4 | AD kit" (id 5424) konfigurációval.

---

## 12. Implementációs útmutató

### 12.1. Egyetlen monolit HTML-fájl (v1.0)

A TS3-kalkulátor v1.0 mintáját követi:

```
<!DOCTYPE html>
<html lang="hu">
<head>
  <meta charset="UTF-8">
  <title>Absen LED-fal Konfigurátor</title>
  <style>/* 7.2 SVG-paletta + általános CSS (TS3-ról adaptálva) */</style>
</head>
<body>
  <div class="layout">
    <aside class="form-panel">
      <h2>Konfiguráció</h2>
      <form id="cfg-form" onchange="render()"><!-- 3. szekció UI --></form>
    </aside>
    <main class="output">
      <div class="summary"><!-- össztömeg, össztérfogat, áram, pixel, warnings --></div>
      <div class="views">
        <div id="view-front"></div>
        <div id="view-side"></div>
        <div id="view-top"></div>
        <div id="view-iso"></div>
      </div>
      <div class="bom-table-wrapper">
        <table class="bom-table"><!-- BoM tételek --></table>
      </div>
      <div class="actions">
        <button onclick="exportCSV()">CSV-export</button>
        <button onclick="exportMD()">Markdown-export</button>
        <button onclick="copyMD()">Markdown vágólapra</button>
      </div>
    </main>
  </div>
  <script>
    /* 6. szekció massVolMap */
    /* 4. szekció algoritmusok */
    /* 5. szekció build_bom_grouped */
    /* 7. szekció draw_frontview, draw_sideview, draw_topview, draw_iso3d */
    /* 9. szekció export funkciók */
    /* 10. szekció validate */

    function render() {
      const inputs = readForm();
      const v = validate(inputs);
      const layout = compute_panel_layout(inputs.W, inputs.H, inputs.pitch, inputs.cabinetPref, inputs.installType, inputs.prismShape);
      const power = compute_power(layout, inputs.pitch, inputs.brightness);
      const data = compute_data(layout, inputs.pitch);
      const rigging = compute_rigging(layout, inputs.installType);
      const stack = compute_stack(layout, inputs.installType);
      const cases = compute_cases(layout, inputs.pitch);
      const optical = compute_optical_cables(data, inputs.fiberLength);
      const bom = build_bom_grouped(layout, power, data, rigging, stack, cases, optical, inputs.brightness);
      // ... render-BoM-tábla, 4 nézet, summary
    }

    document.addEventListener('DOMContentLoaded', render);
  </script>
</body>
</html>
```

### 12.2. Repó-struktúra (új `adam-led-szamolo`)

```
adam-led-szamolo/
├── README.md
├── CHANGELOG.md
├── legacy-html/
│   ├── Absen_LED_Konfigurator_v1_0.html       ← v1.0 első működő
│   ├── Absen_LED_Konfigurator_v1_1.html       ← későbbi iterációk
│   └── CHANGELOG.md
└── docs/
    └── 04_KEZIK_Absen_LED_Konfigurator_Spec_v1_1.md   ← ez a doksi
```

### 12.3. Tesztelési protokoll

A v1.0 működőképesség **manuális tesztelés** mind az 5 példa-konfigurációra (11. szekció):
1. Form-bemenetek beállítása
2. BoM-tábla összehasonlítása a várt darabszámokkal
3. SVG-nézetek vizuális ellenőrzése (különösen prism módban a 4 nézet konzisztenciája)
4. CSV-export Odoo-import-kompatibilitás (UTF-8 BOM + idézőjelek)
5. Markdown-export vágólapra mentés + helyszíni firmware-konfig info-sor ellenőrzése

### 12.4. Modulárisra bontás (későbbi v1.1+)

A monolit struktúra **gyors prototípusra ideális**, de **karbantartásra nehéz**. v1.1+ refaktor candidate:
- `algorithms.js` (4. szekció)
- `bom.js` (5. szekció)
- `views.js` (7. szekció)
- `export.js` (9. szekció)
- `validate.js` (10. szekció)

---

## 13. Nyitott pontok / v1.3 backlog

Az alábbiak **NEM blokkolják** a v1.2-t, de Beni-konzultációval pontosítandók:

### 13.1. Adathiány Rentmanben / Odoo-ban

1. **PL2.5 Plus V2 500×500 (Rentman 5474):** `power=0`, `current=0`, `volume=0` — pótlandó (datasheet alapján: 160 W max, 0,7 A, 0,02 m³)
2. **PL3.9W Plus V2 500×500 + 500×500 SC Odoo-rekord:** csak a 500×1000 (Odoo 920) van — a két másik kabinet **Odoo-import** szükséges (új SKU-k: `ABSEN-PL3.9W-PLUSV2-500x500-STANDARD` és `-CORNER`)
3. **Stack szerkezet (5364, 5365, 5441):** `volume=0` mind a háromnál (az 5473-as 0,5m fokolható már OK 0,01 m³-rel) — pótlandó tényleges térfogat-méréssel
4. **Rigvas tételek (4622, 5154):** `volume=0` — pótlandó
5. **NovaStar standalone tételek (KU20, MX30, CVT10-S):** `power=0`/`volume=0` egyes mezőknél — datasheet-pótlás (KU20=25 W, MX30=55 W, CVT10-S=22 W)
6. **Virtual Package-ek (2668/2678/2857) `tags: "elveszett"`:** Sarbó-megerősítés (2026-05-11): nem fizikai elvesztés, hanem hibás importnál veszett el az adat. A rendszer-szintű átdolgozás a hosszú távú megoldás (virtuális kit-ben csak fizikai tartalom — Beni). Addig a kalkulátor a fizikai komponens-listát (vagy a 4607/4608 fizikai AD KIT-et) adja.
7. **4602 (Absen SC konténer) description-hiba:** "Absen PL3.9W (0,5x0,5) üres konténer"-t mond a description, de a name szerint **PL2.5 Plus V2 SC**-hez tartozik. Beni-feladat: description + internal_remark javítása.
8. **5517 (10G SFP module) custom_16/19-hiba:** "Colorlight H10FIX optikai kifejtő"-t mond, pedig Novastar SFP. Beni-feladat: custom mezők javítása.
9. **5664 (TRUE1 Novastar 1m) description-hiba:** A description és internal_remark "20m"-et mond, de a length már 100 cm (Sarbó-javítás). A description / internal_remark mezők még javítandók "1m"-re.
10. **Kabinet-szintű AD KIT-ek (5476, 4601, 4596, 4603) `volume=0`:** A kit mérete 110×75×50 = 0,4125 m³ a 4596-ban már megvan, de a többiben hiányzik. Beni-feladat: pótlás.
11. **Totem-kit-ek (5548, 5745) `weight=0, power=0`:** Virtual Package-ek, a komponens-tartalom súly-aggregálása nem fut le. A Beni-átdolgozás során pótlandó.
12. **4594 (Absen kábeles táska kicsi) folder-mapping hiba (v1.7):** A rekord `folder: /folders/310` (Absen P3.9) alatt van, pedig **a többi kábel-család (4588-4593, 4595) folder 314 (Absen)** alatt. Logikusan a folder 314 lenne a helyes (az általános Absen-kábel-szekcióban). Beni-feladat: folder-átsorolás (a 4595 nagy táska mellé).
13. **5155, 5156 rigvas konténerek `volume=0` (v1.7):** Mindkét konténer (nagy 40 kg, kicsi 42 kg) `volume=0`-val van regisztrálva, pedig 700×1140×530 mm = ~0,42 m³ a fizikai befoglaló méret. Beni-feladat: pótlás.
14. **5699, 5700 powerCON TRUE1 új tételek (v1.7) `weight=0`, `volume=0`:** A 2m bekötő és 6m átkötő alapadatai hiányosak — datasheet-pótlás (a 2m ~0,30 kg, a 6m ~0,90 kg becsléssel). Beni-feladat: pótlás.
15. **4623, 4624, 4625, 4626 csavarok és kabinet-zár (v1.7) `weight` részleges:** Az M8 csavaroknak (4623, 4624) hiányzik a súlya (datasheet: M8x16 ~0,005 kg, M8x60 ~0,030 kg). A 4625 kabinet-zár megvan (0,12 kg). A 4626 pillangós csavar AD kit megvan (6 kg). Beni-feladat: csavar-súly pótlás.

### 13.2. Szabály-tisztázások — Sarbó-féle döntések (Sarbó 2026-05-11 v1.5)

> **Minden 7 szabály EL VAN DÖNTVE** — a v1.5 ezeket implementálja, a v1.6 a fizikai AD KIT default-tal pontosít.

1. ✅ **Cabinet support brace határ:** **`H ≥ 2,0 m`** (NEM 1,5 m). Alatti magasságon fizikailag nem fér el — építési korlát.
2. ✅ **Front leg darabszám:** **N−1** (csak belső kabinet-kabinet illesztések, a két szélső véget a sztekkvas-rúd vége tartja). NEM N+1.
3. ✅ **Rigvas-AD-kit küszöb:** **≥ 8 db 1m rigvas** → 1 kit-konténer (jelenlegi default megerősítve).
4. ✅ **Kábeles táska tartalom-decode:** v1.6-ban verifikálva mind a 6 Rentman-rekorddal (4588-4593). Részletek a 2.11.1-ben.
5. ✅ **NovaStar AD kit (Virtual Package) NEM default:** A Virtual Package-ek `tags: "elveszett"`. **v1.6 default: fizikai 4608 MX30 AD KIT + standalone CVT10-S + Rack 2U** (az MX30 AD KIT-ben már bent van 2 SFP, +2 extra SFP a CVT10-ekhez).
6. ✅ **Redundáns konfig + 50Hz/8bit:** A processzor-választás **mindig úgy menjen**, hogy redundánsan, 50 Hz-en, 8 bit-en meg tudja hajtani. A default 1× MX30 AD KIT + 2× CVT10 már redundáns. Nagyobb fal: 2× MX30 AD KIT + 4× CVT10.
7. ✅ **P2.5 prism mód KATEGORIKUSAN TILTVA:** P2.5-ből nincs SC, a fokolás max ±7,5°/+10° — 90°-os prism-sarok megvalósíthatatlan. A validate-fv hibalistába teszi, a compute_layout exception-t dob.

### 13.3. v1.8+ funkciók (még nyitva)

1. **Floor tile screen** mód (datasheet említi a PL3.9W Plus V2 500×500 kompakt kabineteknél, padlóra fektethető PC-borítással) — v1.8+ ?
2. **Több különálló LED-fal egy projekten** (fő + 2 oldalsó) — egy konfig-fájlba mentés, kombinált BoM-export
3. **Multi-segment fal-konfig** — a `5700/3260` 6m TRUE1 átkötő és a multi-feed setup automatizálása több szegmensű falaknál
4. **Truss-méretezés** — automatikus választás (12-es vagy 30-as truss) súlyhatár alapján
5. **Vegyes pitch** (P3.9 főfal + P2.5 fókusz) — multi-konfig egy fájlban (jelenleg: kizárt, ld. 7.c döntés)
6. **CVT10 KIT (5349) automata-választás:** mikor (mennyi CVT10 felett) ajánljuk a 3-darabos KIT-et a 3× standalone helyett? (Jelenlegi v1.6: ha cvtCount ≥ 3.)
7. **Kabinet-szintű AD KIT alternative (4596/4601/4603/5476):** mikor használjuk a 6/4-kabinet kit-eket vs stand-alone kabinet + üres konténer kombóját? (Jelenlegi v1.6: ha a fal cabinet-száma a 6/4 többszöröse, opcionálisan kit-szintű.)
8. **LBC + Qiangli + GKGD ledfal-konfigurátor (Colorlight vezérléssel) — KÜLÖN spec/repó (v1.7 új):** A jelenlegi v1.x kifejezetten Absen+Novastar rendszerre vonatkozik. A párhuzamos LBC+Qiangli+GKGD rendszerre (640×640 / 640×480 kabinetek, Colorlight X16/X2S/X20/X4e/A100/A200/C3 Pro vezérlés) később külön konfigurátor készül. Lásd 2.15. szekció a két rendszer szétválasztásáról.
9. **Prizma pre-build kit auto-detect (v1.7 új):** Ha `installType: prism` és a méret pontosan 0,5+1+0,5 m alaplap × 1m vagy 2m magas, automatikusan ajánljuk az 5784 (kis hasáb) vagy 5931 (nagy hasáb) Virtual Package-et a dinamikus számítás helyett.

> **Megoldott pontok (v1.1 → v1.7-ben):**
> - ✅ A NovaStar tételek **léteznek** Rentmanben (12 rekord) [v1.2]
> - ✅ PL3.9 500×500 hordláda **= PL2.5 6-az-1-ben hordláda** [v1.2]
> - ✅ 10G SFP module, Rack 2U Double Door, powerCON TRUE1 → hozzáadva [v1.2]
> - ✅ **0,5 m sztekkvas (fokolható) Rentman ID:** 5473/2783, 8,6 kg [v1.4]
> - ✅ **Absen kábeles táska kicsi (8 cabinet) Rentman ID:** 4594/2152 [v1.4]
> - ✅ **Absen kábeles táska nagy (12 cabinet) Rentman ID:** 4595/2153 [v1.4]
> - ✅ **Kábeles táska `type: case`** — táska + kit egy tétel [v1.4]
> - ✅ **PL2.5 Plus V2 áramfelvétel + szivárgóáram** (160W/54W/0,7mA) [v1.5]
> - ✅ **Szivárgóáram-szabály:** `MAX(by_power, by_leakage)` [v1.5]
> - ✅ **`tags: "elveszett"`** = hibás import, NEM fizikai elvesztés [v1.5]
> - ✅ **Stack szabályok:** H≥2m back brace, N−1 front leg [v1.5]
> - ✅ **Processzor-lánc fizikai komponensek** (NEM Virtual Package) [v1.5]
> - ✅ **Redundáns 50Hz/8bit alap** [v1.5]
> - ✅ **P2.5 prism kategorikus tiltás** [v1.5]
> - ✅ **TRUE1 bekötő SZÉTVÁLASZTÁSA:** 4591/2149 (Absen 20m main feed) vs 5664/3224 (Novastar 1m CVT10-tetőszerelés) [v1.6]
> - ✅ **Új Absen-kábel család felvéve** (4588-4593, 6 tétel) [v1.6]
> - ✅ **Új Absen betápdoboz felvéve** (5718/3278, 32A→6×16A kit) [v1.6]
> - ✅ **Új SC konténer felvéve** (4602/2160) [v1.6]
> - ✅ **Kabinet-szintű AD KIT-ek alternatívaként** (4596, 4601, 4603, 5476) [v1.6]
> - ✅ **Fizikai AD KIT default:** 4608 MX30 10E+2S (NEM stand-alone MX30) [v1.6]
> - ✅ **CVT10 KIT (5349) alternatíva 3+ CVT10-re** [v1.6]
> - ✅ **Totem-kit-ek (5548, 5745) referenciaként** [v1.6]
> - ✅ **Komplett Rentman folder-bejárás** (2394 tétel) → 10 új Absen-rekord [v1.7]
> - ✅ **Rigvas-konténerek** (5155 nagy 40 kg, 5156 kicsi 42 kg) felvéve a 2.5-be [v1.7]
> - ✅ **powerCON TRUE1 család 6 tagra bővítve** — új 5699 (2m bekötő, rövid main feed) és 5700 (6m átkötő, szakaszközi multi-feed) [v1.7]
> - ✅ **Kabinet-összekötő elemek új szekció (2.13)** — M8x16 (4624), M8x60 (4623), kabinet-zár (4625), pillangós AD kit 5×3 (4626) [v1.7]
> - ✅ **Prizma pre-build kit-ek új szekció (2.14)** — 5784 (kis hasáb 1m), 5931 (nagy hasáb 2m), csak P3.9 SC [v1.7]
> - ✅ **Két párhuzamos LED-rendszer (2.15)** explicit dokumentálva — Absen+Novastar (jelen spec) vs LBC+Qiangli+GKGD+Colorlight (külön spec) [v1.7]
> - ✅ **5517 SFP custom_16/19 "Colorlight H10FIX" tisztázás** — NEM HIBA, csak régi cimke (két rendszer parallel fut a folder 64-ben) [v1.7]

---

## 14. Verziótörténet

| Verzió | Dátum | Változás | Szerző |
|---|---|---|---|
| **1.0** | **2026-05-11 (du.)** | **Első kiadás.** TS3 színpadkalkulátor mintáját (`04_KEZIK_Tomkostage_Konfigurator_Spec_v1_0.md`) követve. Tartalmazta a Colorlight X20 processzort + H10FIX optikát + generikus LED stack vas családot + Odoo PL hanging bar SKU-kat. | Sarbó Gergely + Claude |
| **1.1** | **2026-05-11 (este)** | **Sarbó-féle 5 nagy korrekció:** **(1) Stack-rendszer:** generikus LED stack vas család **kihúzva** → Absen-specifikus 3 tétel (Rentman 5364/2682, 5365/2683, 5441/2759). **(2) Rigging:** Odoo hanging bar SKU-k **kihúzva** → Rentman rigvas család (4622/2180 0,5m, 5154/2482 1m, 5157/2485 AD kit). **(3) Processzor:** Colorlight X20 **kihúzva** → NovaStar KU20 (≤ 3,9 M pixel) + MX30 (≤ 6,5 M pixel), két-lépcsős választás. **(4) Optika:** Colorlight H10FIX **kihúzva** → NovaStar CVT10-S (10 Ethernet + 2 optika, hot-swap SFP). **(5) SC szemantika:** SC NEM hajlított falra → **`prism`** (hasáb/kocka 4-oldalú zárt formák) módra. Hajlított fal NORMÁL kabinetekkel a beépített ±7,5°/+10° curve-lock-kal. **(6) Fényerő-konfig:** beltéri/kültéri 1500/4500 nit = konfigurációs firmware feltöltés a helyszínen a fal összeállása után — BoM-on info-sorként, NEM külön termék. | Sarbó Gergely + Claude |
| **1.2** | **2026-05-11 (késő este)** | **Rentman ground truth pótlása + 2 nyitott pont megoldása:** **(1) NovaStar Rentman-rekordok** **mind a 12 megtalálva** (Sarbó-screenshot alapján, Rentman IDs 4605-5664): Novastar KU20 (4605/2163, 2,1 kg, 25 W), MX30 (4606/2164, 7,2 kg, 55 W), KU20 AD KIT (4607/2165), MX30 AD KIT (4608/2166), Novastar KU20 processzor + optika \| 6 port \| AD kit (5360/2678 — DEFAULT a ≤ 3,9 M pixel-falakra), Novastar MX30 + optika \| 10 port \| AD kit (5350/2668 — DEFAULT a ≤ 6,5 M-re), Novastar MX30 + 20 port redundáns AD konfiguráció (5555/2857 — redundáns / nagy fal), CVT10-S (5348/2666), CVT10 KIT (5349/2667), 10G SFP module SM (5517/2819), Novastar Rack 2U Double Door (5553/2855), powerCON TRUE1 bekötő (5664/3224, 20 m). **(2) PL3.9 500×500 hordláda** = **PL2.5 6-az-1-ben hordláda** (Sarbó-megerősítés, közös fizikai csomagolás). **(3) `compute_data` algoritmus átírva** a kit-választás logikára (Virtual Package code-kal). **(4) `massVolMap`** kiegészítve Rentman code + ID + datasheet-power mezőkkel. **(5) Példa A-C-E BoM-kivonatok** frissítve a kit-kódokra (2678 / 2668) és a powerCON TRUE1-re. **(6) 13. nyitott pontok** 2 elemmel csökkent (NovaStar Rentman rekord és PL3.9 hordláda — megoldva). | Sarbó Gergely + Claude |
| **1.3** | **2026-05-11 (éjszaka)** | **Stack-szabály finomítása (Sarbó 2026-05-11) + 4 új Rentman tétel placeholder-rel (ADAM MCP disconnect):** **(1) 0,5 m sztekkvas** hozzáadva a 2.6 szekcióhoz — szögbe rakható (sarkok, fractional width), míg az 1 m csak egyenesen mehet. Rentman ID/code verifikálandó, a session-ben az ADAM MCP disconnect miatt nem volt lekérdezhető. **(2) Új sztekkvas-allokáció algoritmus:** kombinatorikus 0,5+1,0 m greedy (egyenes szakaszra 1 m preferált, sarkokra 0,5 m). **(3) Új front-leg / back-brace szabály** — **csatlakozási pontok** alapján: N kabinet → N+1 csatlakozási pont; minden ponthoz előre 1× első kitámasztó (5365/2683), minden második ponthoz 1× kabinet kitámasztó (5441/2759). A v1.2-es helytelen szabály (`frontSupportLegs = bottomStackBars`) javítva. **(4) Új 2.10 szekció: Absen kábeles készletek** — nagy (12 cabinet = 2× 6-az-1-ben konténer) és kis (8 cabinet = 2× 4-az-1-ben konténer), mindkettőhöz külön konténerrel. Rentman ID/code verifikálandó. **(5) `compute_stack`** algoritmus átírva (sztekkvas_1m + sztekkvas_05m + connectionPoints + frontSupportLegs + cabinetSupportBraces) + új `compute_cable_kits` függvény. **(6) `massVolMap`** kiegészítve a 4 új tétel placeholder-rel. **⚠️ ADAM MCP connector disconnect** a session során — 4 új tétel Rentman-verifikációja a következő szálra marad. | Sarbó Gergely + Claude |
| **1.4** | **2026-05-11 (késő éjszaka)** | **ADAM MCP reconnect → mind a 4 placeholder verifikálva:** **(1) Absen alsó sztekkvas fokolható 0,5 m** Rentman ID `5473`, code `2783`, súly 8,6 kg, méret 500×580×50 mm. A "fokolható" jelzőt a Rentman name megerősíti (= szögbe rakható). **(2) Absen kábeles táska \| kicsi \| 8/8/1+1/1 \| AD kit** Rentman ID `4594`, code `2152`, súly 11,95 kg, méret 300×590×300 mm — **8 cabinet** lefedés. **(3) Absen kábeles táska \| nagy \| 12/12/1+1/1 \| AD kit** Rentman ID `4595`, code `2153`, súly 13,01 kg, méret 300×670×300 mm — **12 cabinet** lefedés. **(4) Felfedezés:** mindkét kábeles táska `type: case` a Rentmanben, vagyis a táska és a kábel-kit egy rekord — **nincs külön konténer-tétel** (Sarbó: "mindkettő pont két konténerre elég" = a táska önmagában 2 hordláda kabinetjét szolgálja ki, NEM külön hordtáska kell). A 2.10. szekció és a massVolMap frissítve a verifikált adatokkal, a 4 placeholder eltávolítva. | Sarbó Gergely + Claude |
| **1.5** | **2026-05-12 (kora hajnal)** | **MIND a 7 Sarbó-féle szabály-tisztázás eldöntve és implementálva — a spec implementációra kész:** **(1) Szivárgóáram-szabály új** a `compute_power`-ben: `circuits_by_leakage = ceil(total_leakage_mA / 9)`, és `recommended_circuits = MAX(by_power, by_leakage)`. PL2.5 Plus V2 = 0,7 mA / panel (Sterling Event Group ground truth) — egy 30 mA RCD-re max ~12 panel a szivárgáskorlát miatt (NEM 18, ami az áramkorlát lenne). **(2) Stack `H ≥ 2,0 m` back brace** (NEM 1,5 m) — alatti magasságon fizikailag nem fér el. **(3) Front leg `N−1`** (NEM N+1) — csak belső kabinet-kabinet illesztések, a két szélső véget a sztekkvas-rúd vége tartja. A v1.4-es spec 6m × 3m falra 13 db legs-et adott; v1.5 helyesen 11 db. **(4) Kábeles táska tartalom-decode** screenshot ground truth Sarbó-tól: 8/8/1+1/1 = 8 power + 8 data + (1 main ether + 1 backup ether) + 1 power. **(5) NovaStar Virtual Package (2678/2668/2857) NEM default** — `tags: "elveszett"` jelentése: hibás importnál veszett el az adat, NEM fizikai elvesztés. A rendszer rendszerszintű átdolgozás alatt (virtuális kit-ben csak fizikai tartalom — Beni). Addig a default: **MX30 + 2× CVT10 + 4× SFP + Rack 2U fizikai komponensek** (≤ 6,5 M pixel); nagyobb fal: **2× MX30 + 4× CVT10 + 8× SFP + 2× Rack 2U**. **(6) Redundáns 50Hz/8bit alap** — minden default-választás redundáns (master + backup fiber-pair). KU20 kihúzva a default-ból (csak 1 fiber out, nem alkalmas redundánsra). **(7) P2.5 prism KATEGORIKUSAN TILTVA** — a validate-fv `errs` listába teszi (NEM warning), a compute_layout exception-t dob. P2.5-ből nincs SC, fokolás max ±7,5°/+10° — 90°-os prism-sarok megvalósíthatatlan. **(8) TRUE1 1m bekötő szerepe tisztázva** (Sarbó-javítás Rentmanben): a CVT10 a ledfal tetejére tehető, és az utolsó kabinet powerCON-out-jából táplálható, nem kell elosztót / hosszabbítót felvinni. A korábbi v1.4-es "hibás name" jelzés visszavonva. **(9) `tags: "elveszett"` magyarázat** beépítve a 13.1-be — hibás importnál veszett el az adat, nem fizikai elvesztés. **A spec mostantól implementációra kész** — csak adathiányok maradnak (PL2.5 power=0 Rentmanben, Odoo-rekord pótlás stb.). | Sarbó Gergely + Claude |
| **1.6** | **2026-05-12 (hajnal)** | **Teljes Rentman ledfal-családi bejárás (47 tétel) — kritikus felfedezések és pontosítások:** **(1) TRUE1 bekötő SZÉTVÁLASZTÁSA** — két különböző bekötő van: **4591/2149 Absen 20m main feed** (a kábeles táskában 1 db, elosztó→fal) vs **5664/3224 Novastar 1m** (CVT10 tetőszerelés, utolsó kabinet powerCON-out-jából). A v1.5-ig a két tétel össze volt keverve. **(2) Új Absen-kábel család** 6 fizikai tétellel (4588-4593) verifikálva mind a kábeles táska tartalom-decode-jához. A 2.10-es szekció kibővítve a komplett Rentman-listával: powerCON TRUE1 átkötő 1,2m (4588), 2m (4589), etherCON átkötő 1,1m (4590), powerCON TRUE1 bekötő 20m (4591), etherCON bekötő 30m (4592), Vention 20m (4593). **(3) Új Absen betápdoboz** (5718/3278) — 32A 3-fázisú bemenettel, 6×16A egyfázisú kimenettel, 26,7 kg. Kit-szintű elosztó-doboz: 1 doboz = 6 áramkör (alternatíva az itemized CEE 16A + RCD listához). 2.8.1. új alszekcióban. **(4) Új SC variánsú konténer** (4602/2160) hozzáadva a 2.7-be — SC-kabinetek külön hordlád-rekordja (a sima 5475-tel párhuzamosan). Note: leírás-hiba a Rentman-ban (Beni-feladat). **(5) Új kabinet-szintű AD KIT-ek** alternatívaként a 2.7-be: 4596 (PL3.9W 0,5×1, 4 kabinet + 4-fakkos konténer), 4601 (PL3.9W 0,5×0,5, 6 kabinet + 6-az-1-ben konténer), 4603 (PL3.9W SC 0,5×0,5, 6 kabinet + SC konténer), 5476 (PL2.5 0,5×0,5, 6 kabinet + 6-az-1-ben). Power és súly adatok teljesek (74,3 kg / kit, 2880 W). **(6) Fizikai AD KIT default a compute_data-ban (NAGY átírás v1.6):** A v1.5 stand-alone MX30 + 2 CVT10 + 4 SFP + Rack-et javasolt. A v1.6 helyett **fizikai 4608 MX30 10E+2S AD KIT** (`type: case`, tartalmazza az MX30-at és a 2 SFP-t flight case-ben, 20,67 kg) + standalone CVT10-S (5348) + extra SFP (5517) + Rack 2U (5553). Egyszerűbb BoM (kevesebb sor, magától tartalmazza a flight case-t). **(7) CVT10 KIT (5349/2667) alternatíva** — ha ≥ 3 CVT10 kell egy konfighoz, az 5349 (14,39 kg, 3× CVT10 + táska) olcsóbb mint 3× standalone CVT10-S. Default: standalone, ajánlat-alternatíva ha cvtCount >= 3. **(8) Új totem-kit-ek referenciaként** (2.12. új szekció): 5548 (P2.5 400×800, Virtual Package) és 5745 (P2.5 400×1000, Virtual Package) — opcionális default a totem installType-ban, ha a fal pontosan ennek a méretnek a többszöröse. **(9) Absen P1.58 (4563)** referenciaként a 2.13-ba — finepitch kabinet, NEM ADAM-saját, a v1.6 default-okból kihagyva. **(10) Adatfeltöltési hibák naplózva** a 13.1-ben Beninek (4602 description, 5517 custom_16/19, 5664 description, 4601/4603/5476 volume=0, 5548/5745 weight+power=0). | Sarbó Gergely + Claude |
| **1.7** | **2026-05-12 (délelőtt)** | **TELJES Rentman LED-fal-szekció bejárása (5× 500-os batch, 2394 aktív tétel, MCP `rentman_equipment_bulk` paginálva) — 10 új Absen-rekord, kabinet-összekötő-elemek dokumentációja, és a két párhuzamos LED-rendszer (Absen+Novastar vs LBC+Qiangli+GKGD+Colorlight) explicit szétválasztása.** **(1) Rigvas-konténerek (2.5 bővítés):** Két új konténer-rekord, amelyek a meglévő AD kit-ek belső tárolódobozai: **5155/2483 (nagy konténer, 40 kg, 700×1140×530 mm)** = a `5157` 10× 1m rigvas AD kit belső konténere (113 kg teljes súly = 73 kg rigvas + 40 kg konténer, stimmel); **5156/2484 (kicsi konténer, 42 kg, 620×1140×560 mm)** = a 0,5m rigvasokhoz. Önállóan is rendelhetők extra-tárolásra. **(2) powerCON TRUE1 család 6 tagra bővítve (2.9 + 2.10):** Két új TRUE1 kábel a folder 314-ben: **5699/3259 (Absen 2m bekötő)** — rövid main feed extra, ha az elosztódoboz közvetlenül a fal mellett van; és **5700/3260 (Absen 6m átkötő)** — szakaszközi multi-feed átkötő (több szegmensű falaknál a szegmensek közötti tápátkötésre). Az új D mintázat: multi-segment setup `interSegmentFeeds_6m = segments - 1`. **(3) Új 2.13 szekció — Kabinet-összekötő elemek (csavarok + zárak + pillangós-csavar kit):** Négy új tétel a kabinetek mechanikai összefogásához: **4624/2182 (M8x16 Absen csavar)** rövid lábcsavar; **4623/2181 (M8x60 Absen csavar)** hosszú lábcsavar; **4625/2183 (Szimpla Absen LED kabinet összefogó 2 csavarhoz)** = kabinet-zár (LED cabinet lock single), 0,12 kg, 10×10×0,3 cm fémlemez; **4626/2184 (P3.9 Absen pillangós csavaros AD kit \| 5×3)** = pillangós csavar kit (LED cabinet lock set), 6 kg koffer, egy 5×3 (15 kabinet) falra elegendő. Darabszám-szabályok: `cabinetLocks = (N-1)*M + N*(M-1)`, `pillangosKits = ceil(N*M/15)`, `screwsM8x16 = N*M*4`. **(4) Új 2.14 szekció — Prizma pre-build kit-ek:** Két új P3.9 SC hasáb-kit Virtual Package-ként: **5784/3336 (3 oldalú hasáb 0,5/1/0,5×1, 512×256 px, 1m magas)** és **5931/3483 (3 oldalú hasáb 0,5/1/0,5×2, 512×512 px, 2m magas)**. CSAK P3.9 SC kabinetekkel, opcionális ajánlás `installType: prism` módban ha a méret pontosan egyezik. **(5) Új 2.15 szekció — Két párhuzamos LED-rendszer explicit dokumentációja:** Sarbó-tisztázás (2026-05-12) — az ADAM-nál **két fizikailag különálló LED-rendszer fut párhuzamosan**: (a) **Absen+Novastar** (jelen spec, 500×500/500×1000 kabinet, Novastar processzor, Absen rigvas/stack) és (b) **LBC+Qiangli+GKGD+Colorlight** (külön spec scope-on kívül, 640×640/640×480 kabinet, Colorlight X16/X2S/X20/X4e/A100/A200/C3 Pro processzor, LBC/QiangLi rigvas, LED stack vas 0.48/0.64/1.28/3.2m). A teljes komparatív táblázat 9 dimenzióval (kabinet, stack, rigvas, kábel, hordláda, vezérlés, kabinet-zár, pulpitus, scope). **(6) 5517 SFP custom mezők tisztázása:** A `custom_16/19` "Colorlight H10FIX optikai kifejtő" felirat **NEM HIBA**, csak régi sablon-szöveg-maradvány a Rentman-rekord-felvételkor — a Novastar SFP-t a Colorlight-rendszer name-jával címkézték (mindkét rendszer a folder 64 LED-szekcióban van). A korábbi v1.6 "Beni-feladat" jelzés visszavonva. **(7) 13.1 adathiányok bővítés:** +4 új tétel a Beni-naplóhoz: 4594 folder-mapping hiba (folder 310 helyett 314 kellene), 5155/5156 rigvas-konténerek volume=0, 5699/5700 új TRUE1 weight/volume hiányos, 4623/4624 csavarok weight hiányos. **(8) 13.3 v1.7+ funkciók átnevezve v1.8+-ra**, és kiegészítve: LBC+Qiangli külön konfigurátor (v1.8+), multi-segment fal-konfig automatizálása, prizma pre-build kit auto-detect. | Sarbó Gergely + Claude |

---

## 15. Kapcsolódó dokumentumok

- **Mintadokumentum:** `04_KEZIK_Tomkostage_Konfigurator_Spec_v1_0.md` (TS3-konfigurátor)
- **Tudástár — LED-konfigurációk Rentmanben:** `05_OPER_Rentman_Kit_Mintazatok_v2_0.md` 3.3. szekció (LED_KONFIG család, 180 kit) + 6.3. (E_AD_REUSE anomália a NOVA CVT10 ID-átírásra)
- **Küszöbszámok:** `07_INDEX_Kuszobszamok.md` (LED fényerő beltér/kültér, pixel pitch vs. nézési távolság, áramelosztás 5%/3%)
- **Reakcióidő:** `00_ADAM_Instructions_v2_7_FULL.md` 7. szekció
- **Operatív protokoll (load-in):** `05_OPER_Mester_Csekklista_v3_0.md` 11. fázis (LED indítóáram 10-100×, szakaszos bekapcsolás)
- **Datasheet források:**
  - Absen PL2.5 Plus V2: publitec.tv datasheet (2024-07)
  - Absen PL3.9W Plus V2: usabsen.com PL-V2-Spec-NA1 PDF (2024-12)
  - NovaStar KU20: novastar.tech, dvsledsystems.com, buynovastar.com
  - NovaStar MX30: oss.novastar.tech/uploads/2023/07/MX30 Specifications V1.0.1.pdf
  - NovaStar CVT10: oss.novastar.tech/uploads/2025/10/CVT10 Specifications V1.3.5.pdf

---

*Az ADAM tudástár élő rendszer — minden tisztázás egy új autoritatív forrást teremt.*
