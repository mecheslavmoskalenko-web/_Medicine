---
name: medical-record
description: Maintain a persistent personal medical record (patient profile, lab results, doctors' reports, medications, timeline, subjective symptoms, research notes, and todo list) built from uploaded medical documents. Use this skill whenever the user uploads or references lab results, analyses, doctor's reports/notes, prescriptions, ultrasound/imaging reports, or any medical document and wants it tracked, summarized, or added to their history — even if they don't explicitly say "medical record" or "skill". Also use whenever the user asks to review their health history, check trends across analyses, list current medications, or asks "what does my last analysis say" type questions. Trigger on any bundle of medical files (zip, folder, multiple PDFs/images) that looks like lab reports, clinical findings, or patient records — including Russian-language lab reports (Гемотест, ДНКОМ, etc.), which are common for this workflow.
---

# Medical Record

A skill for building and maintaining one patient's personal medical record over time, across many separate conversations and document uploads. The record is a set of markdown files (see `assets/`) that Claude keeps updating incrementally — never a one-off summary that gets thrown away.

## Where the record lives

- If the user already has a record (a `MedicalRecord/` folder from a previous session, an uploaded zip of it, or an Obsidian vault path they've mentioned before), read and update that one. Don't start a fresh one next to it.
- If no record exists yet, create it from `assets/` (copy all the template `.md` files into a new `MedicalRecord/` folder) and ask the user once where they'd like it kept going forward (downloaded each time vs. a synced location like Obsidian) — then proceed without asking again in future sessions unless they change their mind.
- Always deliver the **full updated folder** (zipped) after each update, not just the changed file, so the user always has one current copy.

## File map (see `assets/` for the empty templates and their exact section headers)

| File | Contents |
|---|---|
| `patient_profile.md` | Identity, chronic conditions, allergies, surgeries, family history, lifestyle factors |
| `analyzes_summary.md` | Lab results, one dated section per order/visit, tabular: показатель / результат / ед. изм. / референс / статус |
| `research_results.md` | Imaging, ultrasound, biopsy, and other non-lab studies |
| `doctors_reports.md` | Specialist consultations and their findings/recommendations |
| `patient_feelings.md` | Subjective symptoms and wellbeing, in the patient's own words |
| `medications.md` | Current vs. completed medications/supplements |
| `timeline.md` | Chronological log of all of the above, one line per event |
| `todo.md` | Outstanding medical tasks (tests still needed, decisions pending, follow-ups) |

## Core rules (from the record's own README — always follow these)

1. **Update, don't replace.** After processing each new document, update only the relevant file(s). Never delete existing entries without the user explicitly asking to remove them.
2. **Preserve chronology.** Every new data point gets a date and goes into `timeline.md` as well as its subject-specific file. Keep entries ordered oldest → newest within each file.
3. **Missing data.** Use the literal string `Нет данных` (or `No data` if the user's language is English) rather than leaving a field blank or guessing.
4. **Ambiguity → ask, don't guess.** If a document is unclear, contradicts an earlier entry, or a value can't be confidently read, ask the user a clarifying question rather than filling it in with a best guess — this is a medical record, wrong numbers are worse than missing ones.
5. **Don't invent precision.** If OCR/text extraction returns nothing (common with scanned/photographed documents — see below), do not fabricate plausible-looking values. Say so and ask the user to resend a clearer photo, one page at a time, in good light.

## Processing uploaded documents

1. **Inventory first.** List every file the user uploaded/referenced before opening any of them.
2. **Handle disguised archives.** Files with medical-lab-portal names (long numeric order IDs, `_e_a_k_`/`_e_a_l_` style suffixes, or just a `.pdf` extension from a phone export) are frequently **zip archives in disguise**, not real PDFs — `pdfinfo`/`pdftotext` will fail with a trailer/xref error. If a `.pdf` fails standard PDF tools, run `file <name>` — if it reports "Zip archive data", copy it to `<name>.zip`, unzip it, and read the `.txt` (extracted text) and `.jpeg` (page image) files inside, plus `manifest.json` for page structure. This is the normal case for Russian lab-portal exports (Гемотест, ДНКОМ, CMD, etc.), not an error.
3. **Real PDFs** (pdfinfo/pdftotext work normally): use standard PDF text extraction. See the `pdf-reading` skill if bundled/available for detailed technique guidance.
4. **Image-only pages** (empty `.txt`, or a photo/screenshot): view the image directly and transcribe what's legible. If handwriting or resolution makes specific numbers unreliable, flag it explicitly in the relevant file rather than guessing, and ask the user for a clearer copy.
5. **Extract structured lab data** into `analyzes_summary.md`: one dated section per order, one row per analyte, always including the reference range and a status flag (↑/↓/norm) so trends are visible at a glance.
6. **Cross-reference trends.** When a new lab result arrives for an analyte that already has prior entries, note the trend directly in the record (e.g., "ниже, чем в предыдущих анализах") — this is one of the main reasons to keep a longitudinal record rather than one-off summaries.
7. **Non-lab findings** (ultrasound, spermogram, biopsy, imaging) go in `research_results.md`; doctor consultations/diagnoses/recommendations go in `doctors_reports.md`; keep these distinct from raw lab numbers in `analyzes_summary.md`.

## After updating

Give the user a short summary of what changed (new values, new flags/abnormalities, anything that needs their input) — don't just say "done," since the point of the record is that they can act on the deltas. Then zip the updated folder and present it as a file.

## Privacy note

This is sensitive personal health data. Don't summarize or surface any of it in contexts where the user hasn't brought up their health themselves (see the general memory/wellbeing guidance on sensitive attributes) — this skill's job is purely to maintain the record when the user is actively working with it.
