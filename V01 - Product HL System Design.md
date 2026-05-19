# Gravton Labs | Product System Design Doc | May 2026

**1\. System Overview**

Gravton is a multi-DAG AI visibility intelligence platform. It ingests data from brand websites and Google Search Console to synthesizes prompt libraries; fires prompts against live LLM platforms; and surfaces citation gaps and content opportunities. All pipeline work is coordinated through Django, backed by Airflow with CeleryExecutor and Redis as the task broker. PostgreSQL is the canonical data store; S3 holds large artifacts.

**Overall DAG execution order**

**[ArchitectureV01.png](https://drive.google.com/file/d/10pT8Zx2xXVQ0LWzw1NpJvKkEgL7uv7je/view?usp=drive_link)**

**2\. Brand Foundation** 

Brand Foundation is the system's first step; it reads the brand's public website and builds a structured understanding of who the brand is before any analysis begins.

Everything produced here, the brand snapshot, taxonomy, competitor map, and persona set, becomes the shared context that every downstream module reads from. Nothing runs until the Brand Foundation is complete and the client has confirmed the Snapshot.

**Flow**

Brand URL

  ↓  Website Crawl   — discover all pages, extract content, classify page type

  ↓  Brand Kit       — name, description, market segment, brand voice, key topics,

  │                    brand snapshot (3–5 sentence narrative used as context downstream)

  ↓  Product Taxonomy — verticals → products (cross-vertical topics tagged as shared nodes)

  ↓  Competitor Map  — client-confirmed competitor set per vertical

  ↓  Persona Map     — Decision Maker, Influencer, End User, Champion \+ behavioural attributes

  ↓  Regional Controls — geographic scope for all downstream prompt runs and scans

  ↓  Source Config   — which platforms to listen to per vertical/region

                        (SaaS: Reddit, G2, YouTube, LinkedIn, Capterra)

  ↓

  ► Snapshot (client approval gate before prompts run):

    topic clusters per vertical · key competitors per vertical · brand tonality · personas

    Client confirms or course-corrects. Gap identification does not begin until approved.

**Dags associated are Crawl dag and Brandkit dag** 

* Crawl DAG reads the brand's public website. That's its only job. It discovers all pages, extracts the content from each one, and classifies every page by type: product, pricing, blog, help doc, comparison page, etc. The output is a structured page library that Brandkit\_dag reads from. 

* Brandkit dag takes the crawled page library and builds structured brand knowledge in one sequential run — Brand Kit (name, segment, voice, topics, brand snapshot), Product Taxonomy (verticals → products), Competitor Map (client-confirmed per vertical), Persona Map (Decision Maker, Influencer, End User, Champion with behavioural attributes), Regional Controls, and Source Config (which platforms to listen to per vertical/region). It ends with a snapshot, a consolidated view of topics, competitors, and personas per vertical shown to the client for confirmation. Nothing downstream runs until the client approves it.

**3\. Data Ingestion**

The data ingestion DAG pulls two types of data from Google Search Console — via direct API integration or client file upload (GSC export, SEMrush, or Ahrefs exports). Subject to row limits.

**Keyword-level data** — queries, impressions, CTR, and position per keyword. This feeds prompt enrichment and scoring — surfacing real search language to build better prompts.

**Page-level data** — URL, Title, Clicks, Impressions, CTR, and Position per brand page.  
Runs once in full at onboarding, then refreshes on schedule.

**Flow**

GSC API / File Upload

        ↓

Keyword Ingestion

        ↓

Conversational → Keep

Short-tail → Review

Navigational → Discard

        ↓

Score \+ Rank Keywords

        ↓

Generate Prompt Candidates

        ↓

Ranked Keyword Library

**4\. Prompt Synthesis**

Prompt Synthesis step builds the active prompt library its has synthesis\_prompt dag and adoption health dag. It takes the brand snapshot, product taxonomy, external search demand signals, and optionally GSC keywords and client-uploaded documents. It groups all signals by meaning into Intent Clusters, labels each one, assigns a funnel stage (Upper / Mid / Lower / Post-Purchase), and scores them by estimated demand volume so higher-demand topics get more prompts allocated to them. Allocation uses power-law dampening, so no single topic monopolises the budget; every cluster gets a guaranteed floor, and no cluster exceeds a set cap.

For each cluster, two prompt types are generated: breadth (broad questions mapping the topic landscape) and depth (specific pain points, comparisons, and buyer scenarios drawn from real customer language). Default ratio is 25% breadth, 75% depth, configurable per client.

Cross-vertical topics are explicitly detected and tagged as Shared Cluster nodes — a question like "how do I save tax efficiently" spans Mutual Funds and Insurance and gets one prompt tree associated with both verticals, never duplicated. Near-duplicate prompts across the library are collapsed, with the higher-scoring version kept.

No prompt enters the active library without human approval. Once approved, every prompt is tagged with funnel stage, persona, product vertical, region, and breadth/depth type — these tags drive the presence matrix, gap analysis, and opportunity feed.[Prompt Synthesis Approach - Release V1](https://docs.google.com/document/d/1-xWH4t0-0PMzmRrl3763d241i6DW3PvdtENQ1wDIkjc/edit?tab=t.wt9s7fz6qjvx)

**Flow**

Brand Snapshot \+ Product Taxonomy

  \+ GSC conversational keywords (optional)

  \+ Client-uploaded docs/sales decks/support tickets (optional)

  \+ External search demand signals (DataForSEO, Google Trends)

  ↓

  Intent Cluster Synthesis

  — Group all signals by meaning into topic clusters (Intent Clusters)

  — Label each cluster, assign funnel stage: Awareness / Evaluation / Decision / Post-Purchase

  — Merge near-identical clusters. Cross-vertical duplicates → single Shared Cluster node

  ↓

  Volume Allocation

  — Score clusters by estimated demand volume

  — Power-law dampening (β=0.65): high-volume topics don't monopolise budget

  — Floor: 4% of total budget per cluster. Cap: 3× average per cluster

  ↓

  Prompt Generation per cluster  (ratio: 25% breadth / 75% depth)

  — Breadth: broad general questions mapping the topic landscape

  — Depth:   specific sub-problems, pain points, buyer scenarios

  — Tag each prompt: funnel stage, persona, vertical, region, type

  — Deduplicate: near-identical prompts collapsed, higher-scoring kept

  ↓

  Human Approval Gate  →  Active Prompt Library


adoption\_health\_prompt\_dag builds the post-purchase prompt library — separate from the acquisition library, never mixed with it.

The primary classifier is first-person product ownership language. A prompt is assigned here when ownership signals are present — "my \[product\]", "our setup", "we're using". A prospect researching a named vendor without an ownership context routes to acquisition instead.

Sources prompts from three inputs: client-provided ticket data, support forums, and community sites; client-specified topics and reference sites; and system-generated candidates inferred from the product taxonomy. Prompts are clustered by post-purchase topic — setup, integration failures, workflow automation, and upgrade evaluation. Client approves the final library before anything runs.

**5\. Authority Source Layer**

Reddit, Quora, and YouTube are the three most-cited authority sources across LLMs — they are where real customer conversations happen, where buying decisions are influenced, and where LLMs go to construct answers. This layer monitors brand and competitor presence across all three, tracking what is being discussed, how customers frame problems, where the brand appears or is absent, and where competitors are displacing it.

Each source runs as an independent DAG on its own cadence. Discovery for all three works the same way: a pre-seeded list of known citation-heavy communities per vertical, augmented by the brand's product taxonomy at onboarding, and continuously updated by a citation feedback loop — any Reddit, Quora, or YouTube URL that appears in an LLM response tracked for the brand is automatically queued for that source's DAG if it is not already tracked.

All three DAGs produce the same three outputs: an identification library (communities, questions, or videos categorised by vertical, funnel stage, and persona), an insights layer (engagement metrics, brand and competitor mentions, sentiment), and inputs to the Opportunity Engine for content, engagement, and positioning recommendations.

**Flow**

Inputs: Brand Snapshot \+ Product Taxonomy \+ Source Config \+ Citation feedback loop

 ↓

reddit\_dag Daily — hot / rising / new per tracked subreddit Weekly — top-all / top-year \+ citation-confirmed re-scrape Outputs: Subreddit \+ thread library → Identification / Insights / Opportunities

quora\_dag Weekly — Most Recent / Most Upvoted / All Time per tracked Space \+ citation-confirmed re-scrape Outputs: Space \+ question library → Identification / Insights / Opportunities

youtube\_dag Daily — search.list (new \+ trending by view velocity) Weekly — videos.list \+ transcripts \+ playlists \+ citation-confirmed re-scrape Outputs: Channel \+ video library → Identification / Insights / Opportunities

**6\. Intelligence Engine**

The Intelligence Engine is where Gravton measures actual brand presence in AI. Every approved prompt from both the acquisition library and the post-purchase library is fired across all configured LLM platforms in a single sweep. Responses are collected and analysed across three dimensions: who is cited, how the brand is framed, and what related questions the LLM generated internally as part of its response. These LLM-generated questions (query fanouts) are fed back into Prompt Synthesis as new prompt candidates, growing the prompt library over time based on what LLMs themselves surface as relevant to a topic. Acquisition and post-purchase prompts run together but are tagged separately, so their outputs route to the right downstream modules.

**responses\_dag** Fires all active prompts — acquisition and post-purchase — across all configured LLM platforms on a weekly cadence. Multiple repetitions per prompt per platform to account for LLM response variability. Every response is tagged at the point of collection: acquisition or post-purchase. This tag determines where the output goes downstream.

— Active prompt library (acquisition prompts: Awareness → Decision) — Post-purchase prompt library (Setup · Integration · Feature usage · Upgrade) — Both fire across: ChatGPT · Gemini · Perplexity · Claude · Meta AI — Multiple repetitions per prompt per platform — Tag on each response: acquisition or post\_purchase

**citation\_dag** — runs on acquisition responses. Who is cited and how much across all LLM platforms, Brand SOV \+ competitor SOV per prompt per platform. Source type classification: owned · earned · community · editorial Position-weighted citation scoring New Reddit / Quora / YouTube URLs found in citations → fed back to Authority Source Layer discovery queues

**sentiment\_dag** — runs on all responses. How the brand is framed per response: positive/negative/neutral/mixed Acquisition responses → Presence Matrix · Opportunity Engine Post-purchase responses → Adoption Health Monitor

**query\_fanout\_dag** — runs on acquisition responses Related questions and sub-prompts surfaced by LLMs in responses Sufficiently different queries flagged as new prompt candidates → fed back to Prompt Synthesis next cycle

**Adoption Health Monitor** — runs on post-purchase responses. Tracks three dimensions per prompt per platform:

1. Consistency — are responses coherent across LLMs, or are different platforms giving conflicting information about the same product issue?

2. Sentiment — is the post-purchase conversation framed positively or negatively? Is it worsening over time?

3. Source ownership — are responses sourced from owned help docs and knowledge base, or from external forums, Reddit threads, and third-party complaint sites?

All downstream DAGs run in parallel after each responses\_dag sweep completes.

Acquisition outputs → Opportunity Engine · Presence Matrix · Prompt Synthesis (next cycle) Post-purchase outputs → Adoption Health Monitor · Opportunity Engine (Adoption Health opportunity type)

**Flow**

responses\_dag

— Active prompt library (acquisition)

— Post-purchase prompt library

— Both fire across: ChatGPT · Gemini · Perplexity · Claude · Meta AI

— Tag on each response: acquisition or post\_purchase

↓

citation\_dag \+ sentiment\_dag \+ query\_fanout\_dag

— Acquisition responses → Presence Matrix · Opportunity Engine

— Post-purchase responses → Adoption Health Monitor

**7\. Insight Dag**

insights\_dag aggregates all outputs from every upstream module after each intelligence cycle completes — citation, sentiment, fanout, opportunity, and authority source data — and computes the metrics and views the client sees in the dashboard. This includes the Demand Map (Winning / Sharing / Losing / Untapped), presence matrix, citation performance, source distribution, authority source performance, adoption health, and the opportunity feed.

**8.Opportunity Engine(suggestive)**

After the Foundation and Intelligence agents complete their first cycle and the client confirms the Snapshot, the Opportunity Engine begins identifying gaps and generating recommendations. It does not run until the client has confirmed the system has correctly understood their market.

The engine takes inputs from all upstream modules — citation data, presence data, sentiment data, GSC performance data, Reddit insights, Quora insights, and YouTube insights — and identifies where the brand has gaps across five dimensions: where competitors are cited and the brand is not, where the brand has no visibility across platforms, where competitors are gaining citations the brand is losing, where the brand's content does not cover the answer space, and where the brand ranks in Google search but is absent from AI responses.

These gaps are routed to an Opportunity Agent — a reasoning model that synthesises structured briefs across eight opportunity types: Content, SEO, Citation and Backlink, Reddit, Quora, YouTube, Technical SEO, and Adoption Health.

Every recommended action requires a human decision. The Opportunity Engine monitors and recommends only.

**Flow**

citation\_dag · sentiment\_dag · responses\_dag

gsc\_dag · reddit\_dag · quora\_dag · youtube\_dag

  ↓

opportunity\_dag — Gap scoring

Citation gap · Presence gap · Competitor displacement

Semantic gap · GSC vs AI delta

  ↓

Opportunity Agent (reasoning model)

Content · SEO · Citation & Backlink

Reddit · Quora · YouTube

Technical SEO · Adoption Health

  ↓

Structured opportunity briefs

Human decision is required before any action

