---
name: research-report
description: Unified research-to-report pipeline. Spawns parallel research agents, cross-references findings, runs fact-checker, generates publication-ready .docx, optionally creates slides/infographic, and logs to Obsidian.
argument-hint: "<topic> [--slides] [--infographic] [--education] [--light]"
---

# Research Report: $ARGUMENTS

Unified pipeline that chains: parallel research agents, cross-referencing, fact-checking, .docx generation, optional slides/infographic, and Obsidian logging.

## Parse Arguments

Extract from `$ARGUMENTS`:
- `topic`: the research topic (required)
- `--slides`: also generate a PowerPoint presentation
- `--infographic`: also generate an HTML infographic
- `--education`: add TN/education-specific sections and redirect agents to education sources
- `--light`: use 3 agents instead of 5 (faster, fewer sources)

## Pipeline Overview

```
Phase 1: Setup & Planning
Phase 2: Parallel Research Agents (3-5)
Phase 3: Cross-Reference & Synthesis
Phase 4: Fact-Check Pass
Phase 5: Generate Final .docx
Phase 6: Optional Deliverables (slides, infographic)
Phase 7: Obsidian Logging & Cleanup
```

## Phase 1: Setup & Planning

1. Generate a short, descriptive folder name (lowercase, hyphens).
2. Create working directories:
   ```
   mkdir -p "~/Documents/Research/<topic-folder>/drafts"
   ```
3. Initialize run log at `~/Documents/Research/<topic-folder>/research_report_run_log.md`
4. Briefly state the research plan to the user in 2-3 bullets and wait for confirmation before launching agents.

## Phase 2: Parallel Research Agents

### Team Setup

1. `TeamCreate` with `team_name: "research-<topic-slug>"`.
2. Create one `TaskCreate` per agent.
3. Spawn ALL agents in a SINGLE message for parallel execution using `Agent` tool with `subagent_type: "general-purpose"`.

### Agent Assignments

**Default (5 agents):**

| Agent | Name | Focus |
|-------|------|-------|
| 1 | `gov-sources` | Government publications, .gov domains, regulatory docs, official statistics |
| 2 | `news-sources` | Major news outlets, investigative journalism, trade press |
| 3 | `academic-sources` | Peer-reviewed journals, meta-analyses, university research centers |
| 4 | `industry-sources` | Industry reports, white papers, practitioner guides, nonprofit/NGO publications |
| 5 | `case-studies` | Real-world implementations, district/org case studies, practitioner interviews |

**Light mode (3 agents):** Use agents 1, 3, and 4 only.

**Education mode additions:**
- Redirect `gov-sources` to prioritize TDOE, USED, NCES, TN legislature
- Redirect `industry-sources` to include ASCD, Learning Forward, NASSP, TASBO, ASBO
- Add `case-studies` focus on TN districts and comparable small districts

### Structured Output Format

Every agent MUST return findings in this format:

```markdown
## Findings

### Finding 1: <title>
- **Claim**: <specific factual claim>
- **Evidence**: <supporting detail, data, quotes>
- **Confidence**: <high|medium|low> - <reason>
- **Source**: <Author/Org (Year). Title. URL>

## Source List
1. <Author/Org (Year). Title. URL> - Confidence: <high|medium|low>

## Gaps & Limitations
- <areas not adequately covered and why>
```

**Confidence scoring:**
- High: Primary source, peer-reviewed, official government data, multiple corroborating sources
- Medium: Reputable secondary source, single expert opinion, plausible but not independently verified
- Low: Blog post, opinion piece, outdated data (>5 years for fast-moving fields), potential bias

### Validation & Retry

An agent passes if: 5+ distinct findings with confidence scores and 5+ unique sources.

On failure: retry up to 3 times with exponential backoff (30s, 60s, 120s). After 3 failures, proceed without that agent. Minimum 3 successful agents to continue.

## Phase 3: Cross-Reference & Synthesis

### 3a. Claim Matrix

For each major claim across all agents:

| Claim | Gov | News | Academic | Industry | Cases | Agreement |
|-------|-----|------|----------|----------|-------|-----------|
| <claim> | check/x/-- | ... | ... | ... | ... | full/partial/single/conflicting |

Flag: contradictions, single-source claims, universal agreement.

### 3b. URL Verification

Verify up to 30 URLs using `WebFetch`. Track status (Live/Dead/Redirect/Paywalled). Replace dead URLs where possible. Skip .gov 403s as likely valid.

### 3c. Confidence Aggregation

Score = (Source type weight x Agent confidence x URL status) + Corroboration bonus (cap 1.0)

- Source type: Gov/Academic = 1.0, News = 0.8, Industry/Cases = 0.7
- Agent confidence: High = 1.0, Medium = 0.7, Low = 0.4
- URL status: Live = 1.0, Redirect/Paywalled = 0.8, Dead = 0.3
- Corroboration: 2+ agents = +0.1, 3+ agents = +0.2

Classify: >=0.8 High, 0.5-0.79 Medium, <0.5 Low reliability.

## Phase 4: Fact-Check Pass

1. Generate preliminary .docx in drafts folder.
2. Run the fact-check skill logic (Tier 1 checks) on the preliminary report:
   - Internal consistency
   - Unsourced claims
   - Citation integrity
   - Logical coherence
   - Plausibility flags
   - Stale references
3. Fix all ERROR-level issues. Address straightforward WARNING-level issues.
4. Log fact-check results to run log.
5. Delete preliminary draft after fixes applied.

## Phase 5: Generate Final .docx

### Report Structure

**Title Page:**
- Title: `Research Report: <Topic>`
- Date, source categories covered, total sources, claims analyzed

**Executive Summary (1-2 pages):**
- Key findings, bottom-line assessment, confidence level
- Source category coverage notes

**Sections:**
1. Background & Context
2. Key Findings (organized by theme, NOT by source category)
   - Inline confidence indicators: High/Medium/Low
   - Weave all source perspectives together
3. Points of Contradiction (where sources disagree, both sides presented)
4. Single-Source Claims (claims needing additional verification)
5. Practical Implications (decision-maker takeaways, recommended actions)
6. Source Reliability Matrix (all sources with aggregate scores)
7. References (deduplicated, alphabetical, with reliability ratings)

**Education mode additional sections:**
- Federal & State Policy Context (ESSA, IDEA, TDOE, TN legislation)
- Tennessee Data Appendix (TN-specific stats, district comparisons)

### Formatting

Use `python-docx`:
- Calibri 11pt default
- Heading 1/2/3 styles
- Source Reliability Matrix as formatted Word table with header shading
- Inline citations formatted consistently

### Run Document Validation

After generating the .docx, validate:
- Content_Types.xml includes all required entries
- styles.xml is well-formed
- No comment elements render as hashtags
- All tables render properly
If any check fails, auto-fix and regenerate.

### Save Locations

- Final report: `~/Documents/Research/<topic-folder>/YYYY-MM-DD <Topic> Research Report.docx`
- Generation script: `~/Documents/Research/scripts/`
- Run log: kept in topic folder

## Phase 6: Optional Deliverables

### If `--slides`:
Run `/pptx` skill on the final .docx to generate a presentation.
Save to: `~/Documents/Research/<topic-folder>/YYYY-MM-DD <Topic> Slides.pptx`

### If `--infographic`:
Generate a single-page HTML infographic with key stats, findings, and visual hierarchy.
Save to: `~/Documents/Research/<topic-folder>/YYYY-MM-DD <Topic> Infographic.html`

## Phase 7: Obsidian Logging & Cleanup

### Session Log

Create via `mcp__obsidian__write_note`:

Path: `Sessions/YYYY-MM-DD - Research Report: <Topic>.md`

```markdown
---
type: session
date: YYYY-MM-DD
summary: Research report on <topic>
tags: [session, research-report, <topic-tag>]
---

# YYYY-MM-DD - Research Report: <Topic>

## Task
Generated comprehensive research report on <topic>.

## Pipeline Results
- Agents spawned: X (Y successful)
- Total sources: X (X high, X medium, X low reliability)
- URLs verified: X/Y (X live, X dead replaced)
- Contradictions flagged: X
- Single-source claims: X
- Fact-check: X errors fixed, X warnings addressed

## Deliverables
- Report: <path>
- Slides: <path or N/A>
- Infographic: <path or N/A>
- Run log: <path>

## Follow-Up
- <any gaps or areas needing deeper research>
```

Frontmatter: `type: session`, `date:`, `summary:`, `tags: [session, research-report, <topic-tag>]`

### Team Shutdown

1. Send shutdown to all teammates via `SendMessage`.
2. `TeamDelete` to clean up.
3. Delete drafts directory and temp files.

### Open the Report

```bash
open "<final report path>"
```

## Final Summary to User

Report:
- File path
- Sources found (reliability breakdown)
- Contradictions flagged
- Single-source claims noted
- Any source categories with issues
- Fact-check results
- Additional deliverables generated
- Run log location

## Rules

- Always confirm the research plan before launching agents (2-3 bullets).
- Use privacy-safe Obsidian logging (no sensitive details).
- For education topics, always include TN-specific context.
- Never overwrite existing reports without backup.
- If fewer than 3 agents succeed, warn the user about limited coverage before generating the report.
- Triple-check all statistics and financial figures in the final report.
