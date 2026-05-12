# Absen Polaris V2 LED-fal Konfigurátor — CHANGELOG

> **Cél:** A monolit HTML-konfigurátor (v1.0+) verzió-történetének audit-tiszta tárolása. A `docs/` mappában a spec, a `legacy-html/` az aktuálisan üzemben lévő monolit forrás.
>
> **Tárolási konvenció (a TS3-mintára):**
> - Új patch-fájlt mindig a `legacy-html/` mappába tesszük (`Absen_LED_Konfigurator_v{NN_NN_N}{a-z}.html`)
> - Minden patch-hez itt 5–10 soros bejegyzés (mit javít, mi marad nyitva)
> - Régi verziókat NEM törlünk — a `recent_chats` és kontextus-megőrzés kedvéért
> - Build-artifactokat (PDF-export-példák, CSV-export-példák) NEM ide rakjuk — azok R2-attachment-be vagy Drive-ra mennek (lásd `00_ADAM_Instructions_v2_7_FULL.md` 8.8.7)

---

## v1.0 — *(létrehozandó)*

A v1.0 első működő HTML-implementáció a `docs/04_KEZIK_Absen_LED_Konfigurator_Spec_v1_7.md` alapján — egyetlen monolit HTML-fájl, ~120-160 kB, inline CSS+JS, CDN nélkül, offline működő. A teljes spec implementálva (15 fő szekció, 2.1-2.16 termék-katalógus, 8 fő algoritmus, 4 SVG-nézet, 5 példa-konfiguráció).

**A v1.0-tól érkező verziók ide kerülnek bejegyzésként** — mint a TS3 esetén v1.67.x-ig nyúló bejegyzés-lánc.

---

## Build és deployment

A v1.0+ sorozat **monolit HTML** — egyetlen önálló fájl, inline CSS+JS, CDN-mentes, offline futtatható. Build nincs, csak forrás-edit.

Használat:
1. Letöltés a legutolsó stable verzióból (kezdetben: `v1_0`)
2. Megnyitás bármilyen modern böngészővel
3. Form-bemenet (W × H × pitch × telepítési típus) → BoM + 4 nézet + CSV/Markdown-export

Régóta-tervezett refaktor: Beni Astro-projektje (testvér `adam-szinpad-szamolo/src/` mappa mintájára). Időzítés: nyitva — előbb a v1.x sorozat tesztelendő production-ben.
