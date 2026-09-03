# Skill: Generate ISO 29110 Operation Guidelines Document

> **Prerequisite:** Before writing any `.docx` generation script, read and apply all rules in `iso29110-docx-rules.md`. Those rules correct confirmed rendering bugs in Word and LibreOffice and are not repeated here.

Generate a **Product Operation Guideline (OM)** work product conforming to ISO/IEC 29110-5-1-2:2025 (VSE Generic Basic Profile) as a `.docx` file. The Application Overview section comes from the codebase cache. Everything from Section 3 onward is human-specified operational knowledge that cannot be derived from code.

---

## What this document is

The Operation Guidelines document is a **runbook for operations and maintenance personnel**. It covers how to deploy, scale, roll back, back up, monitor, secure, and troubleshoot the running system. It is not a developer guide — it assumes the software is already built and focuses on keeping it running correctly in production.

---

## Document dependencies (links this document contains)

| Placeholder location | What it links to | Shared context field |
|---|---|---|
| Section 2 — repos | Source repository URLs | `vcs.repositories` |
| Section 4 — monitoring | Monitoring dashboard URL (if any) | none — ask user |

**Remind the user at the end of generation:**
> "Fill in after files are set up:
> 1. **Repository URLs** (Section 2) — from `vcs.repositories` in context file.
> 2. **Monitoring dashboard URL(s)** (Section 4) — if you have hosted dashboards.
> 3. Reviewer / approver names in Document History."

---

## Step 1 — Gather context from the shared file and the user

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part A.

1. Check for `.iso29110-context.json`. Read it if present.
2. Relevant shared fields: `projectName`, `companyName`, `systemDescription`, `techStack`, `vcs.repositories`.
3. Ask only for empty shared fields, plus these OM-specific items (ask every time):

```
A. What tool is used to manage infrastructure? (e.g. Helm, Terraform, kubectl directly)
   And what is the IaC repository name?
B. How are application services started/stopped?
   (e.g. Kubernetes manages containers automatically; manual kubectl restart only for troubleshooting)
C. What is the backup strategy for the database?
   (e.g. automated nightly MongoDB Atlas backup; manual export on each release)
D. What monitoring tool(s) are used?
   (e.g. Kuma for liveness, k8s Lens for in-depth, GCP Monitoring for metrics)
E. Where are application logs stored / accessed?
   (e.g. GCP Logging, ELK, Datadog)
F. What are the user roles in the system and what can each role do?
   (e.g. Admin: manage system; User: use core features)
G. How are secrets / credentials managed?
   (e.g. Kubernetes Secrets + GCP Secret Manager; only accessible to the infra lead)
H. List 3–5 common operational issues and their resolution steps.
   (These become the Troubleshooting Guide in Section 6. If none provided,
    the agent will generate generic ones based on the tech stack from cache.)
```

4. Merge shared answers into `.iso29110-context.json` and save.

---

## Step 2 — Load codebase facts (cache first; explore only if cache is absent)

Follow the **Agent protocol** in `iso29110-docx-rules.md` Part B.

- **Cache exists** → read `structure`, `techStack`, `database`, `deployment`, `security`. Skip exploration.
- **Cache absent** → run the exploration checklist (same as IE skill Step 2). Build the cache first.
- **Never re-read** unless explicitly asked.

The Application Overview (Section 2) is generated entirely from cache fields. Sections 3–6 are built from the user's answers in Step 1.

---

## Step 3 — Identify gaps

If troubleshooting scenarios (Step 1 H) were not provided, generate plausible generic ones based on the tech stack from cache (e.g. service unresponsive, slow performance, async job failures, event tracking not recording). Note these as examples the user should verify.

Remind the user:
> "Placeholders to fill in:
> - Repository URLs (Section 2)
> - Backend service endpoint URLs (Section 2)
> - Monitoring dashboard links (Section 4, if applicable)
> - Verify troubleshooting scenarios in Section 6 are accurate for your system"

---

## Step 4 — Generate the document

### Document structure

```
Cover page
  Title: "Product Operation Guideline"
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

Section heading: "{ProjectName} Product Operation Guidelines"  [HEADING_1, 14pt]

1. Introduction                                              [HEADING_2, 13pt]
   [One paragraph: this document provides operational procedures for {projectName}.
    It is intended for maintenance and operations personnel to ensure continuous
    availability and optimal performance of the system.]

2. Application Overview                                      [HEADING_2]
   [Intro sentence: "For application specifications, frameworks, and technical detail:"]
   [Same structure as Section 2 in the Maintenance Document — generated from cache:
    Frontend block, Backend block with per-service detail, IaC block.
    Include repository placeholders for Main/Backup repos.]

3. Operational Procedures                                    [HEADING_2]
   [Intro: "The following procedures are essential for managing {projectName} in production."]
   
   3.1. {IaC Tool} for Infrastructure Management            [HEADING_3]
     [Describe: how to deploy a new version (e.g. helm upgrade), how to scale
      (adjust replica count + redeploy), how to roll back (e.g. helm rollback).]
     — populated from answer A
   
   3.2. Startup and Shutdown Procedures                      [HEADING_3]
     — populated from answer B
     [If Kubernetes-managed: note that K8s handles startup/shutdown automatically;
      manual intervention (kubectl restart) only for troubleshooting specific pods.]
   
   3.3. Backup and Recovery                                  [HEADING_3]
     Backup Strategy: {answer C}
     Recovery Procedure:
       [Step 1: Stop affected services to prevent further data corruption.]
       [Step 2: Restore latest valid backup to a recovery environment.]
       [Step 3: Verify data integrity and functionality.]
       [Step 4: Restart services with restored database.]

4. Monitoring and Logging                                    [HEADING_2]
   Monitoring: {answer D — tool names and what they monitor}
   Logging: {answer E — where logs are stored and how to access them}

5. Security Guidelines                                       [HEADING_2]
   User Roles and Access Control: {answer F}
   Authentication: [Describe auth mechanism from cache — JWT, OAuth, etc.]
   Handling Secrets: {answer G}
   Code Reviews: [Standard note: regular peer reviews; OWASP awareness.]

6. Troubleshooting Guide                                     [HEADING_2]
   [Intro: "Common issues and general resolution approach."]
   
   [For each issue from answer H (or generated from cache if not provided):]
   Issue: {issue description}
   Action:
     {step 1}
     {step 2}
     {step 3 — escalation if unresolved}
   
   [Always include a generic "escalation" step at the end of each issue block:
    "If the issue persists, escalate to the development team for in-depth diagnosis."]
```

### Styling rules
- **Font**: Calibri throughout.
- **Page size**: A4.
- **Heading levels**: follow `iso29110-docx-rules.md` Rule 6.
- **Troubleshooting action steps**: render as `{step text}` body paragraphs with left indent — not numbered lists (matches original style where steps flow as paragraphs).
- **Issue / Action labels** ("Issue:", "Action:"): bold the label, normal weight the value.
- **Cover page / history**: Calibri, standard.
- **No logo** — omit any image placeholder.
- **Author/reviewer fields**: blank.

### Running the script
```bash
node generate_om.js
```
Output: `{ProjectName}_Operation_Guidelines_V1.0.docx`

---

## Step 5 — Post-generation checklist

Remind the user to:
1. Fill in **author name** on cover page and Document History row 1.
2. Add **reviewer rows** when reviewed.
3. Replace **repository URLs** and **service endpoint URLs** in Section 2.
4. Add **monitoring dashboard links** in Section 4 if hosted.
5. **Verify troubleshooting scenarios** in Section 6 — generated ones are plausible but need team review.
6. Update Section 2 **dependency lists** on each major release.
