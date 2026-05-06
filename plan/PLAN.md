# WodConnect → Wodbook Import Feature Plan

**Goal:** Let a user import their WodConnect journal (`.xlsx`) into the Ennätykset (Personal Records) section of Wodbook.  
**Source:** WodConnect export file — `journal.xlsx` (1 139 rows, 693 unique workout names analysed)  
**Target:** `users/{uid}/personalRecords/records` Firestore document, same schema as `saveManualPR()`

---

## 1. Source ↔ Target Field Mapping

### Excel columns (WodConnect)

| Column                | Example values                                                |
| --------------------- | ------------------------------------------------------------- |
| `Id`                  | `16587449`                                                    |
| `Happened at`         | `2026-04-24 03:00:00`                                         |
| `Workout name`        | `"Deadlift Progression 1/4 // WOD"`                           |
| `Workout description` | long text, skip                                               |
| `Workout type`        | `Strength` / `Metcon`                                         |
| `Result type`         | `-` / `Weight` / `Time` / `AMRAP` / `Other` / `No-result`     |
| `Result`              | `"90, 100, 105, 110 kg"` / `42.5` / `"7:54"` / `3` / `"DONE"` |
| `Comment`             | free text                                                     |

### Wodbook PR fields (from `saveManualPR()`)

| Field         | Input element               | Stored as                           |
| ------------- | --------------------------- | ----------------------------------- |
| Movement name | `#apr-move` / `#apr-custom` | key in `personalRecords` object     |
| Type          | `#apr-type`                 | determines record shape             |
| Paino (kg)    | `#apr-kg`                   | `rec.kg`                            |
| Toistot       | `#apr-reps`                 | `rec.reps`, `rec.repMaxes[reps].kg` |
| Aika (mm:ss)  | `#apr-time`                 | `rec.timeStr`, `rec.totalSec`       |
| Matka (m)     | `#apr-meters`               | `rec.meters`                        |
| Tulos (other) | `#apr-other`                | `rec.notes`                         |
| Päivämäärä    | `#apr-date`                 | `rec.date` (`"YYYY-MM-DD"`)         |
| Kommentti     | `#apr-comment`              | `rec.comment`                       |

---

## 2. Result Type Mapping Rules

| WodConnect `Result type` | WodConnect `Result` example     | → Wodbook type | Parsing logic                                                                                                           |
| ------------------------ | ------------------------------- | -------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `-`                      | `"90, 100, 105, 110 kg"`        | `strength`     | Split by `,`, strip `" kg"`, take **highest valid number** as `kg`, `reps = 1` (unless workout name implies e.g. "5RM") |
| `Weight`                 | `42.5`                          | `strength`     | Direct float as `kg`, `reps = 1`                                                                                        |
| `Time`                   | `"7:54"`                        | `time`         | Parse `mm:ss` → `totalSec`, store `timeStr`                                                                             |
| `AMRAP`                  | `3` (rounds)                    | `reps`         | Direct integer as `reps`                                                                                                |
| `Other`                  | `"DONE"`, `"5 rounds + 3 reps"` | `other`        | Store as `notes` — **skip if value is exactly `"DONE"`**                                                                |
| `No-result`              | `"DONE"`                        | skip           | Discard entirely — no result to import                                                                                  |

### Weight series parsing detail (`Result type = "-"`)

```
Input:  "90, 100, 105, 110, 110 kg"
Step 1: strip trailing " kg"         → "90, 100, 105, 110, 110"
Step 2: split(",").map(parseFloat)   → [90, 100, 105, 110, 110]
Step 3: filter(n => n > 0 && n < 500) → remove zeros/typos (e.g. "6570" from "65, 70, 70, 6570 kg")
Step 4: Math.max(...)                → 110
Result: kg = 110, reps = 1
```

**Reps hint from workout name** (optional enhancement):
- `"Back Squat 1RM"` → reps = 1
- `"Back Squat 5RM"` or `"5 @ 80%"` in description → reps = 5
- Default: reps = 1

---

## 3. Movement Name Normalisation

WodConnect names are verbose: `"Deadlift Progression 1/4 // WOD"`, `"Front Squat // WOD"`, `"Back Squat 1RM"`.  
Wodbook has a fixed `MOVEMENTS` list plus free-text custom entries.

### Step 1 — Strip noise suffixes

```js
function normaliseMoveName(raw) {
  return raw
    .replace(/\s*\/\/\s*WOD.*/i, '')      // "Deadlift // WOD" → "Deadlift"
    .replace(/\s+Progression\s+\d+\/\d+/i, '') // "... Progression 1/4" → ""
    .replace(/\s+week\s+\d+.*/i, '')       // "... week 3" → ""
    .replace(/\s+\d+\/\d+$/,'')            // "... 1/4" at end → ""
    .replace(/\s+(1|2|3|4|5|6|7|8|9|10)RM$/i, '') // "Back Squat 1RM" → "Back Squat"
    .trim();
}
```

### Step 2 — Match against known MOVEMENTS list

```js
const ALL_KNOWN_MOVES = [
  // strength
  'Back Squat','Front Squat','Overhead Squat','Deadlift','Romanian Deadlift',
  'Bench Press','Dual DB Bench Press','Strict Press','Push Press','Push Jerk','Split Jerk',
  'Snatch','Clean','Squat Clean','Clean & Jerk','Power Snatch','Power Clean',
  'Hang Power Clean','Hang Squat Clean','Hang Power Snatch','Thruster',
  'Single Arm DB Row','Row (1RM)',
  // gymnastics
  'Pull-up (max)','Chest-to-Bar (max)','Muscle-up (max)','Ring Muscle-up (max)',
  'Handstand Push-up (max)','Toes-to-Bar (max)','Dip (max)','L-sit (sek)',
  // benchmark
  'Fran','Helen','Grace','Isabel','Linda','Cindy (kierrokset)',
  'Murph','Annie','Barbara','Nancy','Karen','Jackie',
  'Diane','Desire','Fight Gone Bad','Grettel'
];

function matchToKnown(normalised) {
  // 1. Exact match
  const exact = ALL_KNOWN_MOVES.find(m => m.toLowerCase() === normalised.toLowerCase());
  if (exact) return { move: exact, isCustom: false };

  // 2. Known move is a prefix of normalised (e.g. "Back Squat" ⊂ "Back Squat 5 sets")
  const prefix = ALL_KNOWN_MOVES.find(m =>
    normalised.toLowerCase().startsWith(m.toLowerCase())
  );
  if (prefix) return { move: prefix, isCustom: false };

  // 3. Normalised is a prefix of known move
  const suffix = ALL_KNOWN_MOVES.find(m =>
    m.toLowerCase().startsWith(normalised.toLowerCase())
  );
  if (suffix) return { move: suffix, isCustom: false };

  // 4. No match → use normalised name as custom
  return { move: normalised, isCustom: true };
}
```

### Step 3 — Deduplicate per movement across rows

Multiple WodConnect rows for the same movement (e.g. 53 "Back Squat" entries) → **keep only the best result per movement** for the initial import (same logic as the existing PR system — only best counts).

For `strength`: highest `kg` across all matching rows.  
For `time`: lowest `totalSec`.  
For `reps`/`meters`: highest value.  
Comment: use the comment from the row that had the best result.

---

## 4. Import Flow (UI)

### Where to add the button

In `index.html`, the Ennätykset (Records) tab of the profile view — next to the existing "LISÄÄ ENNÄTYS" button:

```html
<!-- Existing button -->
<button onclick="openAddPR()">LISÄÄ ENNÄTYS</button>

<!-- New button -->
<button onclick="openImportPR()">TUO WODCONNECT</button>
<input type="file" id="pr-import-file" accept=".xlsx,.csv" style="display:none"
       onchange="handleImportFile(event)">
```

### Import modal screens (3 steps)

**Step 1 — File picker (auto-opens file dialog)**
```
[Tuo WodConnect -tiedosto]
Valitse tiedosto (.xlsx tai .csv)
```

**Step 2 — Preview table (before writing anything)**
```
Löydettiin 287 ennätystä 189 eri liikkeestä.

┌─────────────────────┬──────────┬───────┬──────────┬────────────────────┐
│ Liike               │ Tyyppi   │ Tulos │ Pvm      │ Huomio             │
├─────────────────────┼──────────┼───────┼──────────┼────────────────────┤
│ Back Squat          │ Voima    │ 120kg │ 2025-11  │ ✓ tunnettu liike   │
│ Front Squat         │ Voima    │  90kg │ 2026-01  │ ✓ tunnettu liike   │
│ Murph               │ Aika     │ 47:32 │ 2025-06  │ ✓ tunnettu liike   │
│ DB Strength         │ Voima    │  25kg │ 2026-03  │ ⚠ oma nimi         │
│ AMRAP // WOD        │ Toistot  │     3 │ 2026-04  │ ⚠ oma nimi         │
│ DONE-rivit          │  –       │  –    │    –     │ ✗ ohitettu (340 kpl)|
└─────────────────────┴──────────┴───────┴──────────┴────────────────────┘

[Tuo kaikki]  [Tuo vain tunnetut liikkeet]  [Peruuta]
```

**Step 3 — Result toast**
```
✓ Tuotu 287 ennätystä. 189 liikettä päivitetty.
```

---

## 5. Conflict Resolution

When a WodConnect entry and an existing Wodbook PR exist for the same movement:

| Condition                                | Action                              |
| ---------------------------------------- | ----------------------------------- |
| Import value is **better** than existing | Overwrite (same as manual PR entry) |
| Import value is **equal or worse**       | Keep existing, skip                 |
| Existing PR has no date                  | Add date from import                |

User option in preview step: **"Korvaa kaikki"** checkbox — overwrites regardless.

---

## 6. Firestore Write

Reuse the existing `saveRecordsToFS()` function exactly — no new Firestore paths needed.

```js
async function importPRsToFirestore(importedRecords) {
  // Merge with existing records
  const merged = { ...appState.personalRecords };

  for (const [move, incoming] of Object.entries(importedRecords)) {
    const existing = merged[move];
    if (!existing) {
      merged[move] = incoming;          // new movement, always add
    } else {
      merged[move] = mergeRecord(existing, incoming); // keep best
    }
  }

  appState.personalRecords = merged;
  saveRecordsToFS();                    // existing function — no changes needed
  renderProfPRs();                      // existing function — no changes needed
}

function mergeRecord(existing, incoming) {
  // strength: keep highest kg
  if (incoming.kg && (!existing.kg || incoming.kg > existing.kg)) {
    return { ...existing, ...incoming };
  }
  // time: keep lowest totalSec
  if (incoming.totalSec && (!existing.totalSec || incoming.totalSec < existing.totalSec)) {
    return { ...existing, ...incoming };
  }
  // reps/meters: keep highest
  if (incoming.reps && (!existing.reps || incoming.reps > existing.reps)) {
    return { ...existing, ...incoming };
  }
  if (incoming.meters && (!existing.meters || incoming.meters > existing.meters)) {
    return { ...existing, ...incoming };
  }
  return existing; // existing is better, no change
}
```

---

## 7. CSV Fallback

If user has a `.csv` export instead of `.xlsx`:

Expected CSV format (same columns as WodConnect xlsx):
```
Id,Happened at,Workout name,Workout description,Workout type,Result type,Result,Comment
```

Parse with built-in `FileReader` + manual CSV split (no library needed for simple CSVs). If cells contain commas, they will be quoted — handle with a minimal quoted-CSV parser.

---

## 8. Library: SheetJS (xlsx.js)

For `.xlsx` parsing in the browser, use SheetJS (MIT licensed, no backend needed):

```html
<!-- Add to index.html <head> or just before the import function -->
<script src="https://cdn.sheetjs.com/xlsx-0.20.3/package/dist/xlsx.full.min.js"></script>
```

Usage:
```js
function handleImportFile(event) {
  const file = event.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (e) => {
    const data = new Uint8Array(e.target.result);
    const wb = XLSX.read(data, { type: 'array', cellDates: true });
    const ws = wb.Sheets[wb.SheetNames[0]];
    const rows = XLSX.utils.sheet_to_json(ws, { defval: '' });
    processImportRows(rows);
  };
  reader.readAsArrayBuffer(file);
}
```

SheetJS `cellDates: true` converts the "Happened at" column to JS `Date` objects automatically.

---

## 9. Implementation Checklist

For the implementing Claude session — do these in order:

- [ ] **1.** Add SheetJS CDN `<script>` tag to `index.html` (in `<head>`)
- [ ] **2.** Add "TUO WODCONNECT" button and hidden `<input type="file">` next to the existing LISÄÄ ENNÄTYS button in the Ennätykset section
- [ ] **3.** Add import modal HTML (3-panel: step indicator + preview table + result screen) to `index.html` modal section
- [ ] **4.** Write `normaliseMoveName(raw)` — strip WOD noise suffixes
- [ ] **5.** Write `matchToKnown(normalised)` — match against `ALL_KNOWN_MOVES` (mirrors the `MOVEMENTS` constant)
- [ ] **6.** Write `parseWodConnectResult(row)` — maps result type + value to Wodbook record shape
- [ ] **7.** Write `processImportRows(rows)` — calls 4+5+6, deduplicates to best-per-movement, builds preview data
- [ ] **8.** Write `renderImportPreview(data)` — fills the modal table, shows counts and warnings
- [ ] **9.** Write `importPRsToFirestore(importedRecords)` — merges with existing, calls `saveRecordsToFS()`
- [ ] **10.** Wire up `openImportPR()` → file dialog → `handleImportFile()` → preview → confirm → import
- [ ] **11.** Test with `journal.xlsx` — verify counts: expect ~189 unique movements imported, ~340 DONE rows skipped

---

## 10. Key Numbers from journal.xlsx

| Metric                                                      | Count |
| ----------------------------------------------------------- | ----- |
| Total rows                                                  | 1 139 |
| Rows with no importable result (`No-result` + `Other`=DONE) | ~388  |
| Strength rows (weight series `-`)                           | 440   |
| Time rows                                                   | 198   |
| AMRAP rows (rounds)                                         | 129   |
| Weight rows (single kg)                                     | 32    |
| Unique workout names (raw)                                  | 693   |
| Estimated unique movements after normalisation              | ~189  |
| Movements matching Wodbook's known list                     | ~25   |
| Movements that become custom entries                        | ~164  |

---

## 11. Edge Cases to Handle

| Case                                              | Handling                                                          |
| ------------------------------------------------- | ----------------------------------------------------------------- |
| Weight series with typo (`"65, 70, 70, 6570 kg"`) | Filter out values > 500kg                                         |
| Zero weight in series (`"65, 70, 75, 80, 0 kg"`)  | Filter out zeros                                                  |
| AMRAP result is a large integer (`168`)           | Store as `reps = 168` (rounds × reps, WodConnect aggregation)     |
| Movement name is blank after normalisation        | Skip row                                                          |
| File has extra/missing columns                    | Detect by header name, not column index                           |
| Date is `null` / missing                          | Use today's date as fallback                                      |
| User already has a better PR                      | Keep existing (or respect "Korvaa kaikki" toggle)                 |
| `.csv` with Finnish characters (UTF-8 BOM)        | `reader.readAsText(file, 'utf-8')`, strip BOM `﻿`                 |
| Very large file (10k+ rows)                       | Parse synchronously in `onload` is fine up to ~50k rows in memory |
