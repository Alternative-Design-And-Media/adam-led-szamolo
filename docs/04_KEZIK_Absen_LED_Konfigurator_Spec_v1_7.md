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

## 3-15. szekciók — folytatás a következő commit-ban

A spec hátralévő része (form-bemenetek, 8 algoritmus, BoM-szerkezet, massVolMap, 4 SVG-nézet, auto-figyelmeztetések, CSV/Markdown export, validáció, 5 példa-konfiguráció, implementációs útmutató, 13.1-15 nyitott pontok és verziótörténet) a következő commit-ban kerül beillesztésre, terjedelmi okok miatt szétdarabolva.

A teljes lokális verzió `/home/claude/adam-led-szamolo/docs/04_KEZIK_Absen_LED_Konfigurator_Spec_v1_7.md`-ben elérhető (1572 sor, 108 KB).
