#### **STAGE 0 — Pre-Snapshot**

**0.1 Brand Data Ingestion**

* Scrape brand URL: Extract Brand Description, Personas, Brand Voice, Product Verticals, Market Segments, Regions  
* Identify: Named personas (Owner, Influencer, End User) with behavioral attributes — not just labels. Product Verticals/Line of Business  
* Map: Direct and indirect competitors per LOB

Product verticals (primary, secondary) are identified using prompt templates that take brand name / urls and market segment.

Product vertical allows us to recommend an initial set of competitors in the same line of business, operating at the comparative scale and region as the focal brand. 

Client taxonomy is built based on the topics and keywords extracted from scraped content (pages, headings, keywords, signals), Product verticals, market segments, Regions.

Handle cross-vertical topics explicitly (e.g., Tax Saving spans *Mutual Funds \+ Insurance*) — tag these as **Shared Cluster** nodes, not duplicated.

Above bits are already laid out in the PRD and nothing special is done in the Digit case, if anything – it was mostly suboptimal in the Digit case study. 

**1\. High Authority Source:** For each vertical, we create a system prompt to surface high authority data sources & refer to them. 

We can give keywords extracted in **step 0.1** as a signal for the LLM to find hot trending sources (threads, subreddits, forums, discussion pages, github issues, quora digests, etc.) and sort them by recency, popularity, upvotes. This can be an additional layer to tap customer conversations that drive nuanced prompts later). The reason for doing it as an additional layer is – we noticed a lot of good nuanced prompts with high volumes but what LLM is doing is it is taking two prompts and merging them: this inflates volume numbers as more buzzwords (keywords) are added – this depth prompt could very well be two different breadth prompts. This is based on the Digit data so proper hypothesis testing is required before productizing the approach. 

**NOTE** 	Some data sources (like academic, aggregator comparison) can change very rapidly based on the brand so an enrichment step (either through system prompts or external apis) needs to be decided on. 

| *Source Type* | *Examples* | *Notes* |
| ----- | ----- | :---- |
| Community & Discussion Platforms | Reddit, Quora, Stack Exchange, Discord, YouTube Comments, X (Twitter) threads | Identified by LLM. |
| Institutional & Official Pages | Brand's own Help Center, Official product documentation, Company newsroom, Wikipedia | Generic sources.  |
| Review & Rating Platforms | G2, Trustpilot, Capterra, Glassdoor, App Store / Play Store reviews, Yelp | Can be maintained by us, rule based or LLM driven. |
| Media & Editorial Sources | Forbes, TechCrunch, The Verge, Bloomberg, BBC, Wired, Hacker News | Identified by LLM (System Prompt) |
| Regulatory & Authority Sources | Government portals (.gov, .org), Standards bodies, Sector-specific regulatory pages, Official circulars | Region / Country specific so the region input in system prompt might help find relevant ones. |
| Academic & Research Sources | Google Scholar, PubMed, arXiv, Statista, McKinsey/BCG reports  | Brand-specific so LLM driven. |
| Video & Multimedia Sources | YouTube explainers, Podcast transcripts, Webinar recordings, … | Can be more or less uniform across product lines as these sources are global distributors.  |
| Aggregator & Comparison Sources | NerdWallet-style vertical aggregators, ProductHunt, SimilarWeb category pages | Becomes brand-specific.  |

**1.3 Funnel Stage Definition:** Build a prompt node against exactly one stage: (For definitions see [Gravton MVP for Customer Demos 2.0](https://docs.google.com/document/d/1-TWrJKy1AFpzncjp9QjK4dzy0VQ2hu4f3g1bKFzdYtI/edit?tab=t.9lbm6alnpxua))

* **Upper Funnel** — Awareness, discovery, education  
* **Mid Funnel** — Comparison, evaluation, shortlisting  
* **Lower Funnel** — Intent, decision, purchase triggers  
* **Post-Purchase** — Retention, support, upsell, claims

Edge case: Some queries span two stages (e.g., "best health plan with easy claims") — tag as **dual-stage**, assign primary stage, preserve secondary tag.

**1.4 Persona Mapping**  (For definitions see [Gravton MVP for Customer Demos 2.0](https://docs.google.com/document/d/1-TWrJKy1AFpzncjp9QjK4dzy0VQ2hu4f3g1bKFzdYtI/edit?tab=t.9lbm6alnpxua)) \- deferred for Digit.

* Owner / Decision maker  
* Influencer  
* User

Funnel Stage and Persona Mapping definitions is passed on to a Prompt Synthesis Model with the intent clusters, keywords, identified sources and verticals to create prompts under two broad categories:

1. Breadth wise prompts:

A **breadth wise prompt** is a prompt designed to cover a **broad area within the same topic/category** in a **simple, general way**, instead of focusing on one detailed situation. It helps you capture the main questions buyers ask across the whole category, so you map the full landscape before going deeper.

2. Depth wise prompts:

A **depth wise prompt** is created by combining a **specific keyword, pain point, constraint, workflow, or buyer scenario** with a focused intent. Instead of covering the whole topic broadly, the prompt drills into a precise sub-problem. A depth-first prompt becomes relevant to real internet conversations because it mirrors the exact language, frustrations, comparisons, and edge cases buyers discuss in communities, reviews, support threads, Reddit posts, Quora questions, YouTube comments, G2 reviews, and help forums.

Based on the keywords provided by Digit and GSC keywords present on the internet, we realized many keywords are semantically similar and overlap. We created a system prompt to bundle these into groups.   
It goes like: 

“Act as a keyword strategist that clusters a flat list of search terms into intent/topic groups for later prompt generation. Each keyword goes in exactly one group.” 

Question: What should be the optimal approach here? 

#### **STAGE 2 — Prompt Generation**

**2.1 Intent Cluster Synthesis \-** For each LOB: Group into **Key Topics** from Intent Clusters & Query Data (GSC) — semantic groupings of what users are actually asking, create unique Key Topics (from single intent clusters or bundle similar clusters)

**2.2 Web search for High Authority \-**  For each Key Topics identified across LOB \- we perform a web search on the Source types (Community & Discussion Platforms, Institutional & Official Pages, Review & Rating Platforms, Media & Editorial Sources, Regulatory & Authority Sources, Academic & Research Sources, Video & Multimedia Sources, Aggregator & Comparison Sources) 

Extract: recurring questions, complaint themes, comparison language, pain points, specific buyer scenarios — grouped by source types, specific source and details extracted from that source. 

LOB | Key Topic | Source Type (Community) | Source (reddit) | Discussions within the Source (reddit/discussions1, reddit/discussions2)

**2.3 Volume-Based Prompt Allocation \-** Rank Key Topics by Volume Estimation and prioritize generating more prompts for higher volume topics. The method for prioritization is defined in Tab 2 [Prompt Synthesis Approach - Release V1](https://docs.google.com/document/d/1-xWH4t0-0PMzmRrl3763d241i6DW3PvdtENQ1wDIkjc/edit?tab=t.z51bjzhsqig7) and final formula in appendix A. 

**2.4 Prompt Generation per Cluster \-** We generate prompts through different models \- Tag each prompt as: `ChatGPT` | `Gemini` | `Claude` | `Grok` | `Perplexity`

1. **Breadth First Prompt Generation**

For each Key Topic, generate wider market and commercial intent prompts specific to the topic, semantically adjust to capture wider intent. We should avoid terms that surface “and”, “or”, “but”. 

2. **Depth Level Prompt Generation**

For each Key Topic, referencing the Authority Pages \- extract recurring themes, pain points, question patterns; then generate prompt variants across: `Funnel Stage, Persona, Sources.` 

Mechanism for automated prompt deduplication is yet to be designed. For more information on system prompts, check the Tabs (System Prompt Schema & System Prompts). 

We can generate as many breadth and depth prompts but some internal capping has to be there after deduplication:

**Prompt distribution (breadth : depth) \= 25:75 (**more depth and fewer breadth. This can be a configurable feature.)

**NOTE This ratio is what we followed for digit and is just a recommendation at this point.**

**Output**

Prompts | Topic | Funnel **|** Persona | Volume

**APPENDIX:**

**A:**

**RECOMMENDED DEFAULTS**

**NOS\_i  \=  SASV\_i / K\_i^0**                γ \= 0  ·  

**w\_i    \=  m\_i · NOS\_i^0.65**          β \= 0.65

**p\_i    \=  floor \+ W\_i × residual**    floor \= 4% of T  ·  cap \= 3× avg

 

**B: Examples of Breadth wise prompts:**

Education / Awareness: Buyer is learning the topic or solution space

“what is \[keyword/topic\]”  
“how does \[product category\] work”  
“benefits of \[keyword/topic\]”  
“types of \[keyword/topic\]”  
“how to understand \[topic\]”

Comparison / Evaluation : Buyer is comparing categories, tools, or vendors

“best \[keyword/topic\] tools”  
“top \[vertical\] software platforms”  
“\[brand\] vs \[competitor\]”  
“alternatives to \[competitor\]”  
“how to choose \[product category\]”

Brand & Reputation : Buyer is validating a company or provider

“how does \[brand\] work”  
“\[brand\] reviews”  
“is \[brand\] worth it”  
“\[brand\] customers”  
“does \[brand\] integrate with \[topic/tool\]”

Pricing & Purchase Intent : Buyer is closer to evaluating vendors

“\[brand\] pricing”  
“cost of \[keyword/topic\] software”  
“\[market segment\] \[LOB\] pricing”  
“\[brand\] implementation timeline”  
“enterprise pricing for \[product category\]”

Post-Purchase / Usage : Buyer already uses the product

“how to set up \[product\]”  
“best practices for \[feature/topic\]”  
“\[product\] not working”  
“how to automate workflows in \[product\]”  
“where to find \[feature\] in \[product\]”

**C: Examples of Depth wise prompts:**

