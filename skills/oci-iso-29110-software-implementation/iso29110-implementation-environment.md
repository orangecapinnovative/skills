# Skill: Generate ISO 29110 Implementation Environment Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate an **Implementation Environment (IE)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file. This document is almost entirely codebase-derivable — dependency lists, framework versions, and deployment configs come from files in the repo, not from memory.

---

## What this document is

The Implementation Environment describes the **technical setup used to build, run, and test the software** — frameworks, libraries with pinned versions, runtime environments, deployment platform, development tooling, and testing infrastructure. It is the reference that lets a new developer reproduce the environment from scratch.

---

## Document dependencies (links this document contains)

| Placeholder location | What it links to | Shared context field |
|---|---|---|
| Section 2 — Frontend / Backend repos | Source repository URLs | `vcs.repositories` |
| Section 3 — Development environment | Design tool (e.g. Figma) | `figmaUrl` |

**Remind the user at the end of generation:**
> "Two sets of links need to be filled in:
> 1. **Repository URLs** (Sections 2 & 3) — add after confirming repo access.
> 2. **Design tool link** (Section 3) — add after your design file is shared."

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json`. Read it if present.
2. Relevant shared fields: `projectName`, `companyName`, `systemDescription`, `techStack`, `vcs.repositories`, `figmaUrl`.
3. Ask only for empty shared fields, plus these IE-specific items (ask every time — not stored):

```
A. What operating systems do developers use? (e.g. macOS, Windows, Linux)
B. What IDEs / editors does the team use? (e.g. VS Code, Cursor, IntelliJ)
C. What package manager is used? (npm, yarn, pnpm, etc.)
D. How is the local development environment set up?
   (e.g. port-forwarding to cloud DB, local Docker Compose, mocked services)
E. What are the staging / UAT environment details?
   (same config as prod? separate GCP project? any differences?)
F. What CI/CD tool runs tests before deployment? (e.g. GitHub Actions, Playwright script)
```

4. Merge shared answers into `.iso29110-context.json` and save.

---

## Step 2 — Load codebase facts (cache first; explore only if cache is absent)

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part B.

- **Cache exists** → read `techStack`, `deployment`, `testing` and skip exploration below.
- **Cache absent** → run the checklist; build the cache; continue.
- **Never re-read** unless the human explicitly asks to rebuild.

### Exploration checklist (only when building cache)

| What to find | Where to look |
|---|---|
| Frontend framework & version | `package.json` → `dependencies.next` / `react` / etc. |
| Frontend runtime requirements | `.nvmrc`, `.node-version`, or `engines` in `package.json` |
| Frontend npm dependencies (prod) | `package.json` → `dependencies` — list name: version |
| Frontend npm dependencies (dev) | `package.json` → `devDependencies` — list name: version |
| Backend framework & version | `package.json` in backend app |
| Backend runtime | Node.js version in Dockerfile (`FROM node:X.Y.Z`) |
| Backend npm dependencies (prod & dev) | `package.json` in each backend service |
| Database engine & version | ORM package (mongoose version → MongoDB; prisma schema → engine) |
| Cache layer | `ioredis`, `redis` packages; Redis version in Docker |
| Deployment method | `Dockerfile`, `docker-compose.yml`, K8s manifests, Helm charts |
| Cloud provider | CI config files, SDK packages (`@google-cloud/*`, `aws-sdk`, etc.) |
| Testing framework | `jest`, `vitest`, `playwright` in devDependencies; jest config |
| Test types present | Presence of `*.spec.ts`, `e2e/`, `playwright.config.ts` |
| IaC tool | `helm/`, `terraform/`, `k8s/` directories |

---

## Step 3 — Identify gaps

After reading the cache, check for any fields still missing. For dependency lists, always read the actual `package.json` directly — do not rely on cached summaries for version numbers, as these change frequently.

Remind the user:
> "The following will be `[PLACEHOLDER]` in the document — fill in after files are set up:
> - Repository URLs (Sections 2 & 3)
> - Design tool link (Section 3)
> - Reviewer / approver names in Document History"

---

## Step 4 — Generate the document

### Document structure

```
Cover page
  Title: "Implementation Environment"
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

Section heading: "{ProjectName} Implementation Environment"   [HEADING_1, 14pt bold]

1. Introduction                                              [HEADING_2, 13pt bold]
   [One paragraph: this document presents the implementation environment
    for {ProjectName}, outlining frameworks, libraries, deployment method,
    and deployment platform used by the development team.]

2. Frontend                                                  [HEADING_2]
   Framework: {framework name and version}
   Deployment: {runtime image} managed by {orchestration}, {cloud provider}
   
   Key Libraries/Technologies:                              [HEADING_3]
   {runtime}: {version range} ([PLACEHOLDER — download link])
   {package manager}: {version range}
   
   NPM Dependencies                                         [HEADING_3]
   [One line per dependency: {package}: {version}]
   — read exact versions from package.json
   
   Development Dependencies                                 [HEADING_3]
   [One line per dev dependency: {package}: {version}]

3. Backend                                                   [HEADING_2]
   Framework: {framework name}
   Programming Language: TypeScript
   Database: {DB engine}, {cache}
   Cache: {cache engine}
   
   [For each backend service:]
   {Service Name}:                                          [HEADING_3]
     Purpose: {one sentence}
     Deployment: {runtime image} managed by {orchestration}, {cloud}
     Endpoints: [PLACEHOLDER — add production URL]
   
   Key Libraries/Dependencies:                              [HEADING_3]
   {runtime}: {version range} ([PLACEHOLDER — download link])
   {package manager}: {version range}
   
   NPM Dependencies                                         [HEADING_3]
   [Full dependency list from package.json]
   
   NPM Development Dependencies                             [HEADING_3]
   [Full dev dependency list]
   
   [If IaC exists:]
   Infrastructure as Code                                   [HEADING_3]
     Repository: [PLACEHOLDER — add URL]
     Framework: {Helm / Terraform / etc.}
     Deployment platform: {cloud} using {orchestration}

4. Development and Testing Environment                       [HEADING_2]
   Development Environment:                                  [HEADING_3]
     Operating Systems: {answer from Step 1 A}
     IDEs: {answer from Step 1 B}
     Version Control: Git ({vcs.type})
     Package Managers: {answer from Step 1 C}
     Compilers/Transpilers: {TypeScript compiler, bundler}
     Design Tools: {figmaUrl if set, else [PLACEHOLDER]}
     Local Development Setup: {answer from Step 1 D}
   
   Testing Environment:                                     [HEADING_3]
     Testing Frameworks: {from cache — e2eFramework, unitFramework}
     Testing Types: {unit / integration / e2e / manual — from test files found}
     Test Data Management: {answer from Step 1 F or "Manually set up by Test Cases"}
     CI/CD Pipeline for Testing: {answer from Step 1 F}
     Staging Environment: {answer from Step 1 E}
```

### Styling rules
- **Font**: Calibri throughout; Courier New only for dependency version strings in code context if rendered as code blocks.
- **Page size**: A4.
- **Heading levels**: follow the table in `iso29110-docx-rules.md` Rule 6.
- **Dependency lists**: render as plain `Paragraph` lines (`{package}: {version}`), not as bullet points — this matches the original document style.
- **Cover page**: Calibri; title 14pt bold, sub-title 13pt, company 11pt, centred.
- **No logo** — omit any image placeholder.
- **Author/reviewer fields**: blank.

### Running the script
```bash
node generate_ie.js
```
Output: `{ProjectName}_Implementation_Environment_V1.0.docx`

---

## Step 5 — Post-generation checklist

Remind the user to:
1. Fill in **author name** on cover page and Document History row 1.
2. Add **reviewer rows** to Document History when reviewed.
3. Replace all **`[PLACEHOLDER]`** entries: repository URLs, download links, endpoints, design tool link.
4. **Verify dependency versions** — the document captures versions at time of generation; update on each major release.
