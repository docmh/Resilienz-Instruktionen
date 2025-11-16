This app is a survey app, built to test for parallel test reliability of a specific questionnaire. The app is **GERMAN** and targets self-hosting on an IONOS Plesk server (no Docker). Develop and test on Windows 11; deploy on Linux/Plesk. Use the tech stack and constraints below.

## Tech Stack & Runtime
- Frontend: React + TypeScript (e.g., Vite) SPA, served as static files. Card-based UI; header/body/footer layout with footer nav buttons (none on outro). German copy centralized in a locale file.
- Backend: Python FastAPI + Uvicorn/Gunicorn (no Docker). All OpenAI calls server-side only.
- Storage: CSV files in a writable `data/` directory (configurable via `DATA_DIR`, default `./data`). No database. Use UTF-8, `newline=""`, fixed headers, and a stable delimiter (`,` or `;`).
- Paths: Use `pathlib.Path` everywhere for Windows/Linux portability. Ensure `DATA_DIR.mkdir(parents=True, exist_ok=True)` at startup.
- Concurrency: Protect CSV appends with a global `asyncio.Lock`; open files with `newline=""` and `encoding="utf-8"`. Fsync after each append (tiny traffic, ~50 rows).
- Timestamps: ISO 8601 UTC (`YYYY-MM-DDTHH:MM:SS.sssZ`). On intro→first page: record `start_ts`. On demographics→outro: record `end_ts`, compute `duration_secs`, then write CSV.
- Sessions: Create a `session_id` UUID at consent/intro. Keep session state server-side (in-memory dict keyed by session_id; optional JSON spillover under `data/sessions/`). Each page POST merges data into session. On final submit, assemble one row and append to CSV. No navigation to next page until all items on the page answered; back navigation allowed except outro.
- Download: Provide `/download` to stream the main CSV with `Content-Disposition` for SPSS import.
- OpenAI integration: Model `gpt-5.1` for generation; embedding `text-embedding-3-small` for similarity. Items generated server-side only. Use a job id: client POST starts generation, immediately shows loading “Bitte warten Sie einen Moment, während wir die Fragen für Sie generieren.” Client polls job status until items ready; cache per session. Accept items only if similarity with source > 0.8; retry until all pass. Save for each C item: id, label, similarity.

## Pages and Navigation
1. Introduction with consent
2. Parallel resilience forms (each on its own page; randomized order):
   a) Static parallel form A
   b) Static parallel form B
   c) Dynamic parallel form (generated)
3. Stress perception (PSQ 20)
4. Demographics (age, gender, education level, professional status, seriousness-check)
5. Outro (no nav buttons)

All pages have header, body, footer (footer nav buttons except outro). Next page only when all items on current page answered. Previous page always allowed except on outro.

## Questionnaires and Items
Questionnaires have headline, instruction, list of item cards. Each card shows the item text on top, answer controls below.
- Likert: radio buttons with labels shown; save numeric values.
- Dropdowns: label + select box.
- Options: vertical radio buttons.

## Contents of Questionnaires
### Resilience Forms
Instruction (all forms):  
“Lieber Teilnehmer, in dieser Studie soll untersucht werden, was Menschen stark macht, um ihr Leben auch in schweren Situationen erfolgreich bewältigen zu können. Bitte beantworten Sie die nun folgenden Fragen auf einer Skala von 1 = trifft nicht zu bis 5 = trifft voll zu.”

Likert labels (store numeric): 1 “trifft nicht zu”, 2 “trifft selten zu”, 3 “trifft manchmal zu”, 4 “trifft häufig zu”, 5 “trifft voll zu”.

#### Resilience - Parallel Form A
Items:
1. Ich bin emotional sehr belastbar.
2. Ich lasse mich nicht so schnell unterkriegen.
3. Ich kann gut mit Druck umgehen.
4. Ich kann gut abschalten.
5. Viel Stress macht mir nichts aus.
6. Auch in unerwarteten Situationen bleibe ich gelassen.
7. Mich bringt nichts aus der Ruhe.

Item ids: `starc_5_A_i`. Save item id with response.

#### Resilience - Parallel Form B
Same items as form A (placeholder). Item ids: `starc_5_B_i`. Save item id with response.

#### Resilience - Dynamic Form
- Generate parallel items to form A using OpenAI (`gpt-5.1`, low thinking effort). Embedding similarity via `text-embedding-3-small`.
- Prompt as a professional test developer; match meaning and style, different wording.
- Item naming: parallel to `starc_5_A_i` => `starc_5_C_i`.
- Loop until similarity > 0.8 for each item; only accept passing items.
- Show loading screen text during generation: “Bitte warten Sie einen Moment, während wir die Fragen für Sie generieren.”
- After generation, randomize item order for display.
- Save per item: id, label, similarity, and answer.

### Stress Perception (PSQ 20)
Use instructions/items from `src/assets/psq_20.pdf`. Item ids: `psq_20_i`. Save item id with response.

### Demographic Variables
- Age (`alter`): “Bitte geben Sie Ihr Alter in Jahren an.” Dropdown: "< 16", "16", …, "67", "> 67".
- Gender (`geschlecht`): “Bitte ordnen Sie sich zu.” Dropdown values: weiblich=0, männlich=1, divers=2.
- Education (`bildung`): “Bitte geben Sie Ihren höchsten Bildungsgrad an.” Dropdown values: kein Abschluss=0, Hauptschulabschluss=1, Mittlere Reife=2, (Fach-) Abitur=3, Ausbildungsberuf=4, Bachelor=5, Master oder vergleichbar=6, Promotion=7.
- Professional status (`berufsstatus`): “Bitte geben Sie Ihren aktuellen beruflichen Status an.” Dropdown values: nicht erwerbstätig=0, Teilzeit, abhängig=1, Vollzeit, abhängig=2, Selbstständig=3.
- Seriousness check (`ernsthaftigkeit`): Radio options: Ja=1, Nein=0.
Save item id with response.

## Data Storage
- Store `start_ts` when leaving intro; `end_ts` + `duration_secs` when leaving demographics to outro.
- Keep clicks/answers in RAM per session; optional JSON spillover under `data/sessions/`.
- On demographics→outro transition: assemble row and append to CSV (UTF-8, fixed header, delimiter consistent). Include session_id, start_ts, end_ts, duration_secs, all item ids/answers, dynamic item labels and similarities, demographics.
- Make responses easy to download/SPSS-importable via `/download` endpoint.

## Scaling
Support multiple concurrent participants by isolating session state server-side (session_id keys) and serializing CSV writes with a lock. No participant data leaks across sessions.
