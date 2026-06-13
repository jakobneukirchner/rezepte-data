# Google AI Studio Prompt — „Opas Küche" Android App

## Deine Aufgabe

Du bist ein erfahrener Android-Entwickler (Kotlin + Jetpack Compose). Erstelle eine **vollständige, produktionsreife Android-App** namens **„Opas Küche"** für einen älteren Nutzer, optimiert für Tablets.

**Wichtig:** Du baust ausschließlich die App. Die Rezeptdaten, das Datenformat und das Repository sind bereits vollständig vorbereitet unter `jakobneukirchner/rezepte-data`. Du liest nur — schreibst nie ins Repo.

---

## Tech Stack

- **Sprache:** Kotlin
- **UI:** Jetpack Compose (Material 3)
- **Architektur:** MVVM + Repository Pattern
- **Netzwerk:** Retrofit + OkHttp
- **Bilder:** Coil (`AsyncImage`), DiskCache 100 MB
- **Navigation:** Jetpack Navigation Compose
- **Timer:** `CountDownTimer` + `WorkManager` (Hintergrund-Notifications)
- **Build:** Gradle Kotlin DSL, `minSdk 26`, `targetSdk 35`, `libs.versions.toml`
- **Persistenz:** DataStore (Einkaufsliste + aktive Rezepte über App-Neustart hinweg)

---

## Datenzugriff (GitHub Raw API)

```
Rezeptliste:  https://raw.githubusercontent.com/jakobneukirchner/rezepte-data/main/rezepte/index.json
Rezept:       https://raw.githubusercontent.com/jakobneukirchner/rezepte-data/main/rezepte/{ordner}/rezept.md
Bild:         https://raw.githubusercontent.com/jakobneukirchner/rezepte-data/main/rezepte/{ordner}/{dateiname}
```

---

## Datenformat (exakt so parsen)

### index.json

```json
{
  "version": "1.0",
  "rezepte": [
    {
      "id": "schweinefilet-mit-backpflaumen",
      "title": "Schweinefilet mit Backpflaumen",
      "kategorie": "Fleisch",
      "ordner": "schweinefilet-mit-backpflaumen",
      "vorschaubild": "pic1.jpg",
      "zeit_gesamt": 780,
      "zeit_aktiv": 30,
      "schwierigkeit": "mittel",
      "personen": 4,
      "tags": ["Festlich", "Fleisch"]
    }
  ]
}
```

### rezept.md — Aufbau

Jede Datei hat exakt vier Abschnitte:

```
[YAML FRONT MATTER zwischen ---]
# Zutaten
# Bilder
# Zubereitung
```

**Front Matter:**
```yaml
---
title: Schweinefilet mit Backpflaumen
kategorie: Fleisch
personen: 4
zeit_gesamt: 780
zeit_aktiv: 30
schwierigkeit: mittel
bild: pic1
tags: [Festlich, Fleisch, Schmorgerichte]
---
```

**Zutaten** — eine Zeile pro Zutat:
```
- 800 g Schweinefilet
- 15 Stück Backpflaumen (ohne Stein)
- Salz und Pfeffer nach Geschmack
```
Format: `- <Menge> <Einheit> <Name> (<optionale Anmerkung>)`
Einheiten: `g` | `kg` | `ml` | `l` | `EL` | `TL` | `Stück` | `Prise` | `Bund` | `Zehe`

**Bilder:**
```
[pic1]: pic1.jpg
[pic2]: pic2.jpg
```
Mapping von Referenz → Dateiname im gleichen Ordner.

**Zubereitung — Schritt-Format (jeder Schritt = eine Zeile):**
```
[STEPn] <Text> [TIMER:XDXHXxM|keine] [PARALLEL:STEPn|keine] [BILD:picN|keine]
```

| Tag | Mögliche Werte | Bedeutung |
|---|---|---|
| `[STEPn]` | `[STEP1]`–`[STEP99]` | Schrittnummer, lückenlos ab 1 |
| `[TIMER:...]` | `[TIMER:0D12H00M]` oder `[TIMER:keine]` | Wartezeit: Tage/Stunden/Minuten |
| `[PARALLEL:...]` | `[PARALLEL:STEP3]` oder `[PARALLEL:keine]` | Gleichzeitig mit Schritt n möglich |
| `[BILD:...]` | `[BILD:pic1]` oder `[BILD:keine]` | Bild das bei diesem Schritt angezeigt wird |

Alle vier Bestandteile sind bei **jedem Schritt Pflicht**.

**Beispielschritte:**
```
[STEP1] Backpflaumen in Weinbrand einweichen. [TIMER:0D12H00M] [PARALLEL:keine] [BILD:keine]
[STEP2] Äpfel fein würfeln. [TIMER:keine] [PARALLEL:STEP3] [BILD:keine]
[STEP3] Filet aufschneiden und würzen. [TIMER:keine] [PARALLEL:STEP2] [BILD:pic1]
[STEP7] Bei geringer Hitze schmoren. [TIMER:0D00H35M] [PARALLEL:keine] [BILD:keine]
```

### Kotlin Data-Klassen

```kotlin
data class RecipeIndex(val version: String, val rezepte: List<RecipeMetadata>)

data class RecipeMetadata(
    val id: String,
    val title: String,
    val kategorie: String,
    val ordner: String,
    val vorschaubild: String,
    val zeit_gesamt: Int,
    val zeit_aktiv: Int,
    val schwierigkeit: String,
    val personen: Int,
    val tags: List<String>
)

data class Recipe(
    val metadata: RecipeMetadata,
    val zutaten: List<Ingredient>,
    val bilder: Map<String, String>,  // "pic1" → "pic1.jpg"
    val schritte: List<Step>
)

data class Ingredient(
    val menge: String,
    val einheit: String,
    val name: String,
    val anmerkung: String?
)

data class Step(
    val nummer: Int,
    val text: String,
    val timerMs: Long?,    // null = kein Timer
    val parallelZu: Int?,  // null = PARALLEL:keine, sonst Schrittnummer
    val bild: String?      // null = BILD:keine, sonst "pic1" etc.
)

data class ActiveRecipe(
    val recipe: Recipe,
    val aktuellerSchritt: Int,
    val timerRemainingMs: Long,
    val timerLaeuft: Boolean,
    val pausiert: Boolean
)

data class ShoppingItem(
    val id: String,
    val zutat: Ingredient,
    val rezeptTitle: String,
    val erledigt: Boolean = false,
    val manuell: Boolean = false
)
```

---

## App-Struktur: Drei Tabs

### Tab 1 — „Aktuell 🍳"

- Bis zu **5 Rezepte gleichzeitig** aktiv, jedes als eigene `Card`
- Pro Karte: Rezeptname, `LinearProgressIndicator` (Schritt x von n), aktueller Schritttext groß
- Wenn `parallelZu != null`: Hinweisbox in Teal: „Parallel möglich: [Text des referenzierten Schritts]"
- Wenn `timerMs != null`: Kreisförmiger Countdown-Ring (`Canvas`), große Zeitanzeige
- **Bei Timer-Ablauf:**
  - Vollbild-`AlertDialog`: Rezeptname + „Schritt [n] ist fertig!"
  - Android `Notification` via `NotificationManager` (auch wenn App im Hintergrund via `WorkManager`)
  - Vibration
- Buttons pro Karte (min. 60dp hoch): „◀ Zurück" | „Weiter ▶" | „⏸ Pausieren" | „✕ Beenden"
- Leerzustand: Illustration + „Öffne ein Rezept und tippe ‚Jetzt kochen'"

### Tab 2 — „Rezepte 📖"

- `LazyVerticalGrid` (2 Spalten auf Tablet, 1 auf Phone)
- Jede Karte: `AsyncImage` (Coil, Placeholder wenn kein Bild), Titel, Kategorie-`FilterChip`, Zeitangabe
- Oben: `SearchBar` + horizontale `FilterChip`-Leiste (Kategorien aus `index.json`)
- Rezept antippen → Detail-Screen:
  - Großes Headerbild (`pic1`)
  - Zutaten als `LazyColumn` mit Personenanzahl-Anpasser (Stepper + / -)
  - Schritt-Timeline: Schrittnummer-Badge, Text, Timer-Badge (falls vorhanden), Bild (falls vorhanden)
  - Zwei FABs: „Zur Einkaufsliste ➕" und „Jetzt kochen 🍳"
- `PullToRefreshBox` zum Neuladen aus GitHub

### Tab 3 — „Einkaufsliste 🛒"

- `LazyColumn` mit zusammenklappbaren Gruppenheadern pro Rezept
- **Automatische Mengenaggregation** gleicher Zutaten (gleicher Name + gleiche Einheit werden summiert)
- Zutat antippen → `erledigt = true` (durchgestrichen + ausgegraut)
- `SwipeToDismissBox` (Swipe links) → Eintrag löschen
- FAB: Manuelle Zutat hinzufügen (`AlertDialog` mit `TextField`)
- Oben rechts: „Liste leeren" Button mit Bestätigungs-`AlertDialog`
- Persistenz: Einkaufsliste überlebt App-Neustart (DataStore)

---

## Tablet-Layout & Design

### Responsive Layout
- `WindowSizeClass` nutzen:
  - `COMPACT` (Phone): `BottomNavigationBar`
  - `MEDIUM` / `EXPANDED` (Tablet): `NavigationRail` links
  - `EXPANDED`: Rezepte-Tab zweispaltig (Liste links, Detail rechts persistent als `PermanentNavigationDrawer`-Muster)

### ColorScheme (Material 3, kein dynamicColor)
```kotlin
val TealPrimary      = Color(0xFF01696F)
val TealOnPrimary    = Color(0xFFFFFFFF)
val WarmBackground   = Color(0xFFF7F6F2)
val WarmSurface      = Color(0xFFFFFFFF)
val WarmOnBackground = Color(0xFF1C1B1A)
// Kein Pink, kein Lila
```

### Typography (überschreiben)
```kotlin
val AppTypography = Typography(
    bodyLarge    = TextStyle(fontSize = 18.sp, lineHeight = 26.sp),
    bodyMedium   = TextStyle(fontSize = 16.sp),
    titleLarge   = TextStyle(fontSize = 24.sp, fontWeight = FontWeight.Bold),
    headlineMedium = TextStyle(fontSize = 28.sp, fontWeight = FontWeight.Bold)
)
```

### Touch-Targets
- Alle interaktiven Elemente: `Modifier.minimumInteractiveComponentSize()` + `height(60.dp)`
- Keine Hover-only-Interaktionen

---

## Liefere vollständig:

1. Komplettes Android Studio Projekt:
   - Alle Kotlin-Dateien (ViewModel, Repository, Parser, Composables)
   - `build.gradle.kts` (App + Projekt)
   - `libs.versions.toml`
   - `AndroidManifest.xml` (inkl. Notification-Permission, Internet-Permission, WorkManager)
   - `res/` (Strings, Colors, Theme)

2. Markdown-Parser der alle Tags zuverlässig per Regex extrahiert:
   - YAML Front Matter
   - Zutaten-Zeilen
   - Bilder-Mapping
   - `[STEPn]`, `[TIMER:...]`, `[PARALLEL:...]`, `[BILD:...]`

3. Timer-System:
   - `CountDownTimer` im ViewModel für Vordergrund
   - `WorkManager`-Worker für Hintergrund-Notification wenn App minimiert

4. Vollständige UI für alle drei Tabs inkl. Tablet-Adaptionen
