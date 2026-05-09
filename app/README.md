# WodConnect → Wodbook Import Tool

A standalone browser tool for importing personal records from a WodConnect `journal.xlsx` export into [wodbook.fi](https://wodbook.fi/).

No backend, no build step, no npm. One HTML file.

> This tool is also hosted on GitHub Pages at:
> `https://jarkkojarvinen.github.io/wodbook-import/`

---

## Usage

### 1. Open the app

Option 1: Use the hosted version

Open: **https://jarkkojarvinen.github.io/wodbook-import/**

Option 2: Run locally from the `app/` folder

```bash
python3 -m http.server 8081 --directory app
```

Then open: **http://localhost:8081/**

> **Why localhost and not `file://`?**  
> Firebase Authentication requires an authorized origin. `localhost` is whitelisted by default in Firebase; `file://` is not.

### 2. Log in

Use your normal Wodbook credentials (email + password). The tool connects to the same Firebase project as wodbook.fi — your data, your account.

Your credentials are sent directly to Wodbook/Firebase only — they are not stored in this tool and do not go to any third party.

### 3. Pick your file

Drag `journal.xlsx` onto the drop zone, or click to browse. The file is parsed entirely in the browser — nothing is uploaded anywhere.

> To export from WodConnect: open [wodconnect.com/profile/edit](https://www.wodconnect.com/profile/edit#profile_info), go to **Profile info**, and click **Download all results in Excel format**. The app shows a screenshot guide on this step.

### 4. Review

Before anything is written you see a table of every importable movement:

| Column | Description |
|---|---|
| Liike | Normalised movement name. "TUNNISTAMATON" badge = not a standard Wodbook move |
| Tyyppi | Voima / Aika / Toistot / Metrit / Muu |
| Paras tulos | Best result found across all your WodConnect entries for that movement |
| Pvm | Date of that best result |
| Nykyinen Wodbook PR | What you already have in Wodbook (fetched at login) |
| Tila | UUSI / PAREMPI / SAMA / HUONOMPI |

**Filter buttons** let you view subsets: Kaikki / Uudet / Paremmat / Huonommat / Tunnistamattomat.

**Category filter** — Voima / Gymnastics / Benchmark / Muut.

**Per-row checkboxes** — nothing is checked by default (opt-in). Select the rows you want to import.

**Korvaa kaikki** checkbox — forces overwrite even for results that are worse than your existing PR.

---

## Development Guidelines

### 1. Maintaining the Importer Tool
- Always ensure the importer tool is functional and compatible with the latest `wodbook.fi` site code.
- The `plan` folder contains resources for the `wodbook.fi` site, including source code and startup ideas for implementing the import tool. Always download the latest site code and verify compatibility.

### 2. Testing the Importer Tool
- Use the example journal files located in the root folder to test the functionality of the importer tool.
- Ensure all features, including file parsing, data review, and import functionality, work as expected.

### 3. Updating the README
- Whenever changes are made to the importer tool code, update this `README.md` file to reflect the changes.
- Include any new features, usage instructions, or testing guidelines.

---

## How it works

### File structure

```
import-app/
└── index.html    — the entire tool (HTML + CSS + JS, self-contained)
```

### External dependencies (CDN, no install needed)

| Library | Version | Purpose |
|---|---|---|
| [SheetJS](https://sheetjs.com/) | 0.20.3 | Parse `.xlsx` in the browser |
| [Firebase JS SDK](https://firebase.google.com/docs/web/setup) | 10.14.1 | Auth + Firestore (same version as wodbook.fi) |

### Data flow

```
journal.xlsx
  → SheetJS → raw rows[]
  → normaliseMoveName()     strip "// WOD", "Progression 1/4", "5RM" etc.
  → matchKnown()            match against Wodbook's MOVEMENTS list (longest-first)
  → parseRow()              map Result type + value to Wodbook record shape
  → keepBetter()            deduplicate: one best result per movement
  → toRecord()              build {kg, reps, repMaxes, date} / {totalSec, timeStr, date} etc.
  → review UI
  → mergeRecord()           slot-level repMaxes merge with existing Wodbook PRs
  → setDoc(Firestore)       write merged object with _updated: serverTimestamp()
```

### Firestore schema written

Matches `saveRecordsToFS()` in the live app exactly.

**Strength** (`Result type: "-"` or `"Weight"`):
```js
{
  kg: 110,          // best kg across all entries
  reps: 1,          // from "5RM" in workout name, else 1
  date: "2026-03-15",
  repMaxes: {
    1: { kg: 110, date: "2026-03-15" },
    5: { kg: 90,  date: "2025-11-02" },   // if multiple rep counts found
  }
}
```

**Time** (`Result type: "Time"`):
```js
{ totalSec: 287, timeStr: "4:47", date: "2026-01-10" }
```

**AMRAP / Reps** (`Result type: "AMRAP"`):
```js
{ reps: 12, date: "2025-09-20" }
```

**Meters** (if applicable):
```js
{ meters: 5000, date: "2026-02-14" }
```

**Other** (`Result type: "Other"`, non-DONE value):
```js
{ notes: "5 rounds + 3 reps", date: "2025-12-01" }
```

### Rows that are skipped

- `Result type: "No-result"`
- `Result type: "Other"` with value `"DONE"`
- Empty workout name
- Result that cannot be parsed (e.g. weight series with no valid numbers)

### Movement name normalisation

The tool strips WodConnect-specific noise before matching:

| Pattern | Example | Result |
|---|---|---|
| `// WOD` suffix | `"Front Squat // WOD"` | `"Front Squat"` |
| Progression marker | `"Deadlift Progression 1/4"` | `"Deadlift"` |
| Week marker | `"Back Squat week 3"` | `"Back Squat"` |
| Fraction suffix | `"Snatch 1/4"` | `"Snatch"` |
| RM suffix | `"Back Squat 5RM"` | `"Back Squat"` (reps=5 extracted) |
| `@ XX%` suffix | `"Deadlift @ 80%"` | `"Deadlift"` |

Matching uses the live Wodbook `MOVEMENTS` list (all three categories). The list is sorted longest-first so `"Clean & Jerk"` is matched before `"Clean"`.

#### Description fallback

When the workout name strips down to something too generic (e.g. `"Squat Progression 12/12 // WOD"` → `"Squat"`), the tool also tries the first line of the `Workout description` column. WodConnect often puts the real movement name there — e.g. `"BACK SQUAT 1RM"` — which normalises cleanly to `"Back Squat"`. The reps hint (`1RM`) is also extracted from the description in this case.

---

## Other tools

### clear-prs.html — Remove all personal records

Opens at `http://localhost:8081/clear-prs.html` (or the GitHub Pages URL).

- Logs in with your Wodbook credentials
- Shows the current PR count and a full list before doing anything
- Requires typing `POISTA` to confirm — prevents accidental deletion
- Calls `deleteDoc` on `users/{uid}/personalRecords/records`, the same path the Wodbook app itself uses for account deletion
- Includes a link back to the import tool

Useful for a clean slate before re-importing from WodConnect.

---

## Development

### Making changes

Edit `index.html` directly — it is the entire app. Refresh the browser. No build step. Push to `main` to deploy automatically to GitHub Pages.

### Mobile support

The review table renders as cards on screens ≤ 640 px: each movement is a card with name, result, category and status badge. Filter buttons wrap to multiple rows. Action buttons are full-width. Checkboxes are enlarged for touch.

### Keeping MOVEMENTS in sync with wodbook.fi

The known movement list is hardcoded near the top of the `<script>` block. If wodbook.fi adds new movements, update `ALL_KNOWN_MOVES` to match. The source of truth is the `MOVEMENTS` constant in `https://wodbook.fi/index.html`.

### Updating Firebase SDK version

Wodbook uses Firebase `10.14.1`. If it upgrades, change the version in the three import URLs at the top of the `<script type="module">` block:

```js
import { initializeApp }
  from 'https://www.gstatic.com/firebasejs/10.14.1/firebase-app.js';
```

### Updating SheetJS

Change the version in the `<script src>` tag in `<head>`:

```html
<script src="https://cdn.sheetjs.com/xlsx-0.20.3/package/dist/xlsx.full.min.js"></script>
```

Check [cdn.sheetjs.com](https://cdn.sheetjs.com) for the latest version.

### Testing with a different Firebase project

Replace the `firebaseConfig` object at the top of the script with your own project's config.

### Running with Node instead of Python

```bash
npx serve import-app/
# or
npx http-server import-app/ -p 8080
```

---

## Troubleshooting

| Problem | Likely cause | Fix |
|---|---|---|
| Login fails with "auth/invalid-credential" | Wrong email or password | Use the same credentials as wodbook.fi |
| Login works but PRs don't appear in Wodbook | Email not verified in Firebase | Verify your email via Wodbook first |
| "Sarake 'Workout name' puuttuu" | Wrong file format | Export from WodConnect as Excel, not PDF/CSV manually |
| Movement shows "TUNNISTAMATON" badge | Name not in Wodbook's known list | It will be saved as a custom PR — that's fine |
| Date appears one day off | Timezone edge case | Check that `Happened at` column has dates, not just times |
| Firestore permission denied | Logged in with wrong account | Log out and log in with the correct Wodbook account |
