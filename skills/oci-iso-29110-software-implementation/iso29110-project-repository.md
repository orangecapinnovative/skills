# Skill: Generate ISO 29110 Project Repository Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate a **Project Repository (RP)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file. This document is a **policy document** about how the project's source code and documentation are organised, versioned, backed up, restored, and stored. Almost no content comes from reading code — it comes from the team's policies and storage decisions.

> **Note on language:** The HoneyComb sample was written in Thai. Ask the user whether the output should be in **English** or **Thai** (or another language) and generate accordingly.

---

## What this document is

The Project Repository document covers:
1. **Main repository** — where source code lives and how to access it.
2. **Access control** — who has what level of access.
3. **Backup policy** — when backups happen, per service/module versioning strategy.
4. **Restoration plan** — definitions, trigger conditions, rollback strategy, service-specific rollback steps.
5. **Document management** — how project documents are categorised, where they are stored (e.g. Google Drive folders), and the update policy.

---

## Document dependencies (links this document contains)

| Placeholder location | What it links to | Shared context field |
|---|---|---|
| Section 1 — repos | Source repository URLs | `vcs.repositories` |
| Section 5 — document storage | Google Drive / Confluence folder links | none — ask user |

**Remind the user at the end of generation:**
> "Fill in after storage is set up:
> 1. **Repository URLs** (Section 1) — GitHub / GitLab links per service.
> 2. **Document folder links** (Section 5) — Google Drive or Confluence folder URLs per document category."

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json`. Read it if present.
2. Relevant shared fields: `projectName`, `companyName`, `vcs.type`, `vcs.repositories`, `vcs.accessControl`.
3. Ask only for empty shared fields, plus these RP-specific items (ask every time):

```
A. What language should the document be written in? (English, Thai, other)

B. What roles exist in the team, and what repository access level does each role have?
   (e.g. Developer/QA → Read/Write; PM → Read/Review; Admin → full access)

C. What is the backup strategy for source code?
   - When does a backup happen? (e.g. on every release, nightly, manually)
   - Is backup triggered per-service or for all services together?
   - What versioning scheme is used? (e.g. one version tag per frontend;
     separate tags per backend service: API, Tracking, Cron, Queue Consumer)
   - Where is the backup stored? (e.g. GitHub Release + ZIP on Google Drive)

D. What are the conditions that trigger a code restoration / rollback?
   (e.g. "critical defect in new release affecting core functionality")

E. Who has authority to decide on a rollback?
   (e.g. tech lead, DevOps lead, project manager)

F. What is the rollback mechanism?
   (e.g. helm rollback; docker image tag rollback; git revert + redeploy)

G. What document categories does the project use, and where is each stored?
   (Common: Project Management, Requirements, Design, Implementation/Tests,
    Deployment/Operations, Quality Management)
   For each category: name + storage location (Google Drive folder name or URL)

H. How often must technical documents be updated?
   (e.g. "every time related code changes; the developer making the change is responsible")
```

4. Merge shared field answers into `.iso29110-context.json` and save.

---

## Step 2 — Codebase cache

No codebase exploration is needed for this document. Skip Step 2 entirely.

The only cache field used is `structure.apps` (to list the backend services in the backup policy section). If the cache exists, read it. If not, ask the user to list the backend service names.

---

## Step 3 — Identify gaps

Any unanswered question from Step 1 becomes a `[PLACEHOLDER]` in the document.

Remind the user:
> "Placeholders to fill in:
> - Repository URLs (Section 1)
> - Document storage folder links (Section 5)
> - Reviewer / approver names in Document History"

---

## Step 4 — Generate the document

### Document structure

```
Cover page
  Title: "Project Repository"
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

Section heading: "{ProjectName} Project Repository"         [HEADING_1, 14pt bold]

[Intro paragraph: "This document summarises the repositories used for {projectName}
and access control. It describes how the development team can access the repositories."]

1. Main Repository                                          [HEADING_2, 13pt bold]
   Repository: {vcs.type}
   [For each repo in vcs.repositories:]
   {RepoName}: [PLACEHOLDER — {URL or "add URL"}]
   
   [If repos not yet in context file:]
   [PLACEHOLDER — add repository URLs after they are confirmed]

2. Access Control                                           [HEADING_2]
   Access control for {vcs.type}
   Account type: {vcs.type} Account
   [For each role from answer B:]
   {Role}: {access level}
   Permission: {vcs.accessControl}

3. Backup Policy                                            [HEADING_2]
   [Narrative from answer C, covering:]
   - When backups are created (trigger event)
   - Versioning scheme per service:
     {Frontend name}: {versioning description}
     {Backend services}: {per-service or combined versioning}
   - Backup storage location(s): {answer C — storage}
   - Who is responsible: {role}

4. Restoration Plan                                         [HEADING_2]
   [Intro: "This plan applies to all production code repositories for critical services."]
   
   Definitions                                              [HEADING_3]
   Major Issue: {define — critical defect causing service disruption or data loss}
   Stable Version: {define — last released, verified, working version}
   Changelog: {define — record of changes per release, used to identify rollback targets}
   Repository: {define — centralised code storage (Git)}
   
   Conditions for Restoration                               [HEADING_3]
   [List conditions from answer D — when rollback is triggered]
   - A new version has been deployed to production.
   - The release has a major issue preventing core functionality.
   - The impact is assessed as critical, requiring immediate rollback rather than hotfix.
   - The rollback decision is made by {answer E}.
   
   Restoration Strategy                                     [HEADING_3]
   [Describe the rollback mechanism from answer F:]
   - Prioritisation: roll back to the last confirmed working version per service.
   - Changelog reference: consult changelogs to identify the exact commit/tag if unclear.
   - Service-specific rollback: performed per service independently.

5. Document Management                                      [HEADING_2]
   
   5.1 Document Types and Storage                           [HEADING_3]
   [For each category from answer G:]
   {Category Name}:
     Examples: {examples}
     Storage: [PLACEHOLDER — {folder name or "add Google Drive link"}]
   
   5.2 Document Update Policy                               [HEADING_3]
   Update frequency: {answer H}
   Responsible party: {the developer making the code change, or as specified}
```

### Styling rules
- **Font**: Calibri throughout.
- **Language**: use the language from Step 1 A for all body text. Section headings and cover page follow the same language.
- **Page size**: A4.
- **Heading levels**: follow `iso29110-docx-rules.md` Rule 6.
- **Role/access lines**: plain body paragraphs (`{Role}: {access level}`), not table rows.
- **Definition items**: bold the term, normal weight the definition — `{Term}: {definition}`.
- **Condition/strategy lists**: dash-prefixed paragraphs (Rule 5 in shared rules).
- **Cover page / history**: Calibri, standard.
- **No logo** — omit any image placeholder.
- **Author/reviewer fields**: blank.

### Running the script
```bash
node generate_rp.js
```
Output: `{ProjectName}_Project_Repository_V1.0.docx`

---

## Step 5 — Post-generation checklist

Remind the user to:
1. Fill in **author name** on cover page and Document History row 1.
2. Add **reviewer rows** when reviewed.
3. Replace **repository URLs** in Section 1.
4. Replace **document folder links** in Section 5.1.
5. Verify the **restoration conditions and strategy** in Section 4 reflect actual team policy.
