# Gravton Labs | Insights Engine PRD v2.0 | April 2026

# **1\. Product and Vertical Mapper** 

**User Goal:** Has the system correctly understood my products, services, and business verticals?

**Agent Mechanics:** Crawls the client's website and maps every product and service into a structured taxonomy organized by business vertical. A client like Zoho has distinct verticals (CRM, Helpdesk, Marketing Automation) with different competitor sets and prompt universes for each. The agent builds this multi-vertical taxonomy from the public website, supplemented by client inputs. It is the source of truth for every downstream agent.

**Boundaries:**

1. Does not infer unstated products. Maps only publicly available information plus client inputs.  
2. Does not publish or share the taxonomy externally.  
3. Halts after X failed crawl attempts and alerts GL (Gravton Labs) admin after utilizing backup options. 

**Success:** Client approves taxonomy without structural rework in 8 of 10 onboardings. Zero business verticals left unmapped at handoff.

**Depends on:** Nothing. Runs first.

# **2\. Competitor Mapper** 

**User Goal:** Who are my competitors in each business vertical, and is the system tracking all of them?

**Agent Mechanics:** Maps a distinct competitor set for each of the client's business verticals using the product taxonomy. Competitors are confirmed by the client at onboarding (with modifications). For MVP, the competitor set is static: the client provides names or URLs and the system maps them. Dynamic detection (surfacing new competitors that appear in LLM results but are not yet tracked) is P1.

**Boundaries:**

1. Does not add competitors without human confirmation.  
2. Enforces the client's package limit on total tracked competitors.  
3. Does not infer competitors. Maps only what is publicly identifiable or client-provided.

**Success:** Client approves 8 of 10 entries without edits.

**Depends on:** Product taxonomy from A1.

# **3\. Brand Context Crawler** 

**User Goal:** Does the system understand my brand well enough to produce relevant results from the start?

**Agent Mechanics:** Crawls the client's website at onboarding to auto-generate the Brand Kit: company name, description, market segment, key topics, brand voice, and target personas. All generated fields are editable by the client. This Brand Kit feeds the Context Page used by content and compliance agents downstream.

**Boundaries:**

1. Falls back to manual input if the URL returns empty HTML (e.g., bot-blocked). Explains why.  
2. Recrawl does not replace existing fields. New suggestions require client confirmation.  
3. Halts after X failed crawl attempts and routes to manual input.

**Success:** All Brand Kit fields populated or flagged for manual input on every onboarding. Crawl completes without fallback in 85% of onboardings. 

**Depends on:** Nothing. Runs at onboarding.

# **4\. Prompt Enrichment** 

**User Goal:** What are the real questions my customers are asking, in their own language, across every stage of the buying journey and how much?

**Agent Mechanics:** Discovers real customer language across four input layers simultaneously. 

Layer 1: pulls keyword and search data from DataForSEO and Google Trends to establish verified search demand, supplemented by optional client SEMrush/Ahrefs exports. 

Layer 2: maps competitor products and services against the client taxonomy to identify related demand the client may be losing. 

Layer 3: scans high-authority sources relevant to the client's vertical using the verticalized source configuration (e.g, Capterra) to capture conversational language, reviews, and real customer phrasing. 

Layer 4: ingests optional client-uploaded documents (sales decks, support tickets, product briefs) to surface problem framing that neither keyword tools nor scrapers find. Clients can later connect APIs to internal ticketing systems for continuous ingestion.

The agent produces prompt candidates clustered by journey stage (awareness, evaluation, comparison, decision), tagged by product area and source type, and assigns (estimated) prompt volume. A human filters navigational and brand-name queries, reviews the output, and selects the final prompt library. No prompt enters the library without human approval.

**Boundaries:**

1. Does not access private or gated data. Alerts admin if encountered.  
2. Does not auto-promote any prompt to production.

**Success:** Zero duplicates in output. 100% of prompts attributed can be traced to a source. Human approves 8 of 10 prompts without substantial edits. Baseline established in first 20 client runs.

**Depends on:** Product taxonomy. Verticalized source configuration.

# **5\. Search Data Ingestion** 

**User Goal:** Is the system using my actual search performance data to inform its analysis?

**Agent Mechanics:** Pulls Google Search Console data through direct API integration or file upload, subject to row limits. Surfaces queries, impressions, CTR, and position data per keyword. Clients can also upload exports from SEMrush or Ahrefs for keywords they already track. All ingested data feeds into prompt enrichment and scoring.

**Boundaries:**

1. Reads (only) search data from verified accounts only   
2. Re-attempts if the ingestion breaks or credentials expire.

**Success:** 100% of scheduled pulls completed. 

**Depends on:** Nothing. Runs on schedule independently.

# **6\. Competitor Intelligence** 

**User Goal:** Where are my competitors showing up in AI results, and how much of the conversation do they own?

**Agent Mechanics:** Runs scheduled prompt sweeps across LLMs to track competitor visibility and share-of-voice per product area per business vertical. Produces competitor visibility scores so the client can compare against all tracked competitors on SOV, rank, position, and sentiment, broken down by LLM platform.

**Boundaries:**

1. Does not run prompts outside the client's approved product taxonomy.

**Success:** 100% of approved prompts run each cycle. 

**Depends on:** Product taxonomy and Competitor map.

# **7\. Citation Monitor** 

**User Goal:** Are my citations still being referenced by LLMs, or have I lost ground?

**Agent Mechanics:** Tracks whether the client's published pages continue to appear as citation sources in LLM responses over time. For each page, the agent runs the prompts that originally triggered the citation across ChatGPT, Gemini, Perplexity, and Claude on a weekly cadence. It logs whether the page is still cited, flags drops, identifies which source replaced it, and categorises each citation by type: earned (publications, PR), community (Reddit, G2, reviews), and owned (client's own pages).

**Boundaries:**

1. Monitors and reports only. Does not modify content.  
2. Does not report a drop without 2 runs (to avoid false alarms or LLM response fluctuations) 

**Success:** 100% of scheduled queries run. Drop patterns surfaced within 24 hours of detection.

**Depends on:** Published prompts list. Approved LLM platforms.

# **8\. Emerging Topics / Competitors / Prompt Clusters**

**User Goal:** Where is the market headed? Which topics are rising, which are fading, and what is emerging that I should act on before competitors do?

**Agent Mechanics:** Tracks LLM visibility trend lines over time per topic and vertical. Provides weekly, monthly, and yearly trend views up to a year back. Draws on external signals (search trends, competitor activity) and internal systems (ticketing data where connected) to surface direction. Also identifies emerging topics, emerging competitors, and emerging prompt clusters that are gaining volume but are not yet in the client's tracked set. Flags these for human review and potential addition to the prompt library.

**Boundaries:**

1. Does not generate trend lines for topics with fewer than 4 weeks of data.

**Success:** Identified topics/competitors/prompts get accepted by clients for tracking 80% of the times.  

**Depends on:** Prompt, Competitive data, and Product taxonomy 

# **9\. Earned Media and Brand Presence** 

**User Goal:** Which publications and sources do LLMs actually cite in my category, and where do I have no presence?

**Agent Mechanics:** Builds a category-level ranked source map showing which third-party publications, review sites, expert sources, and high-authority sites LLMs cite most frequently in the client's category. Categorises sources by type: news, expert, community, retailer. Maps where the client has presence versus where they do not, both as LLM citation sources and as brand mention locations. Surfaces where competitors are gaining citations and how much visibility they hold on each source. Gaps are flagged as PR, content, or outreach opportunities.

**Boundaries:**

1. Does not surface sources below a configured citation frequency threshold.  
2. Halts if citation monitor data is unavailable.

**Success:** 100% of cited sources catalogued. 

**Depends on:** Citation data. Client prompt set.

# **10\. Presence Mapper** 

**User Goal:** Where does my brand show up in AI results across every LLM, for every prompt that matters?

**Agent Mechanics:** Runs every approved prompt across ChatGPT, Gemini, Perplexity, Claude, Meta AI weekly., and more. Cadence and platform selection are adjustable per client. For each prompt on each platform, it records whether the brand is cited, at what position, with what sentiment, which source the LLM used, and in what format (article, video, product page, review, structured data). Produces a presence matrix (prompt x platform) that the scoring engine and analytics agents use to prioritize gaps. Metrics include SOV, rank/position, and sentiment per prompt per platform. Also captures query fan-outs: the related queries and sub-prompts that branch from each primary prompt.

**Boundaries:**

1. Records and reports only. Does not modify content or trigger actions.  
2. Does not run prompts outside the approved library.  
3. Does not report presence below minimum confidence threshold.  
4. Weekly default. On-demand sweeps deferred to post-MVP.  
5. Halts if the prompt library is empty or inactive.

**Success:** 100% of approved prompts swept each cycle. 100% of configured platforms queried. Zero results below confidence threshold.

**Depends on:** Approved prompt library. Regional controls

# **11\. Persona Mapper** 

**User Goal:** Which personas are asking which questions, and am I winning or losing differently across decision-makers versus end users?

**Agent Mechanics:** Segments the approved prompt library by persona: decision-makers, influencers, users, champions, and any client-defined persona types. Different personas ask fundamentally different questions about the same product at the same stage. A CMO asks "what is the ROI of X" while an end user asks "is X easy to set up." Both are evaluation-stage prompts, but they trigger different LLM responses and require different content. The agent tags each prompt with its inferred persona, so the demand map, presence matrix, and scoring all carry persona as a first-class dimension.

**Boundaries:**

1. Tags and segments only. Does not generate content or recommendations.  
2. Does not assign a persona without minimum confidence. Flags ambiguous prompts for human review.

**Success:** Zero low-confidence assignments surfaced without flagging. Human accepts persona tags 80% times and does not have to override. 

**Depends on:** Approved prompt library. Client persona definitions from onboarding inputs.

# **12\. Verticalized Source Configuration**

**What this is:** A pre-configured, editable list of priority sources for social listening per industry and region. For SaaS, the default list includes Reddit, G2, YouTube, LinkedIn, and Capterra. For retail, it shifts to Reddit, YouTube, Instagram, Amazon, LinkedIn, TrustPilot, and others. Each vertical and region combination has its own source list. If no pre-configured list exists for a client's vertical, the system identifies the right sources before generating the demand map, flags additions for Gravton admin review, and updates the list over time. Clients can also provide their own list of sources.

**How it is used:** Prompt Enrichment reads this configuration to determine which sources to scan for Layer 3\. The configuration is maintained by Gravton admins, and keeps growing from time to time, depending upon more clients onboarded. Clients can request additions but cannot modify the list directly in MVP. Admin can track additions to the list over time. 

# **13\. Regional Controls**

**What this is:** A system-level configuration that determines the geographic context for prompt runs and source scanning. A client operating in North America sees LLM results and source data scoped to that region. A client operating in the EU or India sees region-appropriate results. Regional controls affect which region prompts run in), which regional sources are scanned, and regional competitor sets.

**How it is used:** Set at onboarding. The client specifies their primary markets and regions. The system scopes all downstream analysis to those regions. Multi-region support (e.g., a client tracking both North America and EU simultaneously) is supported but increases prompt volume and credit consumption proportionally.

# **14\. Snapshot / Demand Map**

**User Goal:** Has the system correctly understood my market, my competitors, and where I stand before it starts identifying gaps?

**What this is:** An aggregated output view produced after Foundation and Intelligence agents complete their first cycle. The Snapshot shows: topic clusters, key competitors per vertical, brand tonality, top citation sources, overall demand universe volume, and how much the client currently captures at each funnel stage. The client confirms or course-corrects conversationally before gap identification begins.

For large clients with hundreds of topics and tens of thousands of prompts, the system imposes scope limits per vertical or processes in chunks. The Snapshot is a checkpoint, not a final answer. Over time, as agents pull more data, the demand map gets richer and directionally more accurate within defined confidence ranges.

**What it aggregates from:** Product taxonomy, competitor map, prompt candidates, search data, competitor intelligence, presence matrix, trend data, earned media source map, persona tags (when available).

**What it does not do:** Does not identify gaps or recommend actions. That is the Opportunity Engine's job. The Snapshot exists to build client confidence that the system sees their market correctly before any gap analysis runs.

# **15\. Platform Controls**

**What this is:** The ability for the client (or Gravton team on behalf of the client) to add or remove prompts, competitors, and tracked LLM platforms.

**MVP scope:** Gravton admins manage additions and removals on behalf of clients. The data model supports client-level controls so the feature can be opened up without re-architecture.

# **16\. Backlink Opportunity** 

**User Goal:** Which high-authority sites should I target for backlinks and guest posts to strengthen my citations?

**Agent Mechanics:** Identifies high-authority backlink and guest post targets relevant to the client's category. Scores each target by domain authority and relevance. Recommends outreach angles based on the client's brand mention gaps and earned media source map.

**Boundaries:**

1. Research and scoring only. Does not draft outreach or contact any third party.  
2. Recommends only publicly accessible domains.

**Success:** 100% of recommendations include domain, relevance score, and suggested angle. 4

**Depends on:** Product taxonomy. Brand mentions gaps. Earned media source map.

# **17\. Adoption Health Prompt Discovery**

**User Goal:** What are my existing customers asking about my product post-purchase, and where are they struggling?

**Agent Mechanics:** Builds a separate post-purchase prompt library distinct from the acquisition prompt library. Sources prompts from three inputs: client-provided ticket data, support forums, and community sites; client-specified topics and reference sites they consider important; and system-generated candidates based on the product taxonomy (if we know the product, we can infer common post-purchase questions like integration issues, setup problems, feature comparisons for upgrades). The agent clusters prompts by post-purchase topic (e.g., "CRM sync and integration failures," "workflow automation setup," "upgrade evaluation"). Client approves the final library. No prompt enters the post-purchase library without human approval.

**Boundaries:**

1. Does not mix post-purchase prompts with the acquisition prompt library. Separate libraries.  
2. Does not auto-promote any prompt to production.  
3. Does not access private or gated client data without explicit authorization.  
4. Halts if product taxonomy is absent.

**Success:** Zero duplicates. Client approves 8 of 10 prompts without substantial edits. Baseline established in first 10 client runs.

**Depends on:** Product taxonomy. Client ticket data or topic inputs. Verticalized source configuration.

# **18\. Adoption Health Monitor**

**User Goal:** Are AI platforms providing my existing customers with consistent, positive, and properly sourced answers about my product?

**Agent Mechanics:** Runs every approved post-purchase prompt across ChatGPT, Gemini, Perplexity, Claude, and Meta AI on a weekly cadence (configurable). For each prompt on each platform, it tracks three dimensions. First, consistency: are responses coherent across LLMs, or are different platforms giving conflicting answers about the same product issue? Inconsistent responses confuse end customers and erode trust. Second, sentiment: is the post-purchase conversation framed positively or negatively? The client wants AI responses about their product experience to be largely positive. If LLMs are surfacing negative framing, the client needs to know and act. Third, source ownership: are responses sourced from owned properties (client help docs, knowledge base, official pages) or from external forums, Reddit threads, and third-party complaint sites? The client wants owned sources to dominate so they control the narrative.

The agent flags when external sources dominate owned sources on a topic, when sentiment turns negative or worsens over time, or when responses are inconsistent across platforms. It surfaces volume and growth trends per post-purchase topic (e.g., "Failure State prompts volume growing at 12.5%") so the client can see which problem areas are escalating.

**Boundaries:**

1. Monitors and reports only. Does not modify content or trigger actions.  
2. Does not run prompts outside the approved post-purchase library.  
3. Does not report results below the minimum confidence threshold.  
4. Halts if the post-purchase prompt library is empty.

**Success:** 100% of approved post-purchase prompts swept each cycle. All three dimensions (consistency, sentiment, source ownership) tracked per prompt per platform. Alerts surfaced within 24 hours when thresholds are breached.

**Depends on:** Approved post-purchase prompt library from Adoption Health Prompt Discovery. Regional controls.

