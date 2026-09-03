# Skill: Generate ISO 29110 Repository Backup Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate a **Project Repository Backup (RPB)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file. This is a short, almost entirely human-specified document describing the **physical / offline backup** of the project's source code and documents. No codebase reading is required.

> **Note on language:** The HoneyComb sample was written in Thai. Ask the user whether the output should be in **English**, **Thai**, or another language and generate accordingly.

---

## What this document is

The Repository Backup document describes the **secondary offline backup** — separate from cloud-hosted storage (GitHub, Google Drive). It covers:
- What is backed up (source code + documents)
- How the backup is made (download ZIPs, copy files)
- How often (per release)
- Where the backup is stored (physical device)
- Who can access it
- How to recover from it

This document is intentionally short. Its value is in making the physical backup process explicit and auditable.

---

## Document dependencies (links this document contains)

| Placeholder location | What it links to | Shared context field |
|---|---|---|
| Backup method — source | GitHub / VCS repository (source of ZIPs) | `vcs.repositories` |
| Backup method — docs | Google Drive shared folder (source of documents) | none — ask user |

**Remind the user at the end of generation:**
> "Fill in:
> 1. **Repository URLs** — source of code ZIP downloads.
> 2. **Google Drive folder name/link** — source of document backups.
> 3. Reviewer / approver names in Document History."

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json`. Read it if present.
2. Relevant shared fields: `projectName`, `companyName`, `vcs.type`, `vcs.repositories`.
3. Ask only for empty shared fields, plus these RPB-specific items (ask every time):

```
A. What language should the document be written in? (English, Thai, other)

B. What is backed up?
   - Source code: downloaded as ZIP from {vcs.type} Release Tags? Other method?
   - Documents: downloaded from Google Drive / Confluence? Other method?
   - Name the Google Drive shared folder (or equivalent) where project documents live.

C. How often does the backup happen?
   (e.g. on every software release; nightly; manually by PM)
   Who is responsible? (e.g. Project Manager)

D. Where is the backup stored?
   (e.g. External hard disk — device name, brand, capacity)
   Physical location: (e.g. locked cabinet in office; CTO's desk drawer)

E. Who is allowed to access the backup?
   (e.g. only the Project Manager; read-only)
   Why is access restricted? (to prevent unauthorised modification)

F. Recovery process:
   How does someone retrieve the backup if the primary system (GitHub / Google Drive) fails?
   (e.g. go to the physical location, copy files to the primary system, verify integrity)
```

4. Merge shared field answers into `.iso29110-context.json` and save.

---

## Step 2 — Codebase cache

No codebase exploration is required for this document. Skip Step 2 entirely.

---

## Step 3 — Identify gaps

Any unanswered question → `[PLACEHOLDER]` in the document.

Remind the user:
> "Placeholders to fill in:
> - Repository URLs (backup source)
> - Google Drive / document storage folder name or link
> - Physical backup device details if left blank
> - Reviewer / approver names in Document History"

---

## Step 4 — Generate the document

### Document structure

```
Cover page
  Title: "Project Repository Backup"
  Sub-title: "for {ProjectName} Project"
  Company: "By {CompanyName}"
  Create By: ← blank
  Last Updated: {today dd/mm/yyyy}
  Version: V1.0

Page break

Document History table
  Columns: No. | Version | Action | By | Date | Status
  Row 1: 1 | V1.0 | Create this document | ← blank | {today} | Initial

Page break

Section heading: "{ProjectName} Project Repository Backup"   [HEADING_1, 14pt bold]

[Intro paragraph: "This document describes the backup of {projectName} — both source code
and project documents — to an external storage medium as a secondary backup, separate from
cloud storage ({vcs.type} and Google Drive), to prevent data loss from unexpected events."]

1. Backup Method                                             [HEADING_2, 13pt bold]
  Source Code: Download ZIP files of the project repository (all services — Frontend
  and Backend) from {vcs.type} via Release Tags, and store on the external backup device.
  
  Documents: Download copies of all baselined project documents from the shared
  Google Drive folder ([PLACEHOLDER — folder name]) and store on the external backup device.

2. Backup Frequency                                          [HEADING_2]
  Backups are created {answer C — frequency}.
  Responsible party: {answer C — role}

3. Storage Location                                          [HEADING_2]
  Device: {answer D — device name / brand / capacity}
  Physical location: {answer D — physical location}

4. Access and Permissions                                    [HEADING_2]
  The backup data on the external device is confidential and read-only.
  Authorised access: {answer E — who}
  Reason: {answer E — why restricted}

5. Recovery Process                                          [HEADING_2]
  If primary systems ({vcs.type} or Google Drive) are lost or corrupted:
  {Step 1 from answer F}
  {Step 2 from answer F}
  {Step 3: verify file integrity and version of recovered files}
```

### Styling rules
- **Font**: Calibri throughout.
- **Language**: use the language from Step 1 A.
- **Page size**: A4.
- **Section numbering**: sections are numbered **1–5** (standalone document). Do NOT use `3.x` — that was a legacy artifact from when this was chapter 3 of a combined template and has been corrected above.
- **Heading levels**: sections 1–5 map to `HEADING_2` (top-level sections); no sub-sections.
- **Body text**: Calibri 11pt, plain paragraphs — no bullet points in the original; use plain text for steps and descriptions.
- **Cover page / history**: Calibri, standard.
- **No logo**.
- **Author/reviewer fields**: blank.

### Running the script
```bash
node generate_rpb.js
```
Output: `{ProjectName}_Repository_Backup_V1.0.docx`

---

## Step 5 — Post-generation checklist

Remind the user to:
1. Fill in **author name** on cover page and Document History row 1.
2. Add **reviewer rows** when reviewed.
3. Replace **Google Drive folder name/link** in Section 3.1.
4. Confirm **device details and physical location** in Section 3.3.
5. Verify **recovery steps** in Section 3.5 reflect actual procedure.
