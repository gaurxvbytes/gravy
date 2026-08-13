# Catalogue Enrichment \- Shopping v0.9

# **Summary**

Conversational attributes are optional product feed fields in the [Google Merchant Center](https://support.google.com/merchants/answer/17085370?hl=en-IE) designed to help AI shopping assistants and conversational search tools understand item nuances. Wildcard’s core product feature builds on this to offer “catalogue enrichment” using conversational attributes with product facts, keywords, and buyer intents. 

This proposal explores Catalogue Enrichment as a method for clients to optimise and build their PDP to be referenced during AI response generation, and implement changes on the pages that influence the AI responses. 

**Recommendation.** We are building a system for users to evaluate, enrich, and publish product descriptions and catalogue copy,t which will win AI responses on lost attributes and mentions. These surface within the Opportunities as Catalogue enrichment, where users evaluate the enriched phrases and draft \- with highlighted changes. Enriched phrases for the product are built from ingested PDP and catalogue data, and grounding methods using web search (community, reviews, and retailer platform comments for the products) and inferences from the AI responses and prompts that are relevant. Users can approve these enriched phrases in the draft to finalize it, and export like other content options. 

# **Product Experience** 

1. **Ingesting catalogue data.** We ingest the product catalogue details using user-uploaded catalogue documents, connectors \- Shopify Store, Google Merchant Center. We also identify attributes linked to the product through web search (community, reviews, and user comments on retail sites) and attribute excerpts from our responses.

2. **Map identified and missing attributes.** We categorise all of the attributes for the product using the following:

   1. Attributes in your product, in the response – from PDP, catalogue details, web search, and excerpts linking from the AI responses. 

   2. Attributes in your product, missing in the response – from the PDP, Catalogue details, and web search. 

3. **Building Conversational Attributes.** We build conversational attributes for the products based on the product’s details and specifications ingested and the attribute (identified and missing) mapped. The conversational agents are built based on the parameters declared under attributes in Merchant Centre and details missing on the description content. 

   1. **Functions & Package.** Conversational attributes are phrases in the format of responses to buyer intent, use cases, comparisons, key topics, and phrases within the relevant prompts. These attribute types can be:

      1. Product Description content. (Content Engine capability)

      2. [Question and answer \[question\_and\_answer\]](https://support.google.com/merchants/answer/17085211), (Content Engine Capability)

      3. [Document link \[document\_link\]](https://support.google.com/merchants/answer/17084656) (Suggested Action Item)

      4. [Related product \[related\_product\]](https://support.google.com/merchants/answer/17085213) (Suggested Action Item)

      5. [Item group title \[item\_group\_title\]](https://support.google.com/merchants/answer/17085146) (Suggested Action Item)

      6. [Variant option \[variant\_option\]](https://support.google.com/merchants/answer/17085214) (Suggested Action Item)

      7. [Popularity rank \[popularity\_rank\]](https://support.google.com/merchants/answer/17085297) (Suggested Action Item)

      

4. **Catalogue Enrichment Opportunity.** Gravton surfaces within the Opportunities the Catalogue Enrichment opportunity, where users evaluate the enriched phrases in a draft. The draft comprises Rational, recommended phrases highlighted in the description content, Action item list of additional attribute components to add (Question answer, document link, variant option, etc). Users can approve these enriched phrases in the draft to finalize it, and export like other content options. Output Formats – Formatted Text and schema markup format.

   1. **Trigger for Catalogue Opportunity.** The engine triggers a catalogue enrichment opportunity on the occurrence of the following:

      1. The attribute was present in a competitor's answer and not in ours.

      2. The attribute was missed/not attached to our product

      3. Drop in citations for our catalogue pages

   2. **Integration to Google Merchant Centre.** Currently, we do not have a dedicated integration to Google Merchant Centre, but our Content Engine builds the content for the attribute components in a readable and acceptable format for the Google Merchant Centre. Users can export the content and appropriate schemas to the content type

5. **Catalogue Enrichment Content.** The content engine utilizes the recommended specifications for the attribute types to build the attributes for the brand. Every catalogue enrichment opportunity generates:

   1. A refined product description copy, with added text highlighted.

   2. Attribute components (text \+ schema code) based on the action items. 

The content studio agent follows the Merchant Centre guidelines and builds the appropriate content. Output Formats – Formatted Text and schema markup format.

### ***Decisions***

1. \[P1 SCOPE\] Integrate changes directly to the Google Merchant Centres \- attribute field areas.   
2. \[P2 SCOPE\] Product Descriptions and details are time-based and updated based on Keyword Volume research. This problem is also part of catalogue enrichment. Solving for this problem would require us to automate recommendations for Product Description updates, based on an Automated Keyword Research tool.   
3. \[P1 SCOPE\] Fact-checking and Identification of Hallucinated responses within product definitions is crucial, as AI can sometimes link false attributes to the product, which violates compliance and other strict standards to be followed by the brands. Identification and resolution of this is an important scope for a brand, which rather invests efforts in manual resolution and checks (backlink checks & manual searches). 

