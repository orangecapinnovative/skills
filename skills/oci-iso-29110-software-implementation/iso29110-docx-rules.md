# ISO 29110 — Shared `.docx` Generation Rules

**Every ISO 29110 document generation skill must load this file before writing any `docx`-js script.**  
Follow Part A (shared context) first, then Part B (rendering rules) when writing the generation script.

---

## Part A — Shared Project Context

### Why this exists

Multiple ISO 29110 work products ask for the same project-level information (project name, company, repo URLs, system description, etc.). Without a shared store, the user gets asked the same questions repeatedly across separate sessions or agents. This pattern eliminates that by writing answers to a single JSON file on first use and reading from it on every subsequent use.

### The shared context file

**Filename:** `.iso29110-context.json`  
**Location:** The directory where `.docx` files are being generated (i.e., the user's working directory for this project's ISO documentation).

#### Schema

```json
{
  "projectName": "",
  "companyName": "",
  "systemDescription": "",
  "vcs": {
    "type": "GitHub",
    "repositories": {
      "frontend": "",
      "backend": "",
      "_note": "Add keys for any additional repos (iac, sdk, docs, etc.) as needed"
    },
    "accessControl": ""
  },
  "techStack": {
    "frontend": "",
    "backend": "",
    "database": "",
    "infrastructure": ""
  },
  "documentRefs": {
    "softwareDesign": "",
    "softwareComponents": "",
    "implementationEnvironment": "",
    "maintenanceDocument": "",
    "operationGuidelines": "",
    "projectRepository": "",
    "repositoryBackup": "",
    "sdkUserGuide": ""
  },
  "infra": {
    "deploymentTool": "",
    "helmRepository": "",
    "rollbackMechanism": ""
  },
  "testCoverageUrls": {
    "frontend": "",
    "backend": ""
  },
  "figmaUrl": ""
}
```

#### Which fields each document type uses

| Field | SD | SC | IE | MD | OM | RP | RPB | SDK |
|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| projectName | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| companyName | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| systemDescription | ✓ | ✓ | ✓ | ✓ | ✓ | | | ✓ |
| vcs.repositories | ✓ | ✓ | ✓ | | ✓ | ✓ | ✓ | ✓ |
| vcs.accessControl | | ✓ | | | | ✓ | ✓ | ✓ |
| techStack | ✓ | | ✓ | ✓ | ✓ | | | |
| documentRefs | | ✓ | ✓ | | | | | |
| infra.deploymentTool | ✓ | | ✓ | ✓ | ✓ | ✓ | | |
| infra.helmRepository | ✓ | | ✓ | ✓ | ✓ | ✓ | | |
| infra.rollbackMechanism | | | | | ✓ | ✓ | | |
| testCoverageUrls | | ✓ | | ✓ | | | | |
| figmaUrl | ✓ | | | | | | | |

### Agent protocol — read → ask only gaps → write

**At the start of every ISO 29110 document generation session:**

```
1. Check if .iso29110-context.json exists in the working directory.
   - If yes: read it. Pre-fill all answers from it. Skip asking those questions.
   - If no: all fields are empty — proceed to ask.

2. Identify which fields THIS document type needs (see table above).
   Among those, identify which are still empty (either file missing or field blank).

3. SPECIAL RULE — Repository URLs (vcs.repositories):
   If this document type uses vcs.repositories (see table — SD, SC, IE, OM, RP, RPB, SDK),
   ask for every repository URL that is still empty BEFORE starting generation.
   Do not defer repository URL collection to a "fill in later" placeholder.
   Collect them now, add them to the context, and embed the real URLs in the document.

   Prompt to use:
     "This document links to source repositories. Please provide the URL for each:
      - Frontend repository URL:
      - Backend repository URL:
      (Add any additional repos — IaC, SDK docs, etc. — if they apply.)"

   If the user cannot provide a URL yet, store "" and render [PLACEHOLDER] in the doc
   as a last resort, but prefer asking again after confirming the repo exists.

3b. SPECIAL RULE — Infrastructure / Deployment / Rollback (infra):
   If this document covers infrastructure, deployment, or rollback content
   (see table — SD, IE, MD, OM, RP), ask for any empty infra fields BEFORE generation.
   Do not assume the deployment tool or rollback mechanism from the codebase cache —
   the cache captures what packages are present, not the team's actual operational process.

   Ask:
     "A. What tool does the team use to deploy the application?
         (e.g. Helm, Terraform, kubectl directly, CI/CD pipeline — describe the process briefly)
      B. What is the rollback mechanism when a deployment goes wrong?
         (e.g. helm rollback <release>, revert git tag + redeploy, manual image tag switch)"

   If the answer to A mentions Helm (or a similar chart-based tool), also ask:
     "C. What is the URL of the Helm charts repository?
         (This is the repo that holds the Helm chart definitions — often separate from
          the application source repo)"

   Save all three answers to context:
     infra.deploymentTool    ← answer A
     infra.helmRepository    ← answer C (empty string if Helm not used)
     infra.rollbackMechanism ← answer B

   Embed infra.helmRepository as a real URL in the document wherever IaC repo links appear.
   If not yet available, store "" and render [PLACEHOLDER — add Helm repo URL].

4. Ask the user ONLY for the remaining empty fields that this document needs.
   Do not ask for fields this document does not use.
   Do not re-ask fields that already have a non-empty value in the file.

5. After collecting all answers, merge them into the context object and write
   .iso29110-context.json back to disk (overwrite with merged content).
   This makes the new answers — including any newly collected repo URLs —
   available to the next agent or session.
```

**Example read logic (Node.js, usable inside the generation script or as a pre-step):**

```js
const fs = require('fs');
const path = require('path');

const CTX_PATH = path.join(process.cwd(), '.iso29110-context.json');

function loadContext() {
  if (fs.existsSync(CTX_PATH)) {
    return JSON.parse(fs.readFileSync(CTX_PATH, 'utf8'));
  }
  return {};
}

function saveContext(ctx) {
  fs.writeFileSync(CTX_PATH, JSON.stringify(ctx, null, 2));
}
```

### What stays document-specific (never goes in the shared file)

Some information only makes sense for one document type. Do not write these to the shared context file:

| Document | Document-specific info |
|---|---|
| SD | Use-case list per role; sequence diagram descriptions |
| SC | Derived component table (generated from codebase each time) |
| IE | Specific tool versions (read from lock files each time) |
| MD | Maintenance procedure steps |
| OM | Runbook / operational steps |

### Placeholders for missing values

If the user says "I don't know" or "leave it blank" for a field that goes in the shared file, store an empty string `""` in the JSON (do not omit the key). In the generated document, render the value as `[PLACEHOLDER]` and include it in the post-generation checklist.

---

## Part B — Codebase Cache

### Why this exists

Every ISO 29110 document generation skill includes a "explore the codebase" step. Without a cache, each agent re-reads schemas, package files, module structures, and docker configs from scratch — even when nothing has changed. The codebase cache captures those findings once per git commit and makes them instantly available to any subsequent agent or session.

### The cache file

**Filename:** `.iso29110-codebase-cache.json`  
**Location:** Same directory as `.iso29110-context.json` (the documentation output directory).

#### Schema

```json
{
  "_meta": {
    "generatedAt": "",
    "gitCommit": "",
    "gitBranch": ""
  },
  "structure": {
    "type": "",
    "repoLayout": "",
    "apps": [],
    "libs": []
  },
  "components": {
    "physical": [
      {
        "name": "",
        "interface": "",
        "entryPoint": "",
        "responsibilities": "",
        "interconnections": []
      }
    ],
    "logical": [
      {
        "name": "",
        "interface": "",
        "responsibilities": "",
        "interconnections": [],
        "hostedIn": ""
      }
    ]
  },
  "techStack": {
    "frontend": { "framework": "", "language": "", "runtime": "", "keyLibraries": [] },
    "backend":  { "framework": "", "language": "", "runtime": "", "keyLibraries": [] }
  },
  "database": {
    "engine": "",
    "version": "",
    "collections": [],
    "indexingNotes": "",
    "cacheLayer": ""
  },
  "api": {
    "style": "",
    "versioning": "",
    "documentationTool": "",
    "validationLibrary": ""
  },
  "security": {
    "authMechanism": "",
    "authLibrary": "",
    "passwordHashing": "",
    "cors": false,
    "helmetOrEquivalent": false,
    "secretManagement": ""
  },
  "performance": {
    "caching": "",
    "queuing": "",
    "scalingStrategy": ""
  },
  "errorHandling": {
    "pattern": "",
    "customExceptionClass": "",
    "loggingLibrary": ""
  },
  "dataConventions": {
    "dbFields": "",
    "tsVariables": "",
    "collectionNaming": "",
    "fileNaming": ""
  },
  "keySchemas": {},
  "testing": {
    "unitFramework": "",
    "e2eFramework": "",
    "testFilePattern": "",
    "coverageReportPaths": []
  },
  "deployment": {
    "containerization": "",
    "orchestration": "",
    "cloudProvider": "",
    "configManagement": ""
  }
}
```

#### Which cache fields each document type uses

| Cache field | SD | SC | IE | MD | OM |
|---|:---:|:---:|:---:|:---:|:---:|
| structure | ✓ | ✓ | ✓ | | |
| components | ✓ | ✓ | | | |
| techStack | ✓ | | ✓ | ✓ | ✓ |
| database | ✓ | | ✓ | ✓ | |
| api | ✓ | | ✓ | | |
| security | ✓ | | ✓ | | |
| performance | ✓ | | ✓ | | ✓ |
| errorHandling | ✓ | | | ✓ | |
| dataConventions | ✓ | | | | |
| keySchemas | ✓ | | | | |
| testing | ✓ | ✓ | | ✓ | |
| deployment | ✓ | | ✓ | ✓ | ✓ |

### Agent protocol — use cache always; rebuild only when the human asks

```
1. Check if .iso29110-codebase-cache.json exists.

2a. Cache EXISTS → use it unconditionally. Skip codebase exploration entirely.
    - Optionally: run `git rev-parse HEAD` and compare to cache._meta.gitCommit.
      If they differ, surface a one-line warning to the user:
        "⚠ Cache was built at commit <short-sha>; HEAD is now <short-sha>.
         Run '/refresh-codebase-cache' or ask me to rebuild the cache if the
         codebase has changed significantly."
      Then continue using the existing cache — do NOT rebuild automatically.

2b. Cache DOES NOT EXIST → tell the user:
        "No codebase cache found. I'll read the codebase now to build one.
         This only needs to happen once; future document generations will
         reuse it instantly."
    Then run the full codebase exploration (exploration checklist below),
    populate every field, and write .iso29110-codebase-cache.json to disk.

3. Rebuild trigger — only rebuild the cache when the human explicitly says so, e.g.:
     "refresh the cache", "re-read the codebase", "rebuild the codebase index"
   On rebuild: repeat step 2b and overwrite the file.
```

**Node.js helper to read/write the cache:**

```js
const fs   = require('fs');
const path = require('path');
const { execSync } = require('child_process');

const CACHE_PATH = path.join(process.cwd(), '.iso29110-codebase-cache.json');

function currentCommit() {
  try { return execSync('git rev-parse HEAD').toString().trim(); }
  catch { return null; }
}

function loadCache() {
  if (!fs.existsSync(CACHE_PATH)) return null;
  const cache = JSON.parse(fs.readFileSync(CACHE_PATH, 'utf8'));
  const commit = currentCommit();
  if (commit && cache._meta?.gitCommit !== commit) return null; // stale
  return cache;
}

function saveCache(cache) {
  cache._meta = {
    generatedAt: new Date().toISOString(),
    gitCommit:   currentCommit() ?? 'unknown',
    gitBranch:   (() => { try { return execSync('git rev-parse --abbrev-ref HEAD').toString().trim(); } catch { return 'unknown'; } })(),
  };
  fs.writeFileSync(CACHE_PATH, JSON.stringify(cache, null, 2));
}
```

### Codebase exploration checklist (use when building the cache)

Run these reads/greps in whatever order is most efficient. Each item maps to a cache field.

| Cache field | Where to look |
|---|---|
| `structure.type` | Does one repo contain multiple `apps/` / `packages/`? → monorepo. Multiple separate repos? → multi-repo. |
| `structure.apps` | Subdirs of `apps/`; or separate repo roots; or `docker-compose.yml` service names |
| `structure.libs` | Subdirs of `libs/` or `packages/` |
| `components.physical` | Dockerfile per service, `docker-compose.yml`, K8s Deployment manifests, package.json `start` scripts |
| `components.logical` | Module dirs inside each app; shared lib dirs; frontend `components/`, `hooks/`, `contexts/`, `api/` |
| `techStack` | `package.json` (dependencies), `tsconfig.json`, `.nvmrc` / `.node-version` |
| `database.engine` | ORM/driver package (mongoose → MongoDB, prisma → check schema, typeorm → check config) |
| `database.collections` | Schema/model files (`*.schema.ts`, `*.model.ts`, `prisma/schema.prisma`) |
| `database.cacheLayer` | Redis client packages (`ioredis`, `redis`), cache module configs |
| `api.style` | Controller decorators (`@Get`, `@Post`) → REST; `@Query`/`@Mutation` → GraphQL |
| `api.versioning` | Route prefixes in controllers or main bootstrap |
| `api.documentationTool` | `@nestjs/swagger`, `swagger-jsdoc`, etc. |
| `security.authMechanism` | Auth guard files, JWT strategy files, `passport-*` packages |
| `security.passwordHashing` | `argon2`, `bcrypt`, `bcryptjs` in dependencies |
| `security.cors` / `helmetOrEquivalent` | `main.ts` or app bootstrap; `helmet`, `@nestjs/helmet`, `cors` calls |
| `performance.queuing` | `bull`, `bullmq`, `amqplib`, `kafkajs` in dependencies |
| `performance.scalingStrategy` | K8s HPA manifests, docker-compose `replicas`, cloud provider config |
| `errorHandling.pattern` | Files named `*exception.filter.ts`, `*error.ts`, custom error classes |
| `dataConventions` | Spot-check 3–5 schema files for field naming, 3–5 DTO files for TS naming |
| `keySchemas` | Copy 1–2 representative schema objects (truncated to key fields) |
| `testing.unitFramework` | `jest`, `vitest` in devDependencies; jest config file |
| `testing.e2eFramework` | `playwright`, `cypress`, `supertest` in devDependencies |
| `deployment.containerization` | Presence of `Dockerfile` |
| `deployment.orchestration` | Presence of `k8s/`, `helm/`, `docker-compose.yml` |
| `deployment.cloudProvider` | GCP, AWS, Azure clues in CI files, config comments, or SDK packages |

### Notes

- The cache stores **derived facts**, not raw source. Summarise — do not paste entire files.
- If a field genuinely cannot be determined (e.g., no Dockerfile exists), store `""` or `[]`. Do not leave the key absent.
- The cache is a developer tool, not a deliverable. It may be `.gitignore`d. Remind the user to add `.iso29110-codebase-cache.json` to `.gitignore` if they don't want it committed.

---

## Part C — Rendering Rules

Apply every rule below when writing the `docx`-js generation script.

### Font

**All text uses Calibri.** Set `font: 'Calibri'` on every `TextRun` in the document — headings, body, table cells, cover page, document history. The only exception is code/JSON display blocks, which use `'Courier New'` (see Rule 3).

Calibri is chosen because it is natively supported by both Microsoft Word and Google Docs, ensuring the document renders consistently when team members open it in either tool.

```js
// Every non-code TextRun must carry font: 'Calibri'
new TextRun({ text: 'Some text', font: 'Calibri', size: 22 })
```

---

## 1. Table column widths — always set `columnWidths` on the `Table`, not just on cells

`docx`-js accepts a `width` property on `TableCell`, but Word silently ignores it unless the `Table` itself also declares `columnWidths` as an array of twip values.

**Always do both:**

```js
new Table({
  width: { size: 9072, type: WidthType.DXA },
  columnWidths: [2200, 6872],   // required — cell-level width alone is ignored
  rows: [...]
})
```

- A4 usable body width = **9072 twips** (6.3 in × 1440 twips/in, assuming 1-in left/right margins).
- Column widths must sum to ≤ 9072.
- **Never use `WidthType.PERCENTAGE`** — mishandled by both Word and LibreOffice when the table has an absolute `size`.

---

## 2. Embedded images — read actual pixel dimensions; scale proportionally

`ImageRun` `transformation: { width, height }` values are in **points**, not pixels. Guessing dimensions causes stretching or squashing.

```js
// Requires: npm install image-size  (install only if not already present)
const sizeOf = require('image-size');
const dim = sizeOf(imgPath);
const imgWidth = 630; // points — fills A4 usable width
const imgHeight = Math.round(imgWidth * (dim.height / dim.width));

new ImageRun({
  data: fs.readFileSync(imgPath),
  transformation: { width: imgWidth, height: imgHeight },
  type: 'png',
})
```

---

## 3. Multi-line code / JSON blocks — one `Paragraph` per line

`\n` inside a `TextRun` is not a paragraph break in OOXML and renders as nothing in Word. Map each line to its own `Paragraph`:

```js
codeLines.map(line =>
  new Paragraph({
    children: [new TextRun({ text: line, font: 'Courier New', size: 18 })],  // Courier New: only exception to Calibri rule
    spacing: { before: 0, after: 0 },
    indent: { left: 360 },
  })
)
```

---

## 4. Cover page title — render once unless the template explicitly requires duplication

Some legacy ISO templates show the title block twice. Unless the client's source document confirms this pattern, render the title **once**. A duplicated block reads as a formatting error.

---

## 5. Table cells with lists — use dash-prefixed `Paragraph` children, never comma strings

When a cell value is a list of items, render each item as its own `Paragraph` with a `"- "` prefix prepended to the text. A `TableCell` accepts an array of `Paragraph` children.

**Never join list items into a comma-separated string** — it makes long lists unreadable.  
**Do not use `bullet: { level: 0 }`** — use a plain dash prefix instead for consistent, portable rendering.

Build a `makeCell` helper that auto-detects arrays:

```js
function makeCell(value, widthTwips, options = {}) {
  const children = Array.isArray(value)
    ? value.map(
        item =>
          new Paragraph({
            children: [new TextRun({ text: `- ${item}`, ...options })],
            spacing: { before: 0, after: 0 },
          })
      )
    : [new Paragraph({ children: [new TextRun({ text: value, ...options })] })];

  return new TableCell({
    width: { size: widthTwips, type: WidthType.DXA },
    children,
  });
}
```

Callers then pass either a string or an array:
```js
makeCell('REST API', 1800)
makeCell(['Uses libs', 'Exposes API', 'Uses DB'], 2400)
```

---

## 6. Headings — always use `HeadingLevel.*` with explicit size overrides; never use a bold `TextRun` alone

A plain `Paragraph` with a bold `TextRun` is not a heading. Word and LibreOffice assign heading styles based on the `heading:` property on `Paragraph`, which is what makes them appear in the document outline, table of contents, and navigation pane. If `heading:` is omitted, the line looks bold on screen but is structurally body text — and agents generating subsequent documents cannot detect it as a heading.

**Always pair `heading: HeadingLevel.*` with explicit `size` in the `TextRun`** so the rendered size is deterministic regardless of the consumer's Word theme.

```js
const { HeadingLevel, Paragraph, TextRun } = require('docx');

// ── Level 0: Document section title ─────────────────────────────────────
// Used for the main titled section, e.g. "{ProjectName} Software Components"
// Appears once per document, right after the Document History page break.
new Paragraph({
  heading: HeadingLevel.HEADING_1,
  spacing: { before: 240, after: 160 },
  children: [new TextRun({ text: `${projectName} Software Components`, font: 'Calibri', bold: true, size: 28 })],
  // size 28 = 14 pt (docx size unit is half-points)
})

// ── Level 1: Top-level section ───────────────────────────────────────────
// e.g. "Introduction", "Components", "Technical Design", "Unit Testing"
// Also used for numbered top-level sections: "1. Architectural Design"
new Paragraph({
  heading: HeadingLevel.HEADING_2,
  spacing: { before: 200, after: 120 },
  children: [new TextRun({ text: '1. Architectural or High-Level Software Design', font: 'Calibri', bold: true, size: 26 })],
  // size 26 = 13 pt
})

// ── Level 2: Sub-section ─────────────────────────────────────────────────
// e.g. "1.1 Overall Software Structure", "Frontend Component"
new Paragraph({
  heading: HeadingLevel.HEADING_3,
  spacing: { before: 160, after: 80 },
  children: [new TextRun({ text: '1.1 Overall Software Structure', font: 'Calibri', bold: true, size: 24 })],
  // size 24 = 12 pt
})

// ── Level 3: Sub-sub-section ─────────────────────────────────────────────
// e.g. "1.4 Design Considerations", "2.1.1 SD-M1 – Authentication"
new Paragraph({
  heading: HeadingLevel.HEADING_4,
  spacing: { before: 120, after: 60 },
  children: [new TextRun({ text: '1.4.1 Performance', font: 'Calibri', bold: true, size: 22 })],
  // size 22 = 11 pt
})
```

**Summary table — heading levels for ISO 29110 documents**

| Content | `HeadingLevel` | `size` (half-pt) | Rendered pt |
|---|---|:---:|:---:|
| Document section title (`{Project} Software Design`) | `HEADING_1` | 28 | 14 pt |
| Top-level section / numbered section (`1. …`) | `HEADING_2` | 26 | 13 pt |
| Sub-section (`1.1 …`, named group headings) | `HEADING_3` | 24 | 12 pt |
| Sub-sub-section (`1.1.1 …`, design consideration labels) | `HEADING_4` | 22 | 11 pt |

**What not to do:**
```js
// ✗ Wrong — looks bold, but is body text; not a real heading; also missing font
new Paragraph({ children: [new TextRun({ text: 'Introduction', bold: true, size: 26 })] })

// ✓ Correct
new Paragraph({ heading: HeadingLevel.HEADING_2, children: [new TextRun({ text: 'Introduction', font: 'Calibri', bold: true, size: 26 })] })
```

---

## Quick reference — A4 twip constants

| Measurement | Twips |
|---|---|
| Page width | 11906 |
| Page height | 16838 |
| Left/right margin (1 in each) | 1440 |
| **Usable body width** | **9072** |
| Top/bottom margin (1 in each) | 1440 |
| 1 inch | 1440 |
| 1 cm | ~567 |
