# Skill: Generate ISO 29110 Software Design Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate a **Software Design (SD)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file by reading the project codebase and asking the user for any context that cannot be derived from code.

---

## What this document is

The Software Design describes both the **high-level architecture** (component topology, relationships, design considerations) and the **detailed design** (sequence diagrams, data structures, program structure). It is the authoritative technical reference for the system as built.

ISO 29110 requires two layers:
- **Section 1 – Architectural / High-Level Design**: overall structure, component list, inter-component relationships, performance/interface/security/DB/error-handling considerations.
- **Section 2 – Low-Level / Detailed Design**: sequence diagrams per use-case, input/output data formats, data storage spec, naming conventions, schema examples, field definitions, program structure patterns.

---

## Document dependencies (links this document contains)

The SD document references the following external resources. Every reference is rendered as `[PLACEHOLDER — <description>]` during generation. After the human uploads all documents to their storage (e.g. Google Drive, Confluence), they must go back and replace each placeholder with the real URL.

| Placeholder location | What it links to | Shared context field |
|---|---|---|
| Section 2.1 — UX/UI Design | Figma (or equivalent design tool) file | `figmaUrl` |
| Section 3 — Deployment / Codebase | Source repository URL(s) | `vcs.repositories` |

**Remind the user at the end of generation:**
> "Two links need to be filled in once your files are in place:
> 1. **Figma link** (Section 2.1) — add after your design file is shared.
> 2. **Repository URL(s)** (Section 3) — add after confirming repo access settings.
> Both values will also be saved to `.iso29110-context.json` so other documents can reuse them."

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** defined in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json` in the working directory. Read it if present.
2. For the SD document, the relevant shared fields are: `projectName`, `companyName`, `systemDescription`, `techStack`, `figmaUrl`.
3. Ask the user **only** for shared fields that are still empty, plus the two SD-specific items below that are never stored in the shared file:

**SD-specific (ask every time — document-specific, not shared):**
```
A. User roles in the system (e.g. Admin, Affiliator, End User)
   and the top 3–5 key use-cases per role
   → used to populate the sequence diagram list in Section 2.1
B. Any major external services/integrations not obvious from code
   (e.g. payment gateways, third-party APIs, CDN providers)
```

4. After collecting answers, merge the shared fields back into `.iso29110-context.json` and save.

---

## Step 2 — Load codebase facts (cache first; explore only if cache is absent)

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part B before doing anything here.

- **Cache exists** → read the fields this document needs (`structure`, `components`, `techStack`, `database`, `api`, `security`, `performance`, `errorHandling`, `dataConventions`, `keySchemas`, `testing`, `deployment`) and skip the exploration checklist below entirely.
- **Cache absent** → run the checklist below, build the cache, then use it.
- **Never re-read the codebase** unless the human explicitly asks to rebuild the cache.

### Exploration checklist (only run when building the cache for the first time)

Work through the following in order. Each task maps to a section of the document.

### 2a. Overall structure (→ Section 1.1, 1.2)
- List the top-level directories and entry points (e.g. `apps/`, `packages/`, `src/`, `frontend/`, `backend/`).
- Identify whether it is a monorepo or separate repos.
- List every deployable application / service (e.g. API, Tracking, Queue Consumer, Cron, Frontend).
- List every shared library / package (e.g. auth lib, DB lib, logger).

### 2b. Component relationships (→ Section 1.3)
- In each application, look at its imports and dependency injection to determine which other apps or libs it depends on.
- Note which components expose HTTP/event interfaces and which consume them.
- Identify any message-queue or pub/sub patterns (e.g. Bull, RabbitMQ, Kafka).

### 2c. Design considerations (→ Section 1.4)
Extract evidence for each sub-section:

| Sub-section | Where to look |
|---|---|
| **Performance** | Redis/memcache setup, queue config, DB index scripts, load-balancer or K8s config |
| **Interface** | Controller decorators (`@Get`, `@Post`…), API versioning strategy, OpenAPI/Swagger setup, DTO files |
| **Security** | Auth guards, JWT strategy files, password-hashing libraries, CORS config, Helmet/security middleware, environment variable usage |
| **Database Design** | Schema/model files, migration files, index definitions; note the DB engine and major collections/tables |
| **Error Handling & Recovery** | Exception filter files, custom error/exception classes, retry logic, logging setup |

### 2d. Detailed design inputs (→ Section 2)
- **DTOs / request-response schemas** → Section 2.2 (input/output format)
- **Schema/model files** → Section 2.3 (data storage), 2.5 (example structure), 2.6 (field list)
- **Naming patterns** across schema fields, DTO properties, and TypeScript types → Section 2.4
- **Module/controller/service/guard/middleware file structure** → Section 2.7
- **Test files** (`*.spec.ts`, `*.test.ts`, Playwright configs) → Section 3 (testing)
- **Dockerfile / docker-compose / K8s manifests** → Section 3 (deployment)

---

## Step 3 — Identify gaps and ask the user

After codebase exploration, compile a list of anything you could not determine:
- Sequence diagrams cannot be auto-generated — confirm the use-case list from Step 1 and tell the user the agent will write **text descriptions** of each flow; actual diagrams must be added manually or linked from a design tool.
- Any external service details not found in code.
- Any intentionally undocumented design decisions.

Remind the user:
> "The following items will be left as placeholders in the document — please fill them in after generation:
> - UX/UI Figma link (Section 2.1)
> - GitHub repository URLs (Section 3)
> - Actual sequence diagram images (Section 2.1 sub-sections)
> - Reviewer / approver names in Document History"

---

## Step 4 — Generate the document

Use the `docx` npm package (preinstalled; `require('docx')` directly). Write a Node.js script and run it with `node`.

### Document structure to implement

```
Cover page
  Title: "Software Design"
  Sub-title: "for {ProjectName} Project"
  Company line: "By {CompanyName}"
  (Repeat title block a second time — this is a style convention)
  "Create By: " ← leave blank
  "Last Updated: {today's date dd/mm/yyyy}"
  "Version: V1.0"

Page break

Document History table
  Columns: No. | Version | Action | By | Date | Status
  Row 1: 1 | V1.0 | Create this document | ← blank | {today} | Initial
  (Leave subsequent rows blank for reviewers to fill)

Page break

Section heading: "{ProjectName} Software Design"

1. Architectural or High-Level Software Design
  1.1 Overall Software Structure
      [narrative paragraph describing monorepo/repo structure, tech stack, design goals]
      High-Level Architecture
      [bulleted list of top-level apps/services]

  1.2 Required Software Components
      [bulleted list: ComponentName: one-sentence responsibility]

  1.3 Relationship Between Components
      [bulleted list of inter-component relationships and data flows]

  1.4 Design Considerations
      Performance
        [bullet points from 2c: caching, queuing, scaling]
      Interface
        [bullet points: API style, versioning, validation, documentation]
      Security
        [bullet points: auth mechanism, access control, hashing, headers, CORS, secrets]
      Database Design Requirements
        [DB engine, list of collections/tables, indexing notes]
      Error Handling and Recovery
        [exception handling pattern, error codes, logging, retry strategies]

2. Low-Level or Detailed Software Design
  2.1 Detailed Design (Sequence Diagram)
      UX/UI Design: [PLACEHOLDER — add Figma link]
      [Group use-cases by role, numbered SD-M1, SD-M2, … per module/role]
      [Each use-case: SD-Mn-k – {Actor} {action}]
      Note: Sequence diagram images must be inserted manually.

  2.2 Format of Input/Output Data
      [request payload format (e.g. JSON), response format, key headers used]

  2.3 Data Storage Specification
      [DB engine + version, key collections/tables, index strategy, cache layer]

  2.4 Data Naming Conventions
      [field naming per layer: DB fields, TS variables, collections, file names]

  2.5 Format of Data Structure
      [one example schema with representative fields in code block]

  2.6 Data Fields and Data Elements
      [per entity: name + key fields listed]

  2.7 Specifications of Program Structure
      [patterns used: Module, Controller, Service, Guard, Middleware, Exception Filter]

3. Additional Design Considerations
      Extensibility: [how the architecture supports adding features/services]
      Testing: [test frameworks, test types, CI gate]
      Deployment: [containerisation, orchestration, config management]
```

### Styling rules (match existing document style)
- **Font**: Calibri throughout. Every `TextRun` must set `font: 'Calibri'`. Exception: code/JSON blocks use `'Courier New'` (see `iso29110-docx-rules.md` Rule 3).
- **Page size**: A4 (default in docx-js).
- **Heading 1** (`1.`, `2.`, `3.`): Calibri bold, 13pt (`size: 26`), `HeadingLevel.HEADING_2`.
- **Heading 2** (`1.1`, `1.2`, …): Calibri bold, 12pt (`size: 24`), `HeadingLevel.HEADING_3`.
- **Heading 3** (`1.1.1`, …): Calibri bold, 11pt (`size: 22`), `HeadingLevel.HEADING_4`.
- **Body text**: Calibri, 11pt (`size: 22`), normal weight.
- **Document History table**: full-width, light grey header row, 1pt borders.
- **Cover page**: centred layout, Calibri; title at 14pt bold (`size: 28`), sub-title at 13pt (`size: 26`), company at 11pt (`size: 22`).
- **No logo** — omit any image placeholder.
- **Author/reviewer fields**: leave as empty strings so team members can fill in.

### Running the script
```bash
node generate_sd.js
```
Output file: `{ProjectName}_Software_Design_V1.0.docx` in the current directory.

---

## Step 5 — Post-generation checklist (tell the user)

After the file is generated, remind the user to:

1. **Fill in author name** in the cover page and Document History row 1.
2. **Add reviewer rows** to Document History when the document is reviewed.
3. **Insert sequence diagram images** (or embed Figma link) in Section 2.1.
4. **Add GitHub / repository URLs** in Section 3.
5. **Verify all external service descriptions** in Sections 1.3 and 1.4 are accurate.
6. **Update "Last Updated" date** after any revision.

---

## Notes

- If the project has separate frontend and backend repos, read both.
- If there is no dedicated security middleware file, infer security approach from auth guards and environment variable usage.
- If the codebase has no tests yet, note "Unit and integration tests are planned" in Section 3.
- Do not invent business logic or data field meanings — if uncertain, ask the user or leave a `[TODO: verify]` note in the document.
