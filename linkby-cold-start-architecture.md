# Cold Start Architecture: Onboarding Intelligence for Linkby

> **Author:** Alexander Kwok | **Date:** April 2026 | **Status:** Architecture Draft
> **Problem:** When a new advertiser or publisher joins Linkby, they have zero history. We need to: (1) generate a hyper-personalized campaign brief that matches their brand voice AND what publishers want to promote, and (2) recommend a starting CPC/CPM that's attractive enough for publishers to pick up but doesn't overspend. Both need to happen in the first session.

---

## 1. The Cold Start Problem — Both Sides of the Marketplace

### New Advertiser Joins

They don't know:
- What CPC/CPM to set (too low = no publishers pick up, too high = wasting budget)
- How to write a brief that publishers actually want (most briefs are advertiser-centric, not publisher-friendly)
- Which placement type works best for their product (listicle vs pubfeed)
- Which publisher verticals will work (they default to their own vertical, missing cross-sector opportunities)

We don't know:
- Their conversion benchmarks (we don't track full-funnel conversions anyway)
- Their actual audience (only what they tell us)
- Their brand voice and content style

### New Publisher Joins

They don't know:
- Which campaigns to accept (they pick based on vibes, not data)
- What CPC/CPM range they should expect for their vertical/audience
- How to position their pitch to advertisers

We don't know:
- Their real audience composition (self-reported vs actual)
- Their content engagement patterns
- Their fulfillment reliability

### The Compounding Problem

Cold start is worst at the beginning and gets better with every transaction. But if the first experience is bad (advertiser overpays, publisher picks wrong campaign, brief doesn't convert), they churn before the system can learn. **The first campaign has to be good enough to create a second campaign.**

---

## 2. Cold Start Resolution Strategy: "Warm Start via Graph Neighbours"

The core insight: **a new advertiser has zero history, but advertisers who look like them have rich history.** The knowledge graph lets us transfer intelligence from similar entities.

### 2.1 New Advertiser Onboarding Flow

```
Step 1: Intake
  Advertiser provides: brand name, website URL, industry, product type,
                       target audience, budget range, campaign goals

Step 2: Graph Neighbour Discovery
  ┌─────────────────────────────────────────────────────────┐
  │  LLM extracts: brand voice, key selling points,        │
  │  content themes from advertiser's website/socials       │
  │                    │                                    │
  │                    ▼                                    │
  │  Generate embedding → find K nearest advertisers        │
  │  in the graph (by industry + embedding similarity)      │
  │                    │                                    │
  │                    ▼                                    │
  │  Pull neighbour data:                                   │
  │  - Their successful campaigns (CTR > benchmark)         │
  │  - Brief text that publishers accepted                  │
  │  - CPC/CPM that got pickup                              │
  │  - Which publishers worked for them                     │
  │  - Which placement types performed                      │
  │                    │                                    │
  │                    ▼                                    │
  │  Transfer to new advertiser as "warm priors"            │
  └─────────────────────────────────────────────────────────┘

Step 3: Brief Generation (LLM-powered)
  Using graph context + brand voice → generate campaign brief

Step 4: CPC/CPM Recommendation
  Using graph neighbour pricing + publisher supply/demand → suggest pricing

Step 5: Publisher Matching
  Using graph-based matching → recommend top publishers
```

### 2.2 New Publisher Onboarding Flow

```
Step 1: Intake
  Publisher provides: domain/platform, vertical, audience description,
                      content examples, desired campaign types

Step 2: Graph Neighbour Discovery
  ┌─────────────────────────────────────────────────────────┐
  │  Crawl/scrape publisher content (3-5 recent articles)   │
  │  LLM extracts: content themes, tone, audience signals   │
  │                    │                                    │
  │                    ▼                                    │
  │  Generate embedding → find K nearest publishers         │
  │  in the graph (by vertical + embedding + audience)      │
  │                    │                                    │
  │                    ▼                                    │
  │  Pull neighbour data:                                   │
  │  - Campaigns they successfully fulfilled                │
  │  - CPC/CPM ranges they typically earn                   │
  │  - Advertiser verticals that worked for them            │
  │  - Placement types that performed                       │
  │  - Avg fulfillment time                                 │
  └─────────────────────────────────────────────────────────┘

Step 3: Campaign Recommendations
  "Here are 5 active campaigns that match your content and audience"

Step 4: Set Expectations
  "Publishers like you typically earn $X CPC / $Y CPM in this vertical"
```

---

## 3. LLM Brief Generation Pipeline

This is the highest-impact cold start feature. A well-written brief that matches the advertiser's brand voice AND what publishers want to promote dramatically increases pickup rate.

### 3.1 Architecture

```
                    ┌──────────────────────┐
                    │  Advertiser Intake    │
                    │  (brand, goals, etc.) │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Brand Intelligence   │
                    │  Extraction           │
                    │  (LLM crawls website, │
                    │   extracts voice,     │
                    │   USPs, tone)         │
                    └──────────┬───────────┘
                               │
                ┌──────────────┼──────────────┐
                │              │              │
     ┌──────────▼─────┐  ┌────▼────┐  ┌──────▼──────────┐
     │ Graph Neighbour │  │ Target  │  │ Publisher        │
     │ Successful      │  │ Topic   │  │ Preferences      │
     │ Briefs          │  │ Context │  │ (what similar    │
     │ (from similar   │  │         │  │  publishers like  │
     │  advertisers)   │  │         │  │  to promote)     │
     └────────┬────────┘  └────┬────┘  └────────┬────────┘
              │                │                 │
              └────────────────┼─────────────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Brief Generator     │
                    │  (Claude/GPT)        │
                    │                      │
                    │  System prompt:      │
                    │  - Brand voice guide │
                    │  - Successful brief  │
                    │    examples          │
                    │  - Publisher prefs   │
                    │  - CPC/CPM context  │
                    └──────────┬───────────┘
                               │
                    ┌──────────▼───────────┐
                    │  Generated Brief     │
                    │  + Recommended CPC   │
                    │  + Publisher Matches  │
                    └──────────────────────┘
```

### 3.2 The Three Context Layers

The LLM needs three layers of context to generate a good brief:

**Layer 1: Brand Intelligence (from the advertiser)**

Extracted by LLM from the advertiser's website, social media, and intake form:

```python
brand_context = {
    "brand_voice": "playful, Gen-Z, casual but informed",  
    "key_usps": ["clean ingredients", "affordable luxury", "cruelty-free"],
    "content_themes": ["skincare routines", "ingredient education", "self-care"],
    "tone_examples": [
        "We believe great skin shouldn't cost a fortune",
        "Your skin barrier called. It wants better ingredients."
    ],
    "target_audience": "Women 22-32, health-conscious, mid-range budget",
    "product_focus": "New vitamin C serum launch",
    "campaign_goal": "awareness + trial"
}
```

**Layer 2: Graph Neighbour Intelligence (from similar advertisers)**

```cypher
// Find successful briefs from similar advertisers
MATCH (new_adv:Advertiser {id: $new_advertiser_id})
MATCH (similar:Advertiser)-[:SIMILAR_BRAND {similarity: sim}]->(new_adv)
WHERE sim > 0.6
MATCH (similar)-[:CREATED_CAMPAIGN]->(c:Campaign)-[:HAS_PLACEMENT]->(pl:Placement)
WHERE pl.ctr > 0.02  // above-average campaigns only
RETURN c.brief_text, c.target_topics, c.pricing_model, c.target_cpc,
       pl.type, pl.actual_cpc, pl.ctr,
       similar.industry, similar.name
ORDER BY pl.ctr DESC
LIMIT 10
```

This gives us: "Here are 10 campaign briefs from similar brands that publishers picked up AND that performed well."

**Layer 3: Publisher Preference Intelligence (from graph)**

```cypher
// What do target publishers like to promote?
MATCH (p:Publisher)-[spec:SPECIALISES_IN]->(t:Topic)
WHERE t.name IN $target_topics
AND spec.avg_engagement > 0.5
MATCH (p)<-[:ON_PUBLISHER]-(pl:Placement)<-[:HAS_PLACEMENT]-(c:Campaign)
WHERE pl.status = 'completed'
RETURN t.name AS topic,
       collect(DISTINCT c.brief_text)[0..3] AS briefs_publishers_accepted,
       avg(pl.actual_cpc) AS avg_cpc_publishers_earned,
       collect(DISTINCT pl.type) AS placement_types_that_worked
```

This gives us: "Publishers in your target topics tend to accept briefs about X, delivered at Y CPC, in Z placement format."

### 3.3 LLM Prompt Template

```python
BRIEF_GENERATION_PROMPT = """
You are writing a campaign brief for {brand_name} on the Linkby marketplace.
The brief needs to appeal to publishers so they pick up the campaign.

## Brand Voice
{brand_voice_summary}
Key selling points: {key_usps}
Tone examples from their site: {tone_examples}

## Campaign Details
Product: {product_focus}
Goal: {campaign_goal}  
Target audience: {target_audience}
Budget: {budget_range}
Recommended CPC: ${recommended_cpc} (based on similar campaigns)

## What Works — Successful Briefs from Similar Brands
These briefs from brands in the same space had high publisher pickup AND performance:
{top_performing_briefs}

## What Publishers Want
Publishers in the target verticals tend to accept campaigns that:
- Focus on these topics: {publisher_preferred_topics}
- Use this placement format: {preferred_placement_type}
- Offer CPC in the range: ${cpc_range_low} - ${cpc_range_high}

## Instructions
Write a campaign brief (150-250 words) that:
1. Matches the brand's voice and tone (use their style, not generic marketing speak)
2. Clearly communicates the value proposition for the publisher's audience
3. Frames the product in a way that aligns with what publishers in this space promote
4. Includes a suggested article angle that a publisher could realistically write
5. Feels like a collaboration opportunity, not an ad placement request

Do NOT write generic marketing copy. Make it sound like {brand_name} wrote it themselves.
"""
```

### 3.4 Brief Quality Scoring

Before showing the generated brief to the advertiser, score it:

```python
def score_brief(brief_text, brand_context, graph_context):
    scores = {
        # Does it match the brand's voice? (LLM evaluation)
        "voice_alignment": llm_evaluate_voice(brief_text, brand_context["tone_examples"]),
        
        # Does it contain topic keywords publishers care about?
        "publisher_appeal": topic_overlap_score(brief_text, graph_context["publisher_preferred_topics"]),
        
        # Is the CPC/CPM in the competitive range?
        "pricing_competitiveness": pricing_score(recommended_cpc, graph_context["cpc_range"]),
        
        # Embedding similarity to successful briefs
        "similarity_to_winners": cosine_sim(embed(brief_text), graph_context["successful_brief_embeddings"]),
    }
    
    scores["composite"] = (
        scores["voice_alignment"] * 0.30 +
        scores["publisher_appeal"] * 0.25 +
        scores["pricing_competitiveness"] * 0.20 +
        scores["similarity_to_winners"] * 0.25
    )
    return scores
```

If composite score < threshold, regenerate with adjusted parameters.

---

## 4. CPC/CPM Recommendation Engine

### 4.1 The Pricing Tension

```
Too low:  Publishers ignore the campaign → advertiser gets no traction → churns
Too high: Advertiser overspends → gets results but feels ripped off → churns
Sweet spot: Attractive to publishers AND efficient for advertiser
```

### 4.2 Three-Signal Pricing Model

The recommended CPC/CPM comes from three signals, blended:

**Signal 1: Graph Neighbour Pricing (strongest signal)**

What did similar advertisers pay for successful campaigns?

```cypher
MATCH (new_adv:Advertiser {id: $new_advertiser_id})
MATCH (similar:Advertiser)-[:SIMILAR_BRAND]->(new_adv)
MATCH (similar)-[:CREATED_CAMPAIGN]->(c:Campaign)-[:HAS_PLACEMENT]->(pl:Placement)
WHERE pl.status = 'completed'
AND c.pricing_model = $pricing_model  // "cpc" or "cpm"

// Separate "successful" vs "all" campaigns
WITH pl,
     CASE WHEN pl.ctr > 0.015 THEN 'high_perform' ELSE 'standard' END AS tier

RETURN tier,
       percentileDisc(pl.actual_cpc, 0.25) AS p25_cpc,
       percentileDisc(pl.actual_cpc, 0.50) AS median_cpc,
       percentileDisc(pl.actual_cpc, 0.75) AS p75_cpc,
       count(pl) AS sample_size
```

**Signal 2: Publisher Supply/Demand (market signal)**

How competitive is the current marketplace for these target topics?

```python
def supply_demand_adjustment(target_topics, target_geo):
    """
    If many campaigns are competing for the same publishers,
    CPC needs to be higher. If publishers are underutilised, lower.
    """
    # Active campaigns targeting these topics
    active_demand = count_active_campaigns(target_topics)
    
    # Available publishers in these topics
    available_supply = count_active_publishers(target_topics, target_geo)
    
    # Ratio
    demand_ratio = active_demand / max(available_supply, 1)
    
    # High demand (>2 campaigns per publisher) → nudge CPC up 10-20%
    # Low demand (<0.5) → can suggest lower CPC
    if demand_ratio > 2.0:
        return 1.15  # +15%
    elif demand_ratio > 1.0:
        return 1.05  # +5%
    elif demand_ratio < 0.5:
        return 0.90  # -10%
    else:
        return 1.0
```

**Signal 3: Existing CPC Prediction Model (our model)**

We already have a regression model that predicts CPC. Use it as a third signal.

```python
model_predicted_cpc = cpc_prediction_model.predict(campaign_features)
```

**Blended Recommendation:**

```python
def recommend_cpc(new_advertiser_id, campaign_features, target_topics, target_geo):
    # Signal 1: Graph neighbours (weight: 0.50)
    neighbour_pricing = get_neighbour_pricing(new_advertiser_id)
    graph_cpc = neighbour_pricing["median_cpc"]
    
    # Signal 2: Supply/demand (weight: 0.20)
    sd_multiplier = supply_demand_adjustment(target_topics, target_geo)
    
    # Signal 3: Existing model (weight: 0.30)
    model_cpc = cpc_prediction_model.predict(campaign_features)
    
    # Blend
    base_cpc = (graph_cpc * 0.50) + (model_cpc * 0.30) + (graph_cpc * sd_multiplier * 0.20)
    
    # Generate range (not just a point estimate)
    return {
        "recommended_cpc": round(base_cpc, 2),
        "range_low": round(base_cpc * 0.85, 2),   # budget-conscious
        "range_high": round(base_cpc * 1.20, 2),   # aggressive pickup
        "confidence": calculate_confidence(neighbour_pricing["sample_size"]),
        "reasoning": generate_pricing_explanation(
            graph_cpc, model_cpc, sd_multiplier, 
            neighbour_pricing["sample_size"]
        )
    }
```

### 4.3 Presenting the Recommendation

Don't just show a number. Show reasoning:

> **Recommended CPC: $0.45** (range: $0.38 – $0.54)
> 
> Based on: 23 successful campaigns from similar brands in the beauty vertical.
> - Brands like yours typically pay $0.40 – $0.52 CPC
> - Publisher demand in beauty is currently moderate (1.3 campaigns per publisher)
> - At $0.45, your campaign is competitive with 68% of active beauty campaigns
>
> 💡 **Tip:** Setting CPC at $0.48+ will put you in the top 25% of beauty campaigns, increasing publisher pickup speed.

### 4.4 Progressive Learning

As the advertiser runs their first campaign, update their profile:

```python
def after_first_campaign(advertiser_id, campaign_results):
    """
    After the cold-start campaign completes, we now have REAL data.
    Transition from graph-neighbour priors to direct performance data.
    """
    # 1. Update advertiser node with actual performance
    update_advertiser_metrics(advertiser_id, campaign_results)
    
    # 2. Create PERFORMED_FOR edges from real data
    create_performance_edges(advertiser_id, campaign_results)
    
    # 3. Adjust SIMILAR_BRAND edges based on actual behavior
    # (they may be more/less similar to neighbours than we thought)
    recalculate_similarity(advertiser_id)
    
    # 4. Next campaign recommendation uses 70% direct data, 30% graph neighbours
    # (vs 100% graph neighbours for the first campaign)
```

Weight transition over time:

| Campaign # | Graph Neighbour Weight | Direct History Weight |
|-----------|----------------------|----------------------|
| 1 (cold start) | 100% | 0% |
| 2 | 60% | 40% |
| 3 | 40% | 60% |
| 5+ | 15% | 85% |
| 10+ | 5% | 95% |

---

## 5. Cold Start for Publishers

### 5.1 Content-Based Warm Start

For new publishers, we can't wait for campaign history. Instead:

```python
def onboard_new_publisher(publisher_intake):
    # 1. Crawl their content (3-5 recent articles)
    articles = crawl_publisher_content(publisher_intake["domain"])
    
    # 2. LLM extraction: topics, tone, audience signals
    content_analysis = llm_extract_publisher_profile(articles)
    # Returns: topics covered, writing style, audience indicators,
    #          content quality score, ad-friendliness score
    
    # 3. Generate embedding from content
    publisher_embedding = embed(content_analysis["summary"])
    
    # 4. Find K nearest publishers in the graph
    neighbours = find_similar_publishers(publisher_embedding, k=10)
    
    # 5. Transfer priors
    priors = {
        "expected_cpc_range": aggregate_cpc_range(neighbours),
        "best_advertiser_verticals": aggregate_top_verticals(neighbours),
        "typical_ctr": aggregate_ctr(neighbours),
        "recommended_campaigns": find_matching_active_campaigns(
            content_analysis["topics"], 
            neighbours
        )
    }
    
    return priors
```

### 5.2 Instant Campaign Recommendations

When a new publisher onboards, immediately show them:

> **Welcome! Based on your content, here are 5 campaigns that match:**
> 
> 1. **Glossier Summer Launch** — Beauty / Skincare — $0.45 CPC
>    *Match: your content about clean beauty + ingredient education aligns with their brief*
> 
> 2. **Ritual Vitamins Q2** — Health / Wellness — $0.52 CPC  
>    *Match: publishers like you (The Urban List, Broadsheet) have delivered 1.8x CTR for Ritual*
>
> Publishers similar to you earn an average of **$0.41 CPC** in the beauty vertical.

### 5.3 Setting CPC/CPM Expectations

```python
def publisher_cpc_expectations(new_publisher_id):
    neighbours = get_publisher_neighbours(new_publisher_id)
    
    return {
        "vertical_avg_cpc": avg([n.avg_cpc_delivered for n in neighbours]),
        "vertical_cpc_range": (
            percentile([n.avg_cpc_delivered for n in neighbours], 25),
            percentile([n.avg_cpc_delivered for n in neighbours], 75),
        ),
        "top_earning_topics": get_highest_cpc_topics(neighbours),
        "message": f"Publishers in your vertical typically earn ${low}-${high} CPC. "
                   f"Top performers earn ${top_quartile}+ by specialising in "
                   f"{top_earning_topics}."
    }
```

---

## 6. The Flywheel

Cold start → warm start → hot start:

```
New advertiser/publisher joins
        │
        ▼
Graph neighbours provide "warm priors"
(CPC range, brief style, publisher matches)
        │
        ▼
First campaign runs with graph-guided defaults
        │
        ▼
Real performance data flows back into graph
        │
        ▼
Next recommendation is better (direct data + neighbours)
        │
        ▼
Every new entity that joins benefits from
ALL previous entities' accumulated intelligence
        │
        ▼
Graph gets smarter with every transaction
(compounding marketplace intelligence)
```

**This is the moat.** A new competitor starting from zero has no graph neighbours to bootstrap from. We have every campaign ever run on Linkby, connected into a reasoning layer that makes every new advertiser's first campaign better than it would be anywhere else.

---

## 7. Implementation Priority

| Priority | Feature | Impact | Effort |
|----------|---------|--------|--------|
| **P0** | CPC/CPM recommendation from graph neighbours | Prevents overspend AND increases pickup rate | Medium (build on existing CPC model) |
| **P1** | LLM brief generation with graph context | Dramatically better first briefs → higher publisher acceptance | Medium (LLM pipeline + graph queries) |
| **P2** | New publisher instant campaign matching | Reduces time-to-first-campaign for publishers | Low (graph query, UI work) |
| **P3** | Brief quality scoring | Catch bad briefs before they go out | Low (LLM evaluation) |
| **P4** | Progressive weight transition (neighbour → direct data) | Smooth handoff from cold to warm | Low (config change) |

### Recommended Starting Point

Build P0 first. The CPC recommendation engine:
- Uses data we already have (campaign history, existing CPC model)
- Measurable impact (compare recommended CPC vs actual pickup rate)
- Quick win that demonstrates graph value to the team
- Natural foundation for P1 (brief generation needs the same graph neighbour queries)

---

## 8. Key Cypher Queries for Cold Start

### Find Graph Neighbours for New Advertiser

```cypher
// Using LLM industry grouping + embedding similarity
MATCH (new:Advertiser {id: $new_id})-[:IN_INDUSTRY]->(t:Topic)
MATCH (existing:Advertiser)-[:IN_INDUSTRY]->(t)
WHERE existing.id <> new.id
AND existing.total_campaigns > 2  // enough history to learn from

// Score by: shared topics + campaign count (more data = better neighbour)
WITH existing, count(t) AS shared_topics, existing.total_campaigns AS campaigns
ORDER BY shared_topics DESC, campaigns DESC
LIMIT 15

// Pull their campaign performance
MATCH (existing)-[:CREATED_CAMPAIGN]->(c:Campaign)-[:HAS_PLACEMENT]->(pl:Placement)
WHERE pl.status = 'completed'
RETURN existing.name, existing.industry,
       shared_topics,
       avg(pl.actual_cpc) AS avg_cpc,
       avg(pl.ctr) AS avg_ctr,
       collect(DISTINCT c.brief_text)[0..3] AS sample_briefs,
       collect(DISTINCT pl.type) AS placement_types
```

### Find Graph Neighbours for New Publisher

```cypher
// Using vertical + channel type matching
MATCH (new:Publisher {id: $new_id})
MATCH (existing:Publisher)
WHERE existing.id <> new.id
AND existing.primary_vertical = new.primary_vertical
AND existing.channel_type = new.channel_type  // web matches web, substack matches substack
AND existing.total_campaigns > 3

// Ranked by campaign volume (more history = better neighbour)
WITH existing
ORDER BY existing.total_campaigns DESC
LIMIT 10

// Pull their performance data
MATCH (existing)<-[:ON_PUBLISHER]-(pl:Placement)<-[:HAS_PLACEMENT]-(c:Campaign)<-[:CREATED_CAMPAIGN]-(adv:Advertiser)
WHERE pl.status = 'completed'
RETURN existing.name, existing.primary_vertical,
       avg(pl.actual_cpc) AS avg_cpc,
       avg(pl.ctr) AS avg_ctr,
       existing.fulfillment_rate,
       collect(DISTINCT adv.industry)[0..5] AS advertiser_verticals_that_worked,
       count(pl) AS completed_placements
```
