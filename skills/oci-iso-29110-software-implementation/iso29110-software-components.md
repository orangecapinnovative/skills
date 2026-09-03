# Skill: Generate ISO 29110 Software Components Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate a **Software Components (SC)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file by reading the project codebase and asking the user for any context that cannot be derived from code.

---

## What this document is

The Software Components document provides a **structured inventory of every significant software unit** in the system — both physically deployable applications and logical in-process modules. It is written at a level of abstraction that helps new team members, auditors, and maintenance personnel understand the system's modular structure without reading source code.

ISO 29110 requires this document to cover:
- **Component identification**: name, type (physical or logical), interface/encapsulation style.
- **Responsibilities**: what each component owns or does.
- **Interconnections**: which other components it depends on or is consumed by.
- **Grouping**: how logical components are packaged into physical deployables.
- **Technical references**: pointers to the Software Design document, Implementation Environment document, source repositories, and test coverage.

---

## Document dependencies (links this document contains)

The SC document references the following external resources. Every reference is rendered as `[PLACEHOLDER — <description>]` during generation. After the human uploads all documents to their storage (e.g. Google Drive, Confluence), they must go back and replace each placeholder with the real URL.

| Placeholder location | What it links to | Shared context field |
|---|---|---|
| Technical Design — Software Design | Software Design (SD) document | `documentRefs.softwareDesign` |
| Technical Design — Implementation Environment | Implementation Environment (IE) document | `documentRefs.implementationEnvironment` |
| Codebase and Code Repository | Source repository URL(s) | `vcs.repositories` |
| Unit Testing — coverage report(s) | Per-component test coverage report | `testCoverageUrls` |

**Remind the user at the end of generation:**
> "Four sets of links need to be filled in once your files are in place:
> 1. **Software Design document link** (Technical Design section) — add after SD is uploaded.
> 2. **Implementation Environment document link** (Technical Design section) — add after IE is uploaded.
> 3. **Repository URL(s)** (Codebase section) — add after confirming repo access.
> 4. **Coverage report URL(s)** (Unit Testing section) — add after CI/CD publishes reports.
> All values are also saved to `.iso29110-context.json` so future documents can reuse them."

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** defined in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json` in the working directory. Read it if present.
2. For the SC document, the relevant shared fields are: `projectName`, `companyName`, `systemDescription`, `vcs.type`, `vcs.repositories`, `vcs.accessControl`, `documentRefs.softwareDesign`, `documentRefs.implementationEnvironment`, `testCoverageUrls`.
3. Ask the user **only** for shared fields that are still empty. Accept "I don't know / leave blank" — store `""` in the JSON and render `[PLACEHOLDER]` in the document.

**No SC-specific questions are needed** — the component table and groupings are derived entirely from the codebase in Step 2.

4. After collecting answers, merge the shared fields back into `.iso29110-context.json` and save.

---

## Step 2 — Load codebase facts (cache first; explore only if cache is absent)

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part B before doing anything here.

- **Cache exists** → read `structure`, `components`, and `testing` from it. Skip the exploration checklist below entirely.
- **Cache absent** → run the checklist below, build the cache, then use it.
- **Never re-read the codebase** unless the human explicitly asks to rebuild the cache.

### Exploration checklist (only run when building the cache for the first time)

### 2a. Identify physical components (→ Components table, rows where Type = Physical)
A **physical** component is a separately deployable runtime unit (a process, a container, a service).

Look for:
- `apps/` or `services/` directories in a monorepo — each sub-folder is typically a physical component.
- `Dockerfile` or `docker-compose.yml` — each `service:` block is a physical component.
- `package.json` scripts with `start`, `start:prod` — each distinct entry point is a physical component.
- Kubernetes manifests (`Deployment` resources) — each deployment is a physical component.
- Separate repositories (frontend vs backend) — each repo root is a physical component.

For each physical component note:
- **Name** (use the folder/service name, capitalised readably, e.g. "API Service", "Frontend App").
- **Encapsulation/Interface style** (e.g. REST API, GraphQL API, Queue listener, Cron scheduler, Web app).
- **Responsibilities** (infer from folder name, README if present, or entry-point file).
- **Interconnections** (what it calls or is called by — check imports, HTTP clients, queue producers/consumers).

### 2b. Identify logical components (→ Components table, rows where Type = Logical)
A **logical** component is a meaningful in-process module that does not run as its own process but has a clear boundary and responsibility.

Look for:
- `libs/` or `packages/` in a monorepo — each shared library.
- Dedicated directories inside an app: `auth/`, `database/`, `logger/`, `utils/`, `components/`, `hooks/`, `contexts/`, `api/` (frontend API layer).
- Named NestJS modules, Next.js route groups, or similar framework-level groupings.

For each logical component note the same four fields (Name, Encapsulation, Responsibilities, Interconnections).

### 2c. Determine grouping structure (→ narrative sections after the table)
Map every logical component to the physical component that contains it. For example:
- "Frontend App (physical)" contains: API Layer, UI Components, React Contexts, React Hooks, Utilities/Constants/Styles.
- "API Service (physical)" contains: Auth lib, DB lib, Logger lib, etc.

### 2d. Locate test evidence (→ Unit Testing section)
- Find test files (`*.spec.ts`, `*.test.ts`, `__tests__/`).
- Identify the test framework (Jest, Vitest, Playwright, etc.).
- Note whether there are CI gates on test pass.
- Note where coverage reports are published (from the user's answers in Step 1).

---

## Step 3 — Identify gaps and ask the user

After exploration, compile anything you cannot determine from the code:
- Component names or responsibilities that are ambiguous (e.g. a folder called `misc/`) → ask.
- Whether a component is physical or logical when it's not clear from the repo structure → ask.
- Any component the user mentioned in Step 1 that you cannot find in the codebase → ask.

Remind the user:
> "The following items will be left as placeholders — please fill them in after generation:
> - Software Design document link (Section: Technical Design)
> - Implementation Environment document link (Section: Technical Design)
> - Repository URLs (Section: Codebase and Code Repository)
> - Test coverage report URLs (Section: Unit Testing)
> - Reviewer / approver names in Document History"

---

## Step 4 — Generate the document

Use the `docx` npm package (preinstalled; `require('docx')` directly). Write a Node.js script and run it with `node`.

### Document structure to implement

```
Cover page
  Title: "Software Components"
  Sub-title: "for {ProjectName} Project"
  Company line: "By {CompanyName}"
  (Repeat title block a second time — style convention)
  "Create By: " ← leave blank
  "Last Updated: {today's date dd/mm/yyyy}"
  "Version: V1.0"

Page break

Document History table
  Columns: No. | Version | Action | By | Date | Status
  Row 1: 1 | V1.0 | Create this document | ← blank | {today} | Initial

Page break

Section heading: "{ProjectName} Software Components"

Introduction
  [One or two paragraphs]
  Paragraph 1: State the document's purpose — to identify and define key software components
  in accordance with ISO 29110-5-1-2:2025. Mention the system's one-sentence description.
  Paragraph 2: Explain that the document covers module responsibilities, technical overview
  (architecture, design, code), and testing.

Components
  [Introductory sentence: "The following table summarises the main software components..."]
  
  Components table — full-width, columns:
    Component Name | Type | Encapsulation/Interface | Responsibilities | Interconnections

  [One row per component — physical components first, then logical]

  [After the table, a brief narrative paragraph:]
  "In the actual codebase, components are grouped into {N} deployable units: {list}."

  [Then, for each physical component, a sub-heading and a bullet list of its logical children:]
  {Physical Component Name}
  Inside {Physical Component Name} the following logical components are included:
    • {Logical Component 1}
    • {Logical Component 2}
    …

Technical Design
  The details of the technical design are described in the following documents.

  Software Design
    [One sentence on what the SD document covers]
    Document reference: [PLACEHOLDER — add link after document is available]

  Implementation Environment
    [One sentence on what the IE document covers]
    Document reference: [PLACEHOLDER — add link after document is available]

  Codebase and Code Repository
    Main repository: GitHub (or other VCS name)
      {Repo 1 name}: [PLACEHOLDER — add URL]
      {Repo 2 name}: [PLACEHOLDER — add URL]
    Access control for {VCS name}
      Account type: {answer from Step 1 q7}
      Permission: {answer from Step 1 q7 — e.g. "Granted by Organisation admin account"}

Unit Testing
  [One paragraph explaining that critical parts are unit-tested to ensure integrity and
  correctness, that unit tests verify code functionality separately from use-case tests,
  and that all tests must PASS before deployment.]
  
  You can find the unit test coverage reports in the following reference:
    {Component 1}: [PLACEHOLDER — add URL]
    {Component 2}: [PLACEHOLDER — add URL]
```

### Styling rules (match existing document style)
- **Font**: Calibri throughout. Every `TextRun` must set `font: 'Calibri'`. Exception: code/JSON blocks use `'Courier New'` (see `iso29110-docx-rules.md` Rule 3).
- **Page size**: A4.
- **Section headings** ("Components", "Technical Design", "Unit Testing"): Calibri bold, 13pt (`size: 26`), `HeadingLevel.HEADING_2`.
- **Sub-headings** (physical component names in the grouping narrative): Calibri bold, 12pt (`size: 24`), `HeadingLevel.HEADING_3`.
- **Body text**: Calibri, 11pt (`size: 22`), normal weight.
- **Components table**: full-width, header row in light grey, 1pt borders, `Responsibilities` column is widest.
- **Document History table**: full-width, light grey header, 1pt borders.
- **Cover page**: centred, Calibri; title 14pt bold (`size: 28`), sub-title 13pt (`size: 26`), company 11pt (`size: 22`).
- **No logo** — omit any image placeholder.
- **Author/reviewer fields**: leave as empty strings.

### Running the script
```bash
node generate_sc.js
```
Output file: `{ProjectName}_Software_Components_V1.0.docx`

---

## Step 5 — Post-generation checklist (tell the user)

After the file is generated, remind the user to:

1. **Fill in author name** on the cover page and in Document History row 1.
2. **Add reviewer/approver rows** to Document History when reviewed.
3. **Insert document links** in the Technical Design section (Software Design, Implementation Environment).
4. **Insert repository URLs** in the Codebase and Code Repository sub-section.
5. **Insert test coverage report URLs** in the Unit Testing section.
6. **Verify responsibilities and interconnections** for each component — the agent's descriptions are inferred from file structure and may not capture all business rules.

---

## Notes

- Distinguish **Type** carefully: a component is **Physical** if it runs as its own OS process / container; it is **Logical** if it exists only as importable code within another process.
- If the project has no shared libraries, the logical components list will consist of in-app modules only — that is fine.
- If there are no tests yet, write "Unit tests are planned but not yet implemented" in the Unit Testing section — do not leave the section blank.
- Do not invent responsibilities or interconnections. If a component's purpose is unclear from the code and the user cannot clarify, note `[Verify: purpose unclear]` as the responsibility.
- Keep the Components table rows concise — one sentence each in the Responsibilities and Interconnections columns. Detailed design belongs in the Software Design document.
