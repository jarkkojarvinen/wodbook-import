# Claude Instructions — wodbook-import-app

## Project purpose

Standalone browser tool that imports personal records (PRs) from a WodConnect `journal.xlsx` export directly into a user's [wodbook.fi](https://wodbook.fi/) profile. No backend, no build step — one self-contained HTML file that runs on localhost.

---

## Folder structure

```
wodbook-import-app/
├── CLAUDE.md              ← this file
├── .vscode/tasks.json     ← VS Code task: python3 -m http.server 8081
├── .github/workflows/
│   └── pages.yml          ← GitHub Actions: auto-deploy app/ to GitHub Pages
└── app/
    ├── index.html              ← THE ENTIRE IMPORT TOOL (HTML + CSS + JS)
    ├── clear-prs.html          ← danger tool: deletes all PRs from Firestore
    ├── favicon.svg             ← app icon
    ├── wodconnect-export.png   ← guide screenshot shown in file-pick step
    └── README.md               ← end-user + developer docs (keep in sync)
```

Test `.xlsx` files and `plan/` snapshots are kept locally but not committed to version control.

---

## Before making any change

1. **Check wodbook.fi source parity.** The MOVEMENTS list and Firebase SDK version in `app/index.html` must match the live site. Fetch a fresh copy of `https://wodbook.fi/index.html` (view-source) and compare before editing. Last verified: **2026-05-09** — SDK 10.14.1, schema unchanged, no new PR categories.

2. **Test with xlsx files.** Local test exports (`journal.xlsx`, `journal_2.xlsx`) are in the root but not committed. Both must parse without errors and show sensible stats.

3. **Always update `app/README.md`** after any change to `app/index.html`. The README is the end-user manual — keep usage, feature list, and the "how it works" data-flow diagram accurate.

---

## How to run locally

```bash
# From the repo root (or use VS Code task "Host App Folder")
python3 -m http.server 8081 --directory app
```

Open `http://localhost:8081/` — Firebase Auth requires `localhost`; `file://` does not work.

The VS Code task in `.vscode/tasks.json` runs `python3 -m http.server 8081` (serves from the working directory, so launch from `app/`).

---

## Firebase configuration

The tool uses the **same Firebase project** as wodbook.fi — no separate backend needed.

```js
apiKey:            'AIzaSyC7NAGjrPUG8fjxvGI0EjLSUO5_tNfN-WQ'
authDomain:        'wodbook-4a6ac.firebaseapp.com'
projectId:         'wodbook-4a6ac'
storageBucket:     'wodbook-4a6ac.firebasestorage.app'
messagingSenderId: '48961069729'
appId:             '1:48961069729:web:38de4d2b49edc68ee186e0'
```

Firestore path written: `users/{uid}/personalRecords/records` — same document the Wodbook app itself reads/writes. Field `_updated: serverTimestamp()` is added on every write.

Firebase JS SDK version: **10.14.1** (must match wodbook.fi).
SheetJS version: **0.20.3** (CDN: `https://cdn.sheetjs.com/xlsx-0.20.3/package/dist/xlsx.full.min.js`).

---

## Key implementation details in `app/index.html`

### MOVEMENTS list

Hardcoded near the top of the `<script type="module">` block. Three categories:

- `strength` — Back Squat, Front Squat, Deadlift, Clean & Jerk, etc. (23 moves)
- `gymnastics` — Pull-up (max), HSPU (max), etc. (8 moves)
- `benchmark` — Fran, Helen, Murph, Cindy (kierrokset), etc. (19 moves — includes Ellen, Kelly, Karabel as intentional extras not in live wodbook.fi)

`ALL_KNOWN_MOVES` is sorted **longest-first** so prefix matching prefers the most-specific move (e.g. "Hang Power Clean" beats "Power Clean" beats "Clean").

### ALIASES table

Maps common abbreviations and WodConnect naming variants to known moves (e.g. `'ohp'` → `'Strict Press'`, `'c2b'` → `'Chest-to-Bar (max)'`). Lives just below the MOVEMENTS block.

### Name normalisation pipeline

`normaliseName(raw)` strips: `// WOD`, `Progression N/N`, `week N`, fraction suffix `N/N`, `NRM` suffix, `@ XX%` suffix.

`matchKnown(normalised)` — 5-tier matching (exact → alias → prefix → word-containment → suffix fallback → custom).

`resolveName(rawName, rawDesc)` — if the workout name gives only a weak match, also tries the first line of `Workout description` (WodConnect often puts the real exercise name there).

### Result parsing (`parseRow`)

| WodConnect `Result type` | Parsed as |
|---|---|
| `No-result` | skipped |
| `Other` = `"DONE"` | skipped |
| `-` (weight series) | strength — split by `,`, strip `kg`, filter 0 and >500, take max |
| `Weight` (single kg) | strength |
| `Time` | time — supports `SS`, `MM:SS`, `HH:MM:SS` |
| `AMRAP` | reps |
| `Other` (non-DONE) | tries time parse first, then `other` with notes |

### Rep-set pairing (`tryPairRepSets`)

For strength rows with multiple weights in the series: tries to pair them with a rep scheme found in name/description.

- Path 1: dash scheme (`"4-4-3-2-2"`) aligned with weight count
- Path 2: `"Set 1-2: 5 reps"` / `"Set 3: 3 reps"` annotations
- Path 3: `"N reps @X%"` → all sets same rep count

Stores results in `repMaxes: { [reps]: { kg, date } }` — matches Wodbook's own PR schema exactly.

### Deduplication

`buildImportData` keeps the **best result per movement** across all rows. For strength, it accumulates all rep-count slots (1RM, 2RM, 3RM, …) into a single `repMaxes` map. For other types, it keeps whichever row had the better result.

### Import / merge (`doImport` + `mergeRecord`)

Re-fetches current Firestore PRs just before writing (avoids stale-data overwrite). Merges slot-by-slot for strength `repMaxes` — never downgrades an existing slot. `_updated: serverTimestamp()` field is added.

---

## UI flow

1. **Login** — email + password (same credentials as wodbook.fi). Privacy note shown: credentials go only to Firebase/Wodbook, not stored locally.
2. **File picker** — guide image (wodconnect-export.png) shows where to find the download button in WodConnect. Drag-and-drop or click; `.xlsx` and `.csv` accepted; parsed in-browser.
3. **Review table** — opt-in selection (nothing checked by default). Filter by status (Uudet / Paremmat / Huonommat / Tunnistamattomat) and by category (Voima / Gymnastics / Benchmark / Muut); per-row checkboxes; "Korvaa kaikki" override. Page scrolls to top automatically when review loads.
4. **Import** — writes only checked rows; status message shows count. Imported rows get a ✓ TUOTU badge.

---

## clear-prs.html

A separate danger tool at `http://localhost:8081/clear-prs.html` (or the GitHub Pages URL). Deletes `users/{uid}/personalRecords/records` entirely after requiring the user to type `POISTA`. Includes a link back to the import tool. Useful for a clean slate before re-importing.

## Deployment

The app is deployed to GitHub Pages automatically via `.github/workflows/pages.yml` on every push to `main`. Live URL: **https://jarkkojarvinen.github.io/wodbook-import/**

For GitHub Pages to work, two things must be configured in the Firebase project (`wodbook-4a6ac`):
- **Firebase Auth authorized domains**: add `jarkkojarvinen.github.io`
- **Google Cloud API key HTTP referrer restriction**: add `https://jarkkojarvinen.github.io/*` to the API key used by the app

---

## Workflow for code changes

1. Fetch `https://wodbook.fi/index.html` (view-source) to verify MOVEMENTS list parity.
2. Edit `app/index.html`.
3. Test by opening `http://localhost:8081/` and loading local test xlsx files. Check stats (movement count, skipped count) are reasonable.
4. Update `app/README.md` to reflect any user-visible changes.
5. Update this `CLAUDE.md` if the architecture or workflow changes.

---

## Known edge cases already handled

| Case | Where handled |
|---|---|
| Weight series typo (`"65, 70, 6570 kg"`) | `parseRow` filters values > 500 |
| Zero weight in series | `parseRow` filters values <= 0 |
| Date one day off (UTC shift) | `parseDate` reads local date from JS `Date` object |
| `Result type = "Other"` with time value (e.g. Grace `"5:09"`) | `parseRow` tries time parse before `other` |
| Workout name strips to empty | `buildImportData` skips the row |
| Description fallback for generic names | `resolveName` checks first non-empty description line |
| Stale Firestore data on import | `doImport` re-fetches before writing |
| `_updated` metadata key in existing PRs | Destructured out before merge |

---

## Do not do

- Do not add a backend — the tool must remain a single static HTML file.
- Do not change the Firestore path or document schema — it must stay identical to what `saveManualPR()` in wodbook.fi writes.
- Do not remove the `_updated: serverTimestamp()` field from writes.
- Do not commit API keys to a public repo (the Firebase config here is safe — it is already public in the live site's HTML and protected by Firestore security rules).
