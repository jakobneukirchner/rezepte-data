# Rezept-Markdown-Schema — Vollständige Spezifikation v1.0

Dieses Dokument definiert das vollständige Format für alle Rezeptdateien in `jakobneukirchner/rezepte-data`.
Jede Abweichung führt zu Parse-Fehlern in der App.

---

## Datei- und Ordnerstruktur

```
rezepte-data/
├── rezepte/
│   ├── index.json                    ← Pflicht: Übersicht aller Rezepte
│   └── <rezept-id>/                  ← Ein Ordner pro Rezept
│       ├── rezept.md                 ← Pflicht: Das Rezept
│       ├── pic1.jpg                  ← Empfohlen: Hauptbild
│       ├── pic2.jpg                  ← Optional: weitere Bilder
│       └── ...
└── SCHEMA.md                         ← Diese Datei
```

**Rezept-ID-Regeln:**
- Kebab-case: Nur Kleinbuchstaben, Ziffern, Bindestriche
- Umlaute ersetzen: ä→ae, ö→oe, ü→ue, ß→ss
- Beispiel: „Omas Käsekuchen" → `omas-kaesekuchen`

---

## Aufbau von rezept.md

Jede Datei besteht aus exakt vier Abschnitten in dieser Reihenfolge:

```
[FRONT MATTER]
# Zutaten
# Bilder
# Zubereitung
```

---

## 1. Front Matter (YAML)

Zwischen zwei `---` Zeilen, immer ganz oben.

```yaml
---
title: <Titel als Freitext>
kategorie: <Pflicht, siehe Kategorien unten>
personen: <Ganzzahl>
zeit_gesamt: <Minuten gesamt inkl. Wartezeiten>
zeit_aktiv: <Minuten nur aktive Kochzeit>
schwierigkeit: <einfach | mittel | schwer>
bild: <pic1 | pic2 | ...>
tags: [<Tag1>, <Tag2>, ...]
---
```

**Erlaubte Kategorien:**
`Fleisch` | `Fisch` | `Vegetarisch` | `Vegan` | `Dessert` | `Backen` | `Suppe` | `Pasta` | `Salat` | `Sonstiges`

- `zeit_gesamt` = Gesamtzeit inkl. aller Wartezeiten (z.B. 12h Einweichen = 720 Min.)
- `zeit_aktiv` = Nur die Zeit, in der man aktiv am Herd steht
- `bild` = Referenz auf einen Eintrag aus dem `# Bilder`-Abschnitt (Hauptbild für Listenansicht)

---

## 2. Zutaten

```markdown
# Zutaten

- <Menge> <Einheit> <Name> (<optionale Anmerkung in Klammern>)
```

**Regeln:**
- Menge immer als Zahl, keine „ca." im Hauptfeld → in Anmerkung: `15 Stück Backpflaumen (ca., ohne Stein)`
- Normalisierte Einheiten: `g` | `kg` | `ml` | `l` | `EL` | `TL` | `Stück` | `Prise` | `Bund` | `Zehe`
- Ohne Einheit wenn sinnlos: `- Salz und Pfeffer nach Geschmack`
- Eine Zeile pro Zutat, nicht zusammenfassen

**Beispiele:**
```
- 800 g Schweinefilet
- 15 Stück Backpflaumen (ohne Stein)
- 6 EL Weinbrand
- 2 Stück säuerliche Äpfel
- Salz und Pfeffer nach Geschmack
- 3 EL Öl (zum Anbraten)
- 125 ml Brühe
- 2 EL Crème fraîche
- 2 EL Mehl (zum Binden)
- 50 ml kaltes Wasser
- 100 g durchwachsener Speck (sehr dünn geschnitten)
```

---

## 3. Bilder

```markdown
# Bilder

[pic1]: dateiname.jpg
[pic2]: dateiname2.jpg
```

**Regeln:**
- `[pic1]` ist immer das Hauptbild (muss mit `bild:` im Front Matter übereinstimmen)
- Dateinamen: lowercase, keine Leerzeichen, Endung `.jpg` oder `.png`
- Bilder liegen im **gleichen Ordner** wie `rezept.md`
- Maximal 9 Bilder (`[pic1]`–`[pic9]`)
- Abschnitt muss vorhanden sein, auch wenn keine Bilder existieren (dann leer lassen)

---

## 4. Zubereitung — Schritt-Format

Jeder Schritt steht auf **einer einzelnen Zeile**, immer in dieser Reihenfolge:

```
[STEPn] <Schritt-Text> [TIMER:XDXHXxM|keine] [PARALLEL:STEPn|keine] [BILD:picN|keine]
```

Alle vier Bestandteile (`[STEPn]`, Text, `[TIMER]`, `[PARALLEL]`, `[BILD]`) sind bei **jedem Schritt Pflicht**.

### Tag-Referenz

| Tag | Format | Beschreibung |
|---|---|---|
| `[STEPn]` | `[STEP1]`–`[STEP99]` | Schrittnummer, lückenlos beginnend bei 1 |
| `[TIMER:...]` | `[TIMER:0D12H00M]` oder `[TIMER:keine]` | Wartezeit in Tagen/Stunden/Minuten oder kein Timer |
| `[PARALLEL:...]` | `[PARALLEL:STEP3]` oder `[PARALLEL:keine]` | Gleichzeitig mit Schritt n möglich, oder keiner |
| `[BILD:...]` | `[BILD:pic1]` oder `[BILD:keine]` | Bild das bei diesem Schritt angezeigt wird |

### TIMER-Format

| Beispiel | Bedeutung |
|---|---|
| `[TIMER:0D12H00M]` | 12 Stunden |
| `[TIMER:0D00H35M]` | 35 Minuten |
| `[TIMER:0D00H08M]` | 8 Minuten |
| `[TIMER:1D00H00M]` | 1 Tag (24 Stunden) |
| `[TIMER:0D02H30M]` | 2 Stunden 30 Minuten |
| `[TIMER:keine]` | Kein Timer für diesen Schritt |

### PARALLEL-Regeln

- `[PARALLEL:STEP3]` → Dieser Schritt kann gleichzeitig mit STEP3 ausgeführt werden
- `[PARALLEL:keine]` → Schritt erfordert volle Aufmerksamkeit
- **Gegenseitig referenzieren:** Wenn STEP2 → `[PARALLEL:STEP3]`, dann muss STEP3 → `[PARALLEL:STEP2]`
- Nur ein Parallelschritt pro Schritt möglich

---

## Vollständiges Beispielrezept

Siehe [`rezepte/schweinefilet-mit-backpflaumen/rezept.md`](rezepte/schweinefilet-mit-backpflaumen/rezept.md)

---

## index.json Format

```json
{
  "version": "1.0",
  "rezepte": [
    {
      "id": "rezept-id-kebab-case",
      "title": "Titel des Rezepts",
      "kategorie": "Fleisch",
      "ordner": "rezept-id-kebab-case",
      "vorschaubild": "pic1.jpg",
      "zeit_gesamt": 90,
      "zeit_aktiv": 30,
      "schwierigkeit": "mittel",
      "personen": 4,
      "tags": ["Tag1", "Tag2"]
    }
  ]
}
```

Bei jedem neuen Rezept wird ein Objekt in das `"rezepte"`-Array eingefügt.
