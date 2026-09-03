# Skill: Generate ISO 29110 Maintenance Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate a **Maintenance Document (MD)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file. The Application Overview section is fully codebase-derivable from the cache; the Maintenance Activities section requires human input for tool names and team-specific procedures.

---

## What this document is

The Maintenance Document describes the software product and its environment in terms that **maintenance personnel** can understand. It covers the system's architecture and tech stack, defines maintenance objectives, and details four categories of maintenance activity (corrective, adaptive, perfective, preventive) with their processes and tooling.

---

## Document dependencies (links this document contains)

This document contains no outbound document links. All external references (monitoring dashboards, CI tools) should be named in prose but URLs are optional — remind the user to add them if they exist.

**Remind the user at the end of generation:**
> "No document links required. Optionally add URLs to:
> - Monitoring dashboards (Section 4)
> - Static analysis / dependency tools (Section 4.4)
> - Reviewer / approver names in Document History"

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json`. Read it if present.
2. Relevant shared fields: `projectName`, `companyName`, `systemDescription`, `techStack`, `testCoverageUrls`.
3. Ask only for empty shared fields, plus these MD-specific items (ask every time):

```
A. What communication/ticketing tool does the team use for issue reporting?
   (e.g. Slack, Jira, Linear, GitHub Issues)
B. What monitoring tools are used in production?
   (e.g. Prometheus, Grafana, Datadog, k8s Lens, GCP Monitoring)
C. What logging platform stores production logs?
   (e.g. GCP Logging, ELK Stack, Datadog Logs)
D. What static analysis / code quality tools are used?
   (e.g. SonarQube, ESLint, Dependabot — leave blank if none)
E. Are there any automated dependency update tools?
   (e.g. Dependabot, Renovate — leave blank if none)
F. Brief description of the release / deployment cycle:
   (e.g. "hotfix PRs go straight to prod; regular features go through dev → UAT → prod")
```

4. Merge shared answers into `.iso29110-context.json` and save.

---

## Step 2 — Load codebase facts (cache first; explore only if cache is absent)

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part B.

- **Cache exists** → read `structure`, `techStack`, `database`, `deployment`, `testing`. Skip exploration.
- **Cache absent** → run the IE exploration checklist (same checklist as in `iso29110-implementation-environment.md` Step 2) to build it, then continue.
- **Never re-read** unless explicitly asked.

The Application Overview section (Section 2) is generated entirely from cache fields: `techStack`, `database`, `deployment`, and `structure`.

---

## Step 3 — Identify gaps

If any tech stack detail is missing from the cache, note `[Verify: check package.json]` inline rather than asking the user.

Remind the user:
> "No external links in this document. Fill in:
> - Author name and reviewer names in Document History
> - Any tool names left as `[PLACEHOLDER]` in Maintenance Activities
> - Monitoring / logging tool links if you want them hyperlinked"

---

## Step 4 — Generate the document

### Document structure

```
Cover page
  Title: "Maintenance Document"
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

Section heading: "{ProjectName} Maintenance Document"     [HEADING_1, 14pt bold]

1. Introduction                                           [HEADING_2, 13pt bold]
   [Two paragraphs:]
   Para 1: This document outlines the maintenance procedures for {projectName},
   a {systemDescription}. It adheres to the guidelines of ISO/IEC 29110-5-1-2:2025
   (VSE Generic Basic Profile).
   Para 2: Per ISO 29110, Maintenance Documentation describes the software product
   and the environment used for development and testing. It is written for
   maintenance personnel to understand.

2. Application Overview                                   [HEADING_2]
   [Intro sentence: "The application is structured as follows:"]
   
   Frontend ({frontend.framework}):                       [HEADING_3]
     Repository: Main, Backup
     Framework: {frontend.framework} with {frontend.language}
     Deployment: {docker image} managed by {orchestration}, {cloud}
     Key Libraries/Technologies:
       {runtime}: {version range}
       {package manager}: {version range}
       NPM Dependencies
       [Key prod dependencies — name: version, one per line]
   
   Backend ({backend.framework}):                         [HEADING_3]
     Repository: Main, Backup
     Framework: {backend.framework}
     Programming Language: {backend.language}
     Database: {database.engine}, {database.cacheLayer}
     Cache: {database.cacheLayer}
     
     [For each backend service from cache.structure.apps:]
     {Service Name}:
       Purpose: {responsibility}
       Deployment: {runtime image} managed by {orchestration}, {cloud}
       Endpoints: [PLACEHOLDER — add URL]
     
     Key Libraries/Dependencies:
       {runtime}: {version range}
       [Key backend prod dependencies — name: version]
   
   [If IaC exists:]
   Infrastructure as Code:                                [HEADING_3]
     Repository: Main, Backup
     Framework: {IaC tool}
     Deployment platform: {cloud} using {orchestration}

3. Maintenance Objectives                                  [HEADING_2]
   The primary objectives of this maintenance plan are to:
   - Ensure continuous availability and optimal performance.
   - Address and resolve defects, bugs, and errors promptly.
   - Implement enhancements, new features, and improvements.
   - Adapt the application to changes in the operating environment or business requirements.
   - Maintain the security and integrity of the application and its data.
   - Minimise downtime and service disruptions.

4. Maintenance Activities                                  [HEADING_2]
   [Intro: "Maintenance activities are categorised into four types:"]

   4.1 Corrective Maintenance                             [HEADING_3]
     Purpose: To identify and fix errors, bugs, or defects.
     Process:
       Issue Reporting: {answer A — tool name}
       Triage & Prioritisation: Reported issues evaluated for severity and impact.
       Diagnosis: Developers investigate via logs, code, and system metrics.
       Fix Implementation: Code changes to resolve the bug.
       Testing: Unit, integration, and regression testing on the fix.
       Deployment: Via hotfix or regular release cycle.
     Tools: {answer A}

   4.2 Adaptive Maintenance                               [HEADING_3]
     Purpose: To modify the application for environmental changes
     (OS updates, DB version changes, third-party API changes, regulatory requirements).
     Process:
       Environmental Monitoring: Monitor for updates or deprecations in technologies used.
       Impact Analysis: Assess impact on the application.
       Modification: Adapt code or configuration.
       Testing: Comprehensive compatibility and functionality testing.
       Deployment: Deploy updated application.
     Examples: Upgrading framework major versions, updating NPM dependencies.

   4.3 Perfective Maintenance                             [HEADING_3]
     Purpose: To improve performance, maintainability, reliability, or usability.
     Process:
       Analysis: Identify areas for improvement (slow queries, complex code, poor UX).
       Design & Development: Optimise code, refactor modules, improve algorithms.
       Testing: Verify improvements; ensure no regressions.
       Deployment: Release optimised version.
     Examples: Code refactoring, performance optimisations (caching, DB indexing),
     UI enhancements, logging improvements.

   4.4 Preventive Maintenance                             [HEADING_3]
     Purpose: To prevent future problems proactively.
     Process:
       Code Reviews: Regular peer reviews for potential bugs, security issues, anti-patterns.
       Automated Testing: Maintain and expand unit, integration, and e2e test suites.
       Security Audits: Regular security scans and penetration testing.
       Infrastructure Monitoring: {answer B} for server resources, network, and app metrics.
       Documentation Updates: Keep technical docs, architecture diagrams, and deployment
       guides up-to-date.
       Dependency Updates: Regularly update third-party packages.
     Tools: {answer D if set, else omit}, {answer E if set, else omit},
            {answer B} for monitoring, {answer C} for logging.
```

### Styling rules
- **Font**: Calibri throughout; Courier New for inline package/version references if rendered as code.
- **Page size**: A4.
- **Heading levels**: follow `iso29110-docx-rules.md` Rule 6.
- **Maintenance process steps** (Issue Reporting, Triage, etc.): render as `{Label}: {description}` body paragraphs with a small left indent — not as bullet lists.
- **Maintenance Objectives**: render as dash-prefixed items (see Rule 5 in shared rules).
- **Cover page / history**: Calibri, same as all other docs.
- **No logo**.
- **Author/reviewer fields**: blank.

### Running the script
```bash
node generate_md.js
```
Output: `{ProjectName}_Maintenance_Document_V1.0.docx`

---

## Step 5 — Post-generation checklist

Remind the user to:
1. Fill in **author name** on cover page and Document History row 1.
2. Add **reviewer rows** to Document History when reviewed.
3. Replace **backend service endpoint URLs** with production URLs.
4. Verify **tool names** in Section 4 (corrective, monitoring, logging tools).
5. Update **dependency lists** in Section 2 on each major release.
