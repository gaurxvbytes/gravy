# Gravton Insights Engine — Handoff Document
**Date:** May 2026  
**Purpose:** Full context of all decisions, clarifications, source attributions, and open questions from this working session.

---

## 1. Source Documents Uploaded

| # | File Name | What it covers |
|---|-----------|----------------|
| 1 | `Gravton_MVP_for_Customer_Demos_2_0.pdf` | Full product PRD. Covers all 18 agents: Product & Vertical Mapper, Competitor Mapper, Brand Context Crawler, Prompt Enrichment, Search Data Ingestion, Competitor Intelligence, Citation Monitor, Emerging Topics, Earned Media, Presence Mapper, Persona Mapper, Verticalized Source Configuration, Regional Controls, Snapshot/Demand Map, Platform Controls, Backlink Opportunity, Adoption Health Prompt Discovery, Adoption Health Monitor. |
| 2 | `GSC___GA4_Integrations.pdf` | GSC integration detail. Keyword-level and page-level data, OAuth flow, keyword classification, prompt transformation patterns, owned citation performance, opportunity through owned citations. |
| 3 | `Reddit__Wiki__Linkedin___YouTube_-_Integrations.pdf` | First version of integrations doc. Reddit agent overview. |
| 4 | `Reddit__Wiki__Linkedin___YouTube_-_Integrations__1_.pdf` | Full integrations document. Covers Reddit (full), Quora (full), YouTube (full), LinkedIn (empty), Wikipedia (empty). All cadences, identification layers, insights, opportunities, and technical implementation. |
| 5 | `Prompt_Synthesis_Approach_-_Release_V1.pdf` | Prompt synthesis approach. Stage 0 (brand ingestion), Stage 2 (prompt generation), intent cluster synthesis, breadth/depth prompt types, volume allocation formula (Appendix A), source type taxonomy (8 categories), funnel stage definitions, breadth prompt templates (Appendix B). |
| 6 | `V01_-_Product_HL_System_Design.pdf` | First version of the high-level system design doc being built. |
| 7 | `V01_-_Product_HL_System_Design__1_.pdf` | Updated version of the high-level system design doc with architecture diagram (ArchitectureV01.png). This is the working document being built — NOT a source of truth. |
| 8 | `gravton-expanded-platform-design-doc.md` | Expanded platform design markdown. Earlier working draft. |
| 9 | `workflow.mmd` | Full Mermaid workflow diagram. 13 layers + storage layer. 429 lines. Contains all DAG names and node connections. Used as supplementary reference where PRDs are silent. |

---

## 2. What is Being Built

Two documents are being produced:

1. **Gravton_System_Design.docx** — Technical implementation-heavy design document (low-level). Based on all PRDs + workflow diagram. Covers all 13+ layers with implementation detail, DDL, code patterns, thresholds.

2. **Gravton_Product_Feature_Map.docx** — High-level product feature map. What each module does, what goes in, what comes out. No low-level implementation. Strictly PRD-sourced. This is the primary working document for this session.

3. **gravton_flow.mmd** — Mermaid DAG flow diagram showing all DAGs, parallel vs sequential execution, dependencies.

---

## 3. Confirmed Architecture & DAG Order

```
BRAND FOUNDATION
crawl_dag → brandkit_dag → Snapshot → Client Confirmation

PARALLEL after Client Confirmation:
  gsc_dag (optional) ──────────────────┐
                                        ↓
                             synthetic_prompt_dag + prompt_volume_dag
  adoption_health_prompt_dag (parallel with above)
  reddit_dag  ─┐
  quora_dag   ─┤ (parallel, independent cadence)
  youtube_dag ─┘

PROMPT SYNTHESIS:
  synthetic_prompt_dag + adoption_health_prompt_dag
    ↓ (both feed)
  responses_dag

INTELLIGENCE ENGINE:
  responses_dag
    ├── citation_dag          (acquisition responses)
    ├── sentiment_dag         (acquisition responses)
    ├── query_fanout_dag      (acquisition responses)
    └── adoption_health_monitor_dag  (post-purchase responses)

INSIGHTS + OPPORTUNITY (parallel):
  citation_dag ──────┐
  sentiment_dag ─────┤
  fanout_dag ────────┤──→ insights_dag ──┐
  adoption_monitor ──┘                   ├──→ opportunity_dag
  reddit_dag ────────────────────────────┤
  quora_dag ─────────────────────────────┤
  youtube_dag ───────────────────────────┘
```

**Key sequencing rules:**
- `synthetic_prompt_dag` runs AFTER `gsc_dag` (takes keyword library as input)
- `adoption_health_prompt_dag` runs in PARALLEL with `synthetic_prompt_dag` (same inputs, different output)
- `snapshot_dag` is NOT a separate DAG — Snapshot is an output of `brandkit_dag`
- `insights_dag` and `opportunity_dag` are PARALLEL — neither feeds the other
- `reddit_dag`, `quora_dag`, `youtube_dag` feed `opportunity_dag` DIRECTLY (per PRD)

---

## 4. All DAGs (Final List)

| DAG | Purpose | Depends on |
|-----|---------|------------|
| `crawl_dag` | Discover + extract brand pages | — (runs first) |
| `brandkit_dag` | Brand Kit, Taxonomy, Competitor Map, Persona Map, Regional Controls, Source Config, Snapshot | `crawl_dag` |
| `gsc_dag` | Keyword ingestion, prompt enrichment. Optional. | `brandkit_dag` |
| `synthetic_prompt_dag` | Acquisition prompt library | `brandkit_dag` + `gsc_dag` (optional) |
| `prompt_volume_dag` | Demand volume estimation per cluster | `synthetic_prompt_dag` |
| `adoption_health_prompt_dag` | Post-purchase prompt library | `brandkit_dag` + client inputs (parallel with `synthetic_prompt_dag`) |
| `reddit_dag` | Reddit subreddit + thread ingestion | `brandkit_dag` |
| `quora_dag` | Quora space + question ingestion | `brandkit_dag` |
| `youtube_dag` | YouTube channel + video + transcript ingestion | `brandkit_dag` |
| `responses_dag` | Fire all prompts across all LLMs | Active prompt library |
| `citation_dag` | Brand + competitor SOV, source classification | `responses_dag` |
| `sentiment_dag` | Brand sentiment per response | `responses_dag` |
| `query_fanout_dag` | Extract LLM-generated related questions | `responses_dag` |
| `adoption_health_monitor_dag` | Post-purchase consistency, sentiment, source ownership | `adoption_health_prompt_dag` |
| `insights_dag` | Demand Map, Presence Matrix, Citation Performance, Adoption Health | `citation_dag` + `sentiment_dag` + `adoption_monitor` |
| `opportunity_dag` | Gap scoring, Opportunity Agent, action briefs | `insights_dag` + `reddit_dag` + `quora_dag` + `youtube_dag` |

**Removed DAGs (not in scope):**
- `snapshot_dag` — merged into `brandkit_dag` output
- `competitor_intelligence_dag` — removed
- `earned_media_dag` — removed
- `emerging_topics_dag` — removed
- `backlink_dag` — removed

---

## 5. Source Attribution Rules (Critical)

**Rule: If it's not in a PRD or the workflow diagram, it must be flagged as reasoning.**

### Document-based (safe to include):
- Reddit/Quora/YouTube cadences — from integrations PDF
- GSC keyword + page data — from GSC Integration PDF
- Breadth/depth ratio (25:75) — from Prompt Synthesis V1
- Volume allocation formula (β=0.65, floor=4%, cap=3×) — from Prompt Synthesis V1 Appendix A
- Funnel stages (Upper/Mid/Lower/Post-Purchase) — from Prompt Synthesis V1
- Source type taxonomy (8 categories) — from Prompt Synthesis V1
- Adoption Health 3 dimensions (consistency, sentiment, source ownership) — from MVP PRD
- Adoption Health 24h alerts — from MVP PRD
- Opportunity types (8 types) — from MVP PRD
- Citation feedback loop — from workflow.mmd + integrations PDF
- Snapshot as `brandkit_dag` output — from MVP PRD Section 14
- Reddit/Quora/YouTube → `opportunity_dag` directly — from all three integration PRDs explicitly

### Reasoning / Not in PRD (flag or remove):
- `SEO vs AI Visibility Correlation` as an output view — NOT in any PRD. My reasoning.
- Scoring formula weights (0.4/0.4/0.2 for clicks/impressions/intent) — NOT in PRD. My reasoning.
- "Primary" vs "secondary" for keyword vs page-level GSC data — NOT in PRD. My interpretation.
- Stale prompt detection — NOT in PRD. My reasoning.
- `adoption_health_prompt_dag` being depth-only — NOT in PRD. Removed.
- Specific similarity thresholds (0.70, 0.75, 0.88, 0.90, 0.92 etc.) — from workflow.mmd, not PRD.
- `insights_dag` and `opportunity_dag` as parallel — inferred from PDF architecture diagram + PRD logic.

---

## 6. Key Decisions Made This Session

### Source Config
- Is a predefined list per vertical/region acting as a starting point for the system prompt template
- Community forums (Reddit, Quora, YouTube) are hard-fed across all verticals
- All other source types (media, editorial, academic, regulatory, aggregator) are examples the LLM reasons over
- If brand LOB doesn't match predefined list, LLM identifies right sources
- Source: MVP PRD Section 12 + Prompt Synthesis V1 source type table

### Snapshot
- NOT a separate DAG
- Output of `brandkit_dag`
- Shows: topic clusters per vertical, competitors per vertical, brand tonality, personas
- Client confirms before anything downstream runs
- Source: MVP PRD Section 14

### Demand Map
- Output of `insights_dag` (not brandkit_dag)
- Produced AFTER `responses_dag` runs
- Shows: Winning / Sharing / Losing / Untapped per prompt per platform
- Source: MVP PRD Section 14 + workflow diagram

### Adoption Health
- `adoption_health_prompt_dag` runs in PARALLEL with `synthetic_prompt_dag` (same inputs)
- `adoption_health_monitor_dag` runs AFTER `responses_dag` on post-purchase responses only
- Both are separate DAGs (not merged into responses_dag)
- Primary classifier: first-person ownership language ("my [product]", "our setup", "we're using")
- Source: MVP PRD Sections 17 + 18

### GSC Data
- Two types: keyword-level (queries, clicks, impressions, CTR, position) and page-level (URL, title, clicks, impressions, CTR, position)
- Keyword-level → prompt enrichment
- Page-level → citation performance tracking
- CSV upload prioritised over API for initial release
- Page-level data deprioritised for initial release
- Both exports available as separate CSV downloads from GSC interface
- Source: GSC Integration PDF

### Crawl Limit
- Current limit: 50 pages
- May not be sufficient for large product portfolios (e.g. Apple-scale brands)
- Decision needed: crawl all pages OR use LLM-based approach to prioritise pages
- Source: Session discussion, not in PRD

### Authority Source Layer → Opportunity Engine
- Reddit, Quora, YouTube feed `opportunity_dag` DIRECTLY
- Explicitly stated in all three integration PRDs
- NOT through `insights_dag`

### GA4
- Explicitly scoped out: "will be backend for v1 - part of Authority Pages v2" — stated in Reddit, Quora, and YouTube PRDs
- Not in current scope

### LinkedIn + Wikipedia
- Pages exist in integrations PDF but are EMPTY
- No product content documented yet
- Not in current scope

---

## 7. Open Questions / To Resolve

| # | Question | Status |
|---|----------|--------|
| 1 | Clicks vs impressions weighting in GSC scoring — should they have different weights? | Open. PRD silent. General SEO practice suggests clicks > impressions (intent signal). No formula defined. |
| 2 | Crawl limit: 50 pages enough for large brands? Need LLM-based approach? | Open. Raised in session. No PRD decision. |
| 3 | SEO vs AI Visibility Correlation view — include or remove? | My reasoning only. Not in PRD. Needs product decision. |
| 4 | `insights_dag` outputs — does it include authority source performance (Reddit/Quora/YouTube views)? | Partially open. Authority sources feed `opportunity_dag` directly per PRD. Their presence in `insights_dag` dashboard views needs clarification. |
| 5 | Source Config — how often does the pre-seeded list update? Who maintains it? | MVP PRD says Gravton admins maintain it. Update cadence not defined. |
| 6 | Volume allocation formula for GSC keywords — apply same power-law dampening as prompt synthesis? | Direction agreed but formula not finalised. |
| 7 | LinkedIn integration — when does it come into scope? | No PRD. Empty page in integrations doc. |
| 8 | Wikipedia integration — when does it come into scope? | No PRD. Empty page in integrations doc. |

---

## 8. Documents Produced This Session

| File | Description |
|------|-------------|
| `Gravton_System_Design.docx` | Technical low-level design doc. 18 sections. All DAGs with implementation detail. |
| `Gravton_Product_Feature_Map.docx` | High-level feature map. PRD-sourced. What each module does, inputs, outputs. |
| `gravton_flow.mmd` | Mermaid DAG flow diagram. Paste into mermaid.live to render. |
| `handoff.md` | This document. |
