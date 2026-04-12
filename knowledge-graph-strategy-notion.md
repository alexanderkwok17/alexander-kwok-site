# Knowledge Graphs for Linkby: Scaling the Marketplace with Graph Intelligence

> **Author:** Alexander Kwok | **Date:** April 2026 | **Status:** Internal Draft
> **TL;DR:** As we expand from web publishers to Substack, email lists, and influencers, our current matching approach won't scale. A knowledge graph built on our existing campaign data, ML models, and publisher performance history can automate marketplace operations, fix the reverse pitch engagement problem, and turn every campaign into compounding intelligence for the next one.

---

## 1. The Problem We're Solving

Linkby's marketplace works: advertisers set campaign details (CPC/CPM, budget, targeting), publishers pick up campaigns, and we deliver via listicle or pubfeed placements. We track impressions and clicks, and our existing ML models (clustering, CPC prediction, publisher recommendations) already add intelligence.

But three things are about to break:

### 1.1 Scale

We're expanding publisher supply to Substack newsletters, email lists, and influencers. That's an order-of-magnitude increase in publisher inventory. The current process of advertisers browsing publishers or our team making manual recommendations won't scale. We need automated, high-precision matching that works across web publishers, newsletters, and influencer channels without human curation for every campaign.

### 1.2 Reverse Pitch Engagement

Publishers can pitch campaigns to advertisers, but advertisers are ignoring them. The core issue isn't pitch quality — it's signal-to-noise. Advertisers see a stream of pitches from publishers they don't know, for reasons they can't evaluate quickly. The fix isn't better pitches. It's surfacing only the pitches that have a high probability of being relevant, with clear reasoning for WHY this publisher is a good match.

### 1.3 Cross-Sector Discovery

Our advantage is having publishers across different sectors — lifestyle, finance, tech, health, etc. But advertisers tend to stick with publishers in their own vertical. The non-obvious matches (a fintech brand performing well on a lifestyle publisher because their audiences overlap on "young professionals") are invisible in a keyword/category system.

---

## 2. Why a Knowledge Graph (Not Just Better Models)

We already have good ML infrastructure — LLM-based industry grouping, clustering algorithms, XGBoost-based CPC prediction and publisher recommendation (in production), and headline/description recommendation models (in development). So why add a knowledge graph?

**Because our current models answer "what's similar?" but can't answer "what's connected and why?"**

| Question | Current ML | Knowledge Graph |
|----------|-----------|-----------------|
| "Which publishers are in the beauty vertical?" | ✅ Clustering handles this | ✅ Also handles this |
| "Which beauty publishers have audiences that also read finance content?" | ❌ Requires multi-hop reasoning | ✅ Graph traversal |
| "For this campaign brief, which publishers have historically delivered strong CPC performance for similar briefs AND reach the target demographic?" | ❌ Separate models, not connected | ✅ Single query across relationships |
| "Why should this advertiser care about this publisher's pitch?" | ❌ Can't explain | ✅ Explainable path through the graph |
| "What changed between Q4 and Q1 for this publisher's performance?" | ❌ Point-in-time predictions | ✅ Temporal edges track changes |

The knowledge graph doesn't replace our existing models. It **connects them** into a unified reasoning layer. Our CPC predictions become properties on edges. Our clustering becomes community detection in the graph. Our publisher recommendations become graph traversals with explainability.

---

## 3. Three High-Impact Applications

### 3.1 Automated Campaign-to-Publisher Matching (Scale Problem)

**Current state:** Advertisers browse publishers or get manual recommendations. Works at current scale, breaks at 10x publisher inventory.

**With knowledge graph:** When an advertiser creates a campaign, the graph automatically:

1. Parses the brief → extracts target topics and audience segments (we already have LLM industry grouping — extend this)
2. Traverses the graph: `Campaign → targets → Topics ← specialises_in ← Publishers` 
3. Filters by: audience reach, historical CPC performance for similar campaigns, publisher fulfillment rate
4. Ranks by composite score (topic relevance × audience fit × performance history)
5. Returns top 10-15 publishers with **reasoning** ("matched because: covers 3 of your target topics, reaches 28% of your target demo, delivered 1.4x above-benchmark CPC for similar campaigns last quarter")

**Works across all publisher types** — whether it's a web publisher, Substack newsletter, or influencer, the graph treats them as the same entity type with different channel properties. No separate matching logic per channel.

**Feeds existing models:** Our CPC prediction model's output becomes a property on the `PLACED_ON` relationship. Our clustering model's output defines `SIMILAR_AUDIENCE` edges. The graph is the connective tissue.

### 3.2 Fixing Reverse Pitch (Engagement Problem)

**Current state:** Publishers choose which advertisers to pitch. Advertisers see a stream of pitches they didn't ask for. Low engagement because low signal-to-noise.

**The insight:** Advertisers ignore pitches because they can't quickly evaluate relevance. The fix is two-fold:

**A. Publisher-side: Guide who to pitch**

Instead of publishers choosing blindly, the graph recommends: "Based on your content, audience, and performance history, these 5 advertisers have active campaigns that are a strong match. Here's why."

Query: `Publisher → specialises_in → Topics ← targets ← active Campaigns ← runs ← Advertisers`
Filter by: audience overlap, budget alignment, no existing placement

This turns reverse pitch from "spray and pray" to "graph-guided targeting."

**B. Advertiser-side: Surface only high-relevance pitches**

Instead of showing all pitches, show only pitches that score above a relevance threshold, with a one-line explanation:

> "The Urban List pitched your Summer Campaign. **Match score: 87%.** They reach 32% of your target demo, their beauty content gets 2.1x above-average engagement, and similar brands saw $0.42 avg CPC on their platform (your target: $0.50)."

Advertisers are much more likely to engage with a pitch that comes with data-backed reasoning than a cold outreach.

**Measuring impact:** Track pitch acceptance rate before/after. Current baseline → graph-guided rate. This is a clean A/B test.

### 3.3 Cross-Sector Discovery (Hidden Value Problem)

**Current state:** Advertisers stick to their vertical. A skincare brand targets beauty publishers. They miss that a travel publisher's audience over-indexes on skincare purchases.

**With knowledge graph:** The `SIMILAR_AUDIENCE` and `COMPLEMENTARY_REACH` edges (computed from our existing data) reveal non-obvious connections:

```
// Find publishers outside the advertiser's primary vertical
// whose audiences overlap with publishers that have performed well
MATCH (a:Advertiser)-[:RUNS]->(c:Campaign)-[:PLACED_ON]->(p1:Publisher)
WHERE c.status = 'completed' AND c.performance_rating > 0.7
MATCH (p1)-[:SIMILAR_AUDIENCE {overlap: score}]->(p2:Publisher)
WHERE p2.primary_vertical <> p1.primary_vertical
AND score > 0.25
AND NOT (a)-[:RUNS]->()-[:PLACED_ON]->(p2)  // never worked together
RETURN p2.name, p2.primary_vertical, score AS audience_overlap,
       collect(DISTINCT p1.name) AS similar_to_your_successful_publishers
```

This surfaces: "You've never worked with TechCrunch, but their audience overlaps 31% with The Urban List, where your campaigns perform 1.8x above benchmark."

---

## 4. What We Already Have (and How It Maps)

The good news: we have most of the data we need. The graph doesn't require new data collection — it requires connecting what we already have.

| Existing Asset | Maps To | Graph Role |
|---------------|---------|------------|
| Campaign history (CPC/CPM, budget, impressions, clicks) | `:Campaign` nodes + `:PerformanceMetric` nodes | Core marketplace relationships |
| Publisher profiles, verticals | `:Publisher` nodes | Central entity |
| Advertiser briefs, targeting | `:Advertiser` nodes + `:Campaign` nodes | Demand-side entities |
| Reverse pitch history (who pitched whom, accepted/rejected) | `:Pitch` nodes with `PITCHED_TO` and `PITCHED_BY` edges | Training data for pitch scoring |
| Impression/click tracking (IP-based) | Properties on `:PerformanceMetric` | Performance edges |
| Advertiser spend data | Property on `:Advertiser` + `:Campaign` | Budget/value signals |
| Publisher fulfillment performance | Property on `:Publisher` + `PLACED_ON` edges | Reliability scoring |
| LLM-based industry grouping | `:Topic` nodes + `SPECIALISES_IN` / `TARGETS` edges | Semantic taxonomy |
| Clustering algorithm outputs | `SIMILAR_AUDIENCE` / `COMPLEMENTARY_REACH` edges | Discovery relationships |
| XGBoost CPC prediction + publisher recommendation (production) | Property on `PLACED_ON` edges (predicted_cpc), publisher ranking signal | Core matching signal — graph wraps around these outputs |
| Headline/description recommendation model (in development) | Graph provides context to improve recommendations | Graph-neighbour successful headlines/descriptions become training signal |

**What we're missing (and it's fine):**

- **Conversion data** — we only track widget loads, not full-funnel conversions. The graph works without this. When we add conversion tracking later, it becomes a new edge type that immediately enriches every query.
- **Article-level content embeddings** — we have publisher-level data but not article-level semantic analysis. Phase 2 adds this with LLM extraction from published content.

---

## 5. Scaling to New Channels

The knowledge graph architecture handles Substack, email lists, and influencers without separate matching logic.

### Publisher Entity Extends Naturally

```
(:Publisher {
  id:              String,
  name:            String,
  channel_type:    String,    // "web" | "substack" | "email_list" | "influencer"
  platform:        String,    // "wordpress" | "substack" | "mailchimp" | "instagram" | "tiktok"
  
  // Channel-specific metrics (nullable)
  monthly_uniques: Integer,   // web publishers
  subscriber_count: Integer,  // substack / email
  follower_count:  Integer,   // influencers
  open_rate:       Float,     // email lists
  engagement_rate: Float,     // influencers
  
  // Universal properties
  primary_vertical: String,
  geo_focus:       String[],
  fulfillment_rate: Float,    // how reliably they deliver
  avg_cpc_delivered: Float,   // historical performance
})
```

### Same Relationships, Different Channels

A Substack newsletter and a web publisher both:
- `SPECIALISES_IN` topics
- `REACHES` audience segments
- Get `PLACED_ON` by campaigns
- Have `SIMILAR_AUDIENCE` edges to other publishers

The graph doesn't care about the channel. It cares about: does this publisher's audience match this advertiser's target? Has this publisher delivered for similar campaigns? The matching query is identical regardless of whether the publisher is a website, newsletter, or influencer.

### Placement Types as Edge Properties

```
(:Campaign)-[:PLACED_ON {
  placement_type:   String,   // "listicle" | "pubfeed" | "newsletter_feature" | "influencer_post"
  negotiated_cpc:   Float,
  negotiated_cpm:   Float,
  budget_allocated: Float,
  status:           String,
}]->(:Publisher)
```

---

## 6. Implementation Roadmap

### Phase 1: Graph Foundation (4-6 weeks)

**Goal:** Load existing data into Neo4j, validate the schema, prove the queries work.

- Export campaign history, publisher profiles, advertiser data, reverse pitch history into graph
- Map existing LLM industry groupings to `:Topic` nodes
- Map clustering outputs to `SIMILAR_AUDIENCE` edges
- Load CPC predictions as properties on `PLACED_ON` edges
- Test the 3 core queries: campaign matching, pitch scoring, cross-sector discovery
- **Success metric:** Graph-based publisher recommendations match or beat current model recommendations on historical campaign data

### Phase 2: Reverse Pitch Intelligence (4 weeks)

**Goal:** Improve reverse pitch acceptance rate.

- Build pitch scoring model using graph features (topic overlap + audience match + performance history)
- Add publisher-facing recommendations: "pitch these advertisers"
- Add advertiser-facing filtering: surface only pitches above relevance threshold with reasoning
- **Success metric:** Pitch acceptance rate improvement (current baseline → graph-guided)

### Phase 3: Automated Matching at Scale (6 weeks)

**Goal:** Handle 10x publisher inventory without manual curation.

- Integrate graph matching into campaign creation flow
- Auto-recommend publishers when advertiser creates campaign
- Handle Substack/email/influencer publishers in same matching pipeline
- Add brief embedding + semantic search for hybrid retrieval
- **Success metric:** Time from campaign creation to first publisher match < 5 minutes (automated)

### Phase 4: Compounding Intelligence (Ongoing)

**Goal:** Every campaign makes the next one smarter.

- Temporal edges: track how publisher performance changes over time
- Feedback loops: campaign outcomes update relationship weights
- Seasonal pattern detection: "this publisher's audience engagement spikes in Q4"
- Agent layer: MCP-connected agents that can query the graph to auto-generate campaign proposals
- **Success metric:** Recommendation accuracy improves month-over-month as more campaign data flows in

---

## 7. Technical Stack

| Layer | Technology | Why |
|-------|-----------|-----|
| **Graph database** | Neo4j AuraDB | Most mature, native vector index, Python driver, free tier for prototyping |
| **Python integration** | LlamaIndex PropertyGraphIndex | Hybrid search (vector + graph), fits our Python stack |
| **Entity extraction** | Claude Sonnet (extend existing LLM grouping) | Already using LLM for industry grouping — extend to topic extraction |
| **Embeddings** | OpenAI text-embedding-3-large | For brief/publisher semantic matching in Phase 3 |
| **Query layer** | Direct Cypher + Text-to-Cypher for internal tools | Cypher for production, natural language for internal exploration |
| **Integration** | Existing CPC prediction model outputs as graph properties | Graph connects existing models, doesn't replace them |

---

## 8. Risks & Mitigations

| Risk | Mitigation |
|------|-----------|
| **Graph quality depends on data quality** | Start with highest-confidence data (campaign history, spend). Add ML-extracted signals in Phase 2. |
| **Cold start for new publishers** | Use content-based matching (topic extraction from their existing content) until performance data accumulates. Substack/email have subscriber counts as a proxy. |
| **No conversion data** | Design the schema to accept conversion data as a future edge type. Use CPC/impression/click data as proxy in the meantime. |
| **Team doesn't know Cypher** | LlamaIndex Text-to-Cypher provides natural language interface. Internal dashboards abstract away query complexity. |
| **Over-engineering risk** | Phase 1 is deliberately minimal — load data, test queries, prove value. Only expand if Phase 1 shows measurable improvement over current recommendations. |

---

## 9. Cold Start & New Advertiser Experience

This is arguably the most important application. Right now, when a new advertiser joins Linkby — or a dormant advertiser returns after 12+ months of no campaign spend — they face a blank slate. No recommended CPC, no suggested budget, no headline guidance, no publisher shortlist. Every decision is guesswork. That friction kills first-campaign launch rate and time-to-value.

The same problem applies to dormant advertisers returning. Their historical data is stale — publisher inventory has changed, CPC benchmarks have shifted, new publishers have joined. Treating them like returning users with outdated priors is almost as bad as treating them like brand new.

### The Problem: Friction to First Campaign

Today, a new advertiser has to make 5+ decisions with zero data:

1. **What CPC/CPM to set** — too low and no publisher picks up; too high and they overspend
2. **What budget to allocate** — no sense of what's typical for their vertical/goal
3. **What headline and description to write** — no insight into what publishers respond to
4. **Which placement type** — listicle vs pubfeed, no guidance
5. **Which publishers to target** — browsing a list with no ranking or reasoning

Each of these is a potential drop-off point. The graph eliminates all five by transferring intelligence from similar advertisers who have already figured it out.

### The Approach: "Warm Start via Graph Neighbours"

A new advertiser has zero history. A dormant advertiser has stale history. But advertisers who look like them have fresh, rich history. The graph transfers that intelligence immediately.

### What the New Advertiser Experience Looks Like (With Graph)

Advertiser signs up, provides: brand name, website, product/objective, target audience.

**The system returns (in seconds, not days):**

**1. Recommended Starting CPC**

Not just a number — a range with reasoning:

> **Recommended CPC: $0.45** (range: $0.38 – $0.54)
> Based on 23 successful campaigns from brands similar to yours in the beauty vertical.
> At $0.45, you're competitive with 68% of active beauty campaigns.
> 💡 Setting CPC at $0.48+ puts you in the top 25%, increasing publisher pickup speed.

The recommendation blends: graph-neighbour median pricing (what similar brands actually paid), current supply/demand (how competitive your target topics are right now), and our XGBoost CPC prediction model output.

**2. Suggested Budget**

> **Suggested budget: $500 – $800 for first campaign**
> Similar brands typically start at $600. At this level, you'll get 3-5 publisher placements with enough impressions to learn what works.

Derived from: graph-neighbour first-campaign budgets, filtered to successful campaigns (those that led to a second campaign).

**3. Headline & Description Recommendations**

This is where the graph directly improves our headline/description recommendation model. Instead of generating copy in a vacuum, the LLM gets three layers of context:

- **Brand voice** — extracted from the advertiser's website (tone, USPs, style)
- **High-performing headlines from graph neighbours** — "these headlines from similar brands had the highest publisher pickup rate and CTR"
- **Publisher preferences** — "publishers in your target topics tend to accept campaigns framed around [angles], not [angles]"

> **Suggested headline:** "Clean Beauty That Actually Works: Your Readers Will Thank You"
> **Why:** Headlines that frame the product as reader-value (not brand-push) see 2.3x higher publisher acceptance in the beauty vertical.

This context loop feeds back into improving the headline/description model itself — successful graph-neighbour headlines become training signal for the model, and model outputs get validated against graph performance data.

**4. Publisher Recommendations with CPC Preferences**

For each recommended publisher, show the CPC range they typically accept:

> | Publisher | Vertical | Their CPC Range | Your CPC | Fit |
> |-----------|----------|----------------|----------|-----|
> | The Urban List | Lifestyle/Beauty | $0.35 – $0.55 | $0.45 | ✅ Competitive |
> | Broadsheet | Lifestyle | $0.40 – $0.60 | $0.45 | ✅ Mid-range |
> | Beauty Crew | Beauty | $0.30 – $0.45 | $0.45 | ⭐ Top of range — fast pickup |
> | Refinery29 AU | Beauty/Culture | $0.50 – $0.75 | $0.45 | ⚠️ Below range — may not pick up |

This transparency removes guesswork and lets the advertiser make an informed CPC decision per publisher. The publisher CPC preference ranges come directly from the graph: historical `actual_cpc` distributions per publisher, filtered to accepted/completed placements.

**5. One-Click Campaign Launch**

With all five decisions pre-filled (CPC, budget, headline, description, publishers), the advertiser can review, adjust, and launch in one session. No back-and-forth, no waiting for manual recommendations.

### Dormant Advertiser Re-Activation

For advertisers returning after 12+ months of no spend, the same graph-neighbour approach applies with one addition: **we show what changed.**

> **Welcome back! Here's what's different since your last campaign:**
> - 12 new publishers joined in your vertical (beauty) — including 3 Substack newsletters
> - Average CPC in beauty shifted from $0.38 to $0.45
> - Your previous publisher (The Urban List) now has 1.4x more traffic
> - New publisher type available: pubfeed placements (products embedded in articles)
>
> **Your recommended starting point** (updated for current market):
> CPC: $0.45 | Budget: $600 | [View recommended publishers →]

The graph treats dormant advertisers as "warm cold start" — they have some historical signal (previous campaigns, preferences), but the graph re-anchors their priors to current market conditions using fresh neighbour data.

### Progressive Learning

The graph smoothly transitions from cold to warm:

| Advertiser State | Graph Neighbour Weight | Direct History Weight |
|-----------------|----------------------|----------------------|
| Brand new | 100% | 0% |
| After 1st campaign | 60% | 40% |
| After 3 campaigns | 30% | 70% |
| Active (5+ campaigns) | 10% | 90% |
| Dormant returning | 50% (re-anchored to current) | 50% (stale history, discounted) |

### For New Publishers

Same approach, publisher side:

1. **Content-based warm start** — crawl recent articles, extract topics, find similar publishers in graph, transfer performance priors
2. **Instant campaign recommendations** — "here are 5 active campaigns that match your content" shown immediately on onboarding
3. **CPC expectations** — "publishers like you earn $0.38-$0.52 CPC in this vertical"
4. **What advertisers look for** — "brands in your target verticals respond best to pitches that emphasise [audience data, engagement metrics, content alignment]"

> 📎 Full cold start architecture (LLM pipeline, pricing model, Cypher queries, Python pseudocode) in separate doc: **linkby-cold-start-architecture.md**

---

## 10. The Pitch to the Team

### Where We Are Now

Our XGBoost-based optimization is already live — it recommends publishers and predicts CPC for campaigns. This is working and we should keep investing in it. The question isn't "replace XGBoost" — it's "what can't XGBoost do that we need as we scale?"

### What XGBoost Does Well

- Predicts CPC given campaign features (budget, vertical, placement type)
- Recommends publishers based on historical performance patterns
- Handles tabular feature relationships efficiently

### Where It Hits a Ceiling

- **Multi-hop reasoning** — "find publishers whose audience overlaps with brands similar to this advertiser AND who cover topics adjacent to the brief" requires chaining relationships across entities. XGBoost sees rows and features, not connected entities.
- **Explainability for users** — XGBoost outputs a score. A graph outputs a path: "we matched you because: 3 shared topics, 28% audience overlap, similar brands averaged $0.42 CPC here." That reasoning is what turns an ignored reverse pitch into an accepted one.
- **Cold start** — XGBoost needs training data for a new advertiser. The graph can bootstrap from similar advertisers immediately because similarity is a relationship, not a feature.
- **Cross-channel scaling** — as we add Substack, email, and influencer publishers, the graph treats them as nodes with the same relationship types. XGBoost would need new features and retraining for each channel type.

### The Graph as the Next Layer

The knowledge graph doesn't replace our XGBoost models — it wraps around them. Our CPC predictions become properties on graph edges. Our publisher recommendation scores feed into graph-based ranking. The graph adds the relationship reasoning and explainability layer that tabular models can't provide.

Think of it as: **XGBoost answers "what's the best prediction given these features." The graph answers "what's connected and why."** We need both.

### Four Problems the Graph Solves

1. **Scale** — as we 10x publisher inventory with Substack/email/influencer, we can't manually curate matches or retrain models per channel. The graph automates matching across all channel types with the same query logic.
2. **Reverse pitch** — advertisers ignore pitches because they can't evaluate relevance. The graph scores relevance AND explains why, so only high-signal pitches surface.
3. **Discovery** — the best advertiser-publisher matches are often cross-sector. The graph finds connections that category-based matching misses.
4. **Cold start** — new advertisers need a great first campaign. The graph bootstraps their experience from similar advertisers' history, feeding our existing CPC model with warm priors instead of zero data.

### How We Measure Success

Phase 1 success metric: do graph-enhanced recommendations improve on our current XGBoost recommendations when backtested on historical campaign data? Specifically:

- **Publisher match acceptance rate** — does graph matching + explainability increase the rate at which advertisers accept recommended publishers?
- **Reverse pitch engagement** — does graph-scored pitch filtering increase advertiser response rate?
- **Cold start first-campaign CPC efficiency** — for new advertisers, does the graph-neighbour CPC recommendation reduce overspend vs the XGBoost default?

If yes, the graph becomes the connective layer for all our ML models. If no, we've learned something about our data relationships that improves our existing models anyway.

Low downside. High upside. Let's prototype.
