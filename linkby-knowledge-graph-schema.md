# Linkby Knowledge Graph Schema v2

> **Author:** Alexander Kwok | **Date:** April 2026 | **Status:** Architecture Draft
> **Purpose:** Graph schema designed around Linkby's actual marketplace: CPC/CPM campaigns, listicle/pubfeed placements, impression/click tracking, reverse pitch, and expansion to Substack/email/influencer channels.

---

## 1. Entity Types (Nodes)

### Publisher

Central entity. Covers web publishers, Substack, email lists, influencers.

```
(:Publisher {
  id:                  String,     // Linkby internal ID
  name:                String,     // "The Urban List"
  domain:              String,     // "theurbanlist.com" (nullable for influencers)
  channel_type:        String,     // "web" | "substack" | "email_list" | "influencer"
  platform:            String,     // "wordpress" | "substack" | "mailchimp" | "instagram" | "tiktok"
  
  // Audience metrics (channel-appropriate)
  monthly_uniques:     Integer,    // web
  subscriber_count:    Integer,    // substack / email
  follower_count:      Integer,    // influencer
  open_rate:           Float,      // email
  engagement_rate:     Float,      // influencer
  
  // Linkby performance
  primary_vertical:    String,     // from LLM industry grouping
  secondary_verticals: String[],   // additional verticals from clustering
  geo_focus:           String[],   // ["AU", "NZ"]
  fulfillment_rate:    Float,      // % of accepted campaigns delivered on time
  avg_cpc_delivered:   Float,      // historical average CPC across all campaigns
  avg_cpm_delivered:   Float,      // historical average CPM
  total_campaigns:     Integer,    // lifetime campaigns completed
  total_revenue:       Float,      // lifetime revenue earned on Linkby
  status:              String,     // "active" | "paused" | "churned"
  onboarded_date:      Date,
  
  // ML signals (from existing models)
  cluster_id:          String,     // from clustering algorithm
  embedding:           Float[],    // semantic embedding (Phase 2)
})
```

### Advertiser

```
(:Advertiser {
  id:                  String,
  name:                String,     // "Glossier"
  industry:            String,     // from LLM industry grouping
  sub_category:        String,     // finer grouping
  brand_tier:          String,     // "enterprise" | "growth" | "startup"
  
  // Spend & behavior
  total_spend:         Float,      // lifetime spend on Linkby
  avg_campaign_budget: Float,
  campaigns_created:   Integer,
  avg_cpc_target:      Float,      // typical CPC they set
  avg_cpm_target:      Float,      // typical CPM they set
  preferred_placement: String,     // "listicle" | "pubfeed" | "both"
  
  // Targeting
  target_demographics: String[],   // ["F25-34", "M18-24"]
  geo_targets:         String[],   // ["AU", "US"]
  
  // Engagement
  pitch_response_rate: Float,      // % of reverse pitches they've responded to
  avg_time_to_respond: Float,      // hours
  status:              String,
  onboarded_date:      Date,
  
  // ML signals
  cluster_id:          String,
  embedding:           Float[],    // Phase 2
})
```

### Campaign

```
(:Campaign {
  id:                  String,
  name:                String,
  status:              String,     // "draft" | "active" | "completed" | "cancelled"
  
  // Pricing
  pricing_model:       String,     // "cpc" | "cpm"
  target_cpc:          Float,      // advertiser's target CPC (nullable)
  target_cpm:          Float,      // advertiser's target CPM (nullable)
  budget:              Float,      // total campaign budget
  budget_remaining:    Float,
  
  // Details
  brief_text:          String,     // campaign brief / description
  target_topics:       String[],   // extracted from brief
  placement_types:     String[],   // ["listicle", "pubfeed"]
  start_date:          Date,
  end_date:            Date,
  
  // Aggregate performance
  total_impressions:   Integer,
  total_clicks:        Integer,
  total_spend:         Float,
  overall_ctr:         Float,
  
  // ML signals
  predicted_cpc:       Float,      // from CPC prediction model
  brief_embedding:     Float[],    // Phase 2
})
```

### Placement

A specific campaign-publisher execution. One campaign can have multiple placements across different publishers.

```
(:Placement {
  id:                  String,
  type:                String,     // "listicle" | "pubfeed" | "newsletter_feature" | "influencer_post"
  status:              String,     // "pending" | "live" | "completed" | "rejected"
  
  // Performance (from widget/IP tracking)
  impressions:         Integer,
  clicks:              Integer,
  ctr:                 Float,
  actual_cpc:          Float,      // realized CPC
  actual_cpm:          Float,      // realized CPM
  spend:               Float,      // amount spent on this placement
  
  // Timing
  created_date:        Date,
  live_date:           Date,
  end_date:            Date,
  
  // Content
  article_url:         String,     // the published article/content URL
  article_title:       String,
})
```

### Pitch (Reverse Pitch)

```
(:Pitch {
  id:                  String,
  pitch_text:          String,     // publisher's pitch content
  status:              String,     // "sent" | "viewed" | "accepted" | "rejected" | "ignored"
  
  created_date:        Date,
  viewed_date:         Date,       // nullable — when advertiser first saw it
  responded_date:      Date,       // nullable
  response_time_hours: Float,      // nullable
  
  // Graph-computed (Phase 2)
  match_score:         Float,      // relevance score from graph
  match_reasons:       String[],   // ["topic_overlap", "audience_fit", "performance_history"]
})
```

### Topic

Taxonomy node — derived from LLM industry grouping + content extraction.

```
(:Topic {
  id:                  String,
  name:                String,     // "sustainable fashion"
  level:               Integer,    // 0=sector, 1=category, 2=subcategory
  source:              String,     // "llm_grouping" | "content_extraction" | "manual"
  embedding:           Float[],
})
```

### AudienceSegment

```
(:AudienceSegment {
  id:                  String,
  name:                String,     // "AU Women 25-34"
  age_range:           String,
  gender:              String,
  geo:                 String,
  interest_tags:       String[],
  estimated_size:      Integer,
})
```

---

## 2. Relationships (Edges)

### Core Marketplace Flow

```
// Advertiser creates campaigns
(:Advertiser)-[:CREATED_CAMPAIGN]->(:Campaign)

// Campaign is placed on publishers
(:Campaign)-[:HAS_PLACEMENT]->(:Placement)
(:Placement)-[:ON_PUBLISHER]->(:Publisher)

// Or: direct edge for simpler queries
(:Campaign)-[:PLACED_ON {
  placement_count: Integer,
  total_spend: Float,
  avg_cpc: Float,
  avg_ctr: Float,
}]->(:Publisher)
```

### Reverse Pitch Flow

```
// Publisher pitches to advertiser for a specific campaign
(:Publisher)-[:PITCHED {date: Date}]->(:Pitch)
(:Pitch)-[:FOR_CAMPAIGN]->(:Campaign)
(:Pitch)-[:TO_ADVERTISER]->(:Advertiser)

// Shortcut for pattern analysis
(:Publisher)-[:PITCHED_TO {
  pitch_count: Integer,
  acceptance_rate: Float,
  avg_response_time: Float,
}]->(:Advertiser)
```

### Topic & Content Relationships

```
// From LLM industry grouping (existing)
(:Publisher)-[:SPECIALISES_IN {
  article_count: Integer,       // how many articles in this topic
  avg_engagement: Float,        // avg CTR on this topic
  confidence: Float,            // LLM grouping confidence
}]->(:Topic)

(:Advertiser)-[:IN_INDUSTRY {confidence: Float}]->(:Topic)

(:Campaign)-[:TARGETS {priority: String}]->(:Topic)
  // priority: "primary" | "secondary"

(:Topic)-[:PARENT_OF]->(:Topic)
  // "Beauty" → "Skincare" → "Clean Beauty"
```

### Audience Relationships

```
(:Publisher)-[:REACHES {
  percentage: Float,
  confidence: Float,
  source: String,              // "analytics" | "estimated" | "self_reported"
}]->(:AudienceSegment)

(:Campaign)-[:TARGETS_AUDIENCE]->(:AudienceSegment)
```

### Discovery Relationships (from existing clustering models)

```
// Publishers with overlapping audiences
(:Publisher)-[:SIMILAR_AUDIENCE {
  overlap_score: Float,        // from clustering algorithm
  computed_date: Date,
}]->(:Publisher)

// Publishers with complementary (non-overlapping) audiences in same vertical
(:Publisher)-[:COMPLEMENTARY_REACH {
  combined_unique_pct: Float,
  shared_vertical: String,
}]->(:Publisher)

// Advertisers with similar profiles
(:Advertiser)-[:SIMILAR_BRAND {
  similarity: Float,           // from clustering
}]->(:Advertiser)
```

### Performance History (Temporal)

```
// Links placement to a time period for temporal queries
(:Placement)-[:IN_PERIOD]->(:TimePeriod {
  period: String,              // "2026-Q1" | "2026-03" | "2026-W14"
  start_date: Date,
  end_date: Date,
})

// Aggregated publisher performance per advertiser vertical
(:Publisher)-[:PERFORMED_FOR {
  campaign_count: Integer,
  avg_cpc: Float,
  avg_ctr: Float,
  avg_fulfillment_time_days: Float,
  period: String,
}]->(:Topic)
  // "This publisher delivered avg $0.42 CPC for beauty campaigns in Q1 2026"
```

---

## 3. Indexes & Constraints

```cypher
// Uniqueness
CREATE CONSTRAINT FOR (p:Publisher) REQUIRE p.id IS UNIQUE;
CREATE CONSTRAINT FOR (a:Advertiser) REQUIRE a.id IS UNIQUE;
CREATE CONSTRAINT FOR (c:Campaign) REQUIRE c.id IS UNIQUE;
CREATE CONSTRAINT FOR (pl:Placement) REQUIRE pl.id IS UNIQUE;
CREATE CONSTRAINT FOR (pi:Pitch) REQUIRE pi.id IS UNIQUE;
CREATE CONSTRAINT FOR (t:Topic) REQUIRE t.id IS UNIQUE;

// Performance lookups
CREATE INDEX FOR (c:Campaign) ON (c.status);
CREATE INDEX FOR (pl:Placement) ON (pl.status);
CREATE INDEX FOR (pi:Pitch) ON (pi.status);
CREATE INDEX FOR (p:Publisher) ON (p.channel_type);
CREATE INDEX FOR (p:Publisher) ON (p.primary_vertical);

// Vector indexes (Phase 2 — hybrid search)
CREATE VECTOR INDEX publisher_embedding FOR (p:Publisher) ON (p.embedding)
  OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarity_function`: 'cosine'}};
CREATE VECTOR INDEX brief_embedding FOR (c:Campaign) ON (c.brief_embedding)
  OPTIONS {indexConfig: {`vector.dimensions`: 1536, `vector.similarity_function`: 'cosine'}};
```

---

## 4. Core Queries

### Q1: Campaign-to-Publisher Auto-Match

"For this active campaign, find the best publishers across ALL channel types."

```cypher
MATCH (c:Campaign {id: $campaign_id})-[:TARGETS]->(t:Topic)
MATCH (p:Publisher)-[spec:SPECIALISES_IN]->(t)
WHERE p.status = 'active'
AND spec.confidence > 0.6

// Check audience fit
OPTIONAL MATCH (c)-[:TARGETS_AUDIENCE]->(seg:AudienceSegment)<-[reach:REACHES]-(p)

// Check historical performance for this topic
OPTIONAL MATCH (p)-[perf:PERFORMED_FOR]->(t)

// Exclude already placed
WHERE NOT (c)-[:HAS_PLACEMENT]->()-[:ON_PUBLISHER]->(p)

RETURN p.name, 
       p.channel_type,
       p.primary_vertical,
       collect(DISTINCT t.name) AS matching_topics,
       avg(spec.avg_engagement) AS topic_engagement,
       avg(reach.percentage) AS audience_reach,
       avg(perf.avg_cpc) AS historical_cpc,
       p.fulfillment_rate,
       // Composite score: topic fit (30%) + audience (25%) + performance (25%) + reliability (20%)
       (coalesce(avg(spec.confidence), 0) * 0.30 +
        coalesce(avg(reach.percentage), 0) * 0.25 +
        coalesce(1.0 / (1.0 + avg(perf.avg_cpc)), 0) * 0.25 +
        coalesce(p.fulfillment_rate, 0) * 0.20) AS match_score
ORDER BY match_score DESC
LIMIT 15
```

### Q2: Reverse Pitch Scoring

"Score an incoming pitch from a publisher to an advertiser."

```cypher
// How well does this publisher match this advertiser?
MATCH (pub:Publisher {id: $publisher_id})
MATCH (adv:Advertiser {id: $advertiser_id})

// Topic overlap
OPTIONAL MATCH (pub)-[ps:SPECIALISES_IN]->(t:Topic)<-[ai:IN_INDUSTRY]-(adv)
WITH pub, adv, collect(t.name) AS shared_topics, count(t) AS topic_overlap

// Historical performance: has this publisher delivered for similar advertisers?
OPTIONAL MATCH (pub)<-[:ON_PUBLISHER]-(:Placement)<-[:HAS_PLACEMENT]-(c:Campaign)<-[:CREATED_CAMPAIGN]-(similar:Advertiser)-[:SIMILAR_BRAND]->(adv)
WITH pub, adv, shared_topics, topic_overlap,
     avg(c.overall_ctr) AS similar_brand_ctr, count(c) AS similar_campaigns

// Past relationship
OPTIONAL MATCH (pub)-[past:PITCHED_TO]->(adv)
WITH pub, adv, shared_topics, topic_overlap, similar_brand_ctr, similar_campaigns,
     past.acceptance_rate AS past_acceptance

// Audience fit
OPTIONAL MATCH (pub)-[:REACHES]->(seg:AudienceSegment)<-[:TARGETS_AUDIENCE]-(c:Campaign)<-[:CREATED_CAMPAIGN]-(adv)

RETURN shared_topics,
       topic_overlap,
       similar_brand_ctr,
       similar_campaigns,
       past_acceptance,
       pub.fulfillment_rate,
       pub.avg_cpc_delivered,
       // Generate match reasons
       CASE WHEN topic_overlap > 2 THEN 'strong_topic_overlap'
            WHEN topic_overlap > 0 THEN 'partial_topic_overlap'
            ELSE 'no_topic_overlap' END AS topic_signal,
       CASE WHEN similar_campaigns > 3 THEN 'proven_with_similar_brands'
            WHEN similar_campaigns > 0 THEN 'some_similar_brand_experience'
            ELSE 'no_similar_brand_data' END AS performance_signal
```

### Q3: Cross-Sector Discovery

"Find publishers outside this advertiser's typical vertical who might perform well."

```cypher
// Find publishers that performed well for similar advertisers
// but are in DIFFERENT verticals than what this advertiser usually targets
MATCH (adv:Advertiser {id: $advertiser_id})-[:IN_INDUSTRY]->(adv_topic:Topic)
MATCH (adv)-[:SIMILAR_BRAND]->(sim_adv:Advertiser)
MATCH (sim_adv)-[:CREATED_CAMPAIGN]->(c:Campaign)-[:PLACED_ON]->(p:Publisher)
WHERE c.overall_ctr > 0.02  // above 2% CTR = good campaign
AND NOT p.primary_vertical IN [adv_topic.name]  // different vertical
AND NOT (adv)-[:CREATED_CAMPAIGN]->()-[:PLACED_ON]->(p)  // never worked with

RETURN p.name, p.primary_vertical, p.channel_type,
       count(c) AS successful_campaigns_for_similar_brands,
       avg(c.overall_ctr) AS avg_ctr,
       collect(DISTINCT sim_adv.name) AS similar_brands_that_worked
ORDER BY successful_campaigns_for_similar_brands DESC
LIMIT 10
```

### Q4: Publisher Pitch Recommendations

"For this publisher, which active campaigns should they pitch?"

```cypher
MATCH (pub:Publisher {id: $publisher_id})-[:SPECIALISES_IN]->(t:Topic)
MATCH (c:Campaign {status: 'active'})-[:TARGETS]->(t)
MATCH (c)<-[:CREATED_CAMPAIGN]-(adv:Advertiser)

// Exclude campaigns already placed on this publisher
WHERE NOT (c)-[:HAS_PLACEMENT]->()-[:ON_PUBLISHER]->(pub)
// Exclude advertisers who blocked this publisher
AND NOT (pub)-[:PITCHED_TO {acceptance_rate: 0}]->(adv)

// Check if publisher's historical CPC is competitive with campaign target
WHERE pub.avg_cpc_delivered <= c.target_cpc * 1.2  // within 20% of target

RETURN c.name, adv.name, c.target_cpc, c.budget,
       collect(DISTINCT t.name) AS matching_topics,
       pub.avg_cpc_delivered AS your_avg_cpc,
       c.target_cpc AS their_target_cpc,
       CASE WHEN pub.avg_cpc_delivered < c.target_cpc THEN 'strong_fit'
            ELSE 'marginal_fit' END AS cpc_alignment
ORDER BY size(collect(DISTINCT t.name)) DESC, pub.avg_cpc_delivered ASC
LIMIT 10
```

### Q5: Marketplace Health — Channel Type Performance

"How are Substack/email/influencer placements performing vs web publishers?"

```cypher
MATCH (p:Publisher)-[:ON_PUBLISHER]-(pl:Placement)-[:HAS_PLACEMENT]-(c:Campaign)
WHERE pl.status = 'completed'
RETURN p.channel_type,
       count(pl) AS total_placements,
       avg(pl.actual_cpc) AS avg_cpc,
       avg(pl.ctr) AS avg_ctr,
       sum(pl.spend) AS total_spend,
       avg(pl.impressions) AS avg_impressions
ORDER BY total_placements DESC
```

---

## 5. Data Pipeline

### Phase 1: Direct Import from Linkby DB

```
Linkby PostgreSQL / Data Warehouse
         │
         ├── publishers table ──────► :Publisher nodes
         ├── advertisers table ─────► :Advertiser nodes  
         ├── campaigns table ───────► :Campaign nodes
         ├── placements table ──────► :Placement nodes + relationships
         ├── pitches table ─────────► :Pitch nodes + relationships
         ├── industry_grouping ─────► :Topic nodes + SPECIALISES_IN / IN_INDUSTRY edges
         ├── cluster_assignments ───► SIMILAR_AUDIENCE edges (publisher-to-publisher)
         └── cpc_predictions ───────► predicted_cpc property on Campaign nodes
```

### Phase 2: ML-Enriched Signals

```
Campaign brief text ──► Claude Sonnet extraction ──► TARGETS edges to Topics
Publisher content   ──► Embedding model ──────────► publisher.embedding property  
Campaign briefs     ──► Embedding model ──────────► campaign.brief_embedding property
Audience data       ──► Overlap analysis ─────────► REACHES edges to AudienceSegments
```

### Phase 3: Feedback Loop

```
Completed placement ──► Update PERFORMED_FOR aggregates
                    ──► Update SIMILAR_AUDIENCE weights
                    ──► Update publisher.avg_cpc_delivered
                    
Accepted/rejected pitch ──► Update PITCHED_TO.acceptance_rate
                        ──► Retrain pitch scoring model
```

---

## 6. Schema Diagram

```
                         ┌──────────┐
                    ┌───►│  Topic   │◄──── PARENT_OF ────┐
                    │    └──────────┘                     │
           TARGETS  │         ▲                     ┌──────────┐
                    │         │ SPECIALISES_IN      │  Topic   │
                    │         │                     └──────────┘
              ┌─────────┐    │    ┌────────────┐
              │Campaign │    │    │ Publisher   │◄─── SIMILAR_AUDIENCE ───► Publisher
              └────┬────┘    │    └─────┬──────┘
  CREATED_        │         │          │
  CAMPAIGN   HAS_ │         │          │ REACHES
     ▲      PLACEMENT       │          ▼
     │          │           │    ┌──────────────┐
┌────────────┐  ▼           │    │  Audience    │
│ Advertiser │ ┌──────────┐ │    │  Segment     │
└────────────┘ │Placement │─┘    └──────────────┘
     ▲         └──────────┘
     │          ON_PUBLISHER
     │
     │ TO_ADVERTISER
     │
  ┌──────┐  PITCHED    ┌───────────┐
  │Pitch │◄────────────│ Publisher  │
  └──┬───┘             └───────────┘
     │
     └──── FOR_CAMPAIGN ────► Campaign
```

---

## Next Steps

1. **Map to actual Linkby DB schema** — which tables/columns map to which node properties?
2. **Estimate data volumes** — how many publishers, advertisers, campaigns, placements, pitches?
3. **Spin up Neo4j AuraDB free tier** — load a sample dataset
4. **Test Q2 (pitch scoring) first** — most direct business impact, clearest success metric
5. **Benchmark against existing recommendation model** — does graph-based matching beat current model?
