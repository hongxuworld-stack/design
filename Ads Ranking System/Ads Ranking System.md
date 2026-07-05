# Ads Ranking System

## 0. Interview Framing

Design an ads ranking system that chooses the best ad for a user request, such as a feed slot, search result page, video break, or product page.

In the interview, scope it like this:

```text
I will focus on the online ad serving and ranking path. I will include campaign
management, event processing, budget pacing, reporting, and model training only
where they affect online ranking correctness, latency, or revenue.
```

The core problem is not just sorting ads by ML score. The system must select eligible ads, predict user response, run an auction, respect budgets, protect user experience, and feed events back into budget, reporting, and model training.

## 1. Requirements

### Functional Requirements

1. Return top N ads for a given user, placement, and page context.
2. Support advertiser-created campaigns, ad groups, ads, creatives, bids, budgets, and targeting rules.
3. Filter ads that are not eligible to serve.
4. Rank eligible candidates using relevance, predicted engagement/conversion, bid, quality, and pacing state.
5. Generate tracking tokens so impressions, clicks, and conversions can be attributed back to the served ad decision.
6. Process impression, click, and conversion events for budget updates, reporting, attribution, and model training.

### Non-Functional Requirements

1. Low latency: online serving p99 under 100-150 ms.
2. High availability: serving should degrade gracefully when features, models, or event pipelines are unhealthy.
3. High throughput: millions of ad requests per second at peak.
4. Fresh budget and feature signals: seconds to minutes for spend, frequency cap, and recent engagement.
5. Accurate money-related processing: events must be deduplicated and idempotent because they affect billing and advertiser reporting.
6. Privacy and policy compliance: honor user privacy choices and enforce ads policy before ranking.

Out of scope unless asked:

1. Full advertiser UI.
2. Full billing and payment settlement.
3. Full ML training platform internals.
4. Creative review implementation.

## 2. Capacity Estimation

Assumptions:

1. 1B daily active users.
2. 50 ad opportunities per user per day.
3. Average QPS: 1B * 50 / 86400 = about 580K QPS.
4. Peak QPS: 2M-3M.
5. Each request may start with thousands of eligible or retrievable ads, then shrink to hundreds for heavy ranking.

Design impact:

1. Online serving cannot synchronously query source-of-truth databases.
2. Campaign metadata, targeting indexes, budget state, frequency caps, and features need serving-optimized stores.
3. Ranking must be multi-stage to control inference cost and latency.
4. Events need a durable streaming pipeline because they update spend, reporting, attribution, and training data.

## 3. Core Entities

Typical hierarchy:

```text
Advertiser
  -> Campaign
      -> AdGroup
          -> Ad
              -> Creative
```

Entity meaning:

1. `Campaign`: high-level objective, flight dates, and budget.
2. `AdGroup`: targeting, bid, pacing strategy, and optimization settings for a group of ads.
3. `Ad`: the serving/ranking unit. In this design, one ad maps to one creative.
4. `Creative`: the actual user-visible asset, such as image, video, title, text, CTA, and landing page.
5. `BudgetState`: near-real-time spend and remaining budget used by online serving.
6. `AdRequest`: one opportunity to show one or more ads.
7. `Impression`, `Click`, `Conversion`: events used for billing, reporting, attribution, and ML feedback.

Key fields, with the most important serving-time fields marked as IMPORTANT:

```text
Campaign:
  campaign_id
  advertiser_id
  status              IMPORTANT: inactive campaigns must never serve
  daily_budget        IMPORTANT: budget filtering and pacing
  lifetime_budget
  start_time          IMPORTANT: eligibility filtering
  end_time            IMPORTANT: eligibility filtering
  objective           IMPORTANT: click, conversion, install, impression, etc.

AdGroup:
  ad_group_id
  campaign_id
  bid_type            IMPORTANT: CPC, CPM, CPA changes auction math
  bid_value           IMPORTANT: core input to expected value
  targeting_rules     IMPORTANT: determines eligibility for a request
  pacing_strategy     IMPORTANT: controls spend rate

Ad:
  ad_id
  ad_group_id
  creative_id
  status              IMPORTANT: paused/rejected ads must not serve
  quality_score       IMPORTANT: protects user experience and marketplace quality
  category            IMPORTANT: relevance, policy, and diversity

Creative:
  creative_id
  type                IMPORTANT: image, video, carousel, text
  review_status       IMPORTANT: unapproved creatives must not serve
  asset_url
  landing_page_url
  format              IMPORTANT: must match placement

BudgetState:
  campaign_id
  spent_today
  remaining_budget    IMPORTANT: fast online budget check
  pacing_multiplier   IMPORTANT: controls under/over-delivery
  last_updated_at     IMPORTANT: detects stale spend state
```

## 4. API

### Rank Ads

```http
POST /v1/ads:rank
```

Request:

```json
{
  "placement_id": "feed_top",
  "user_context": {
    "user_id": "u123",
    "device": "ios",
    "country": "US",
    "language": "en",
    "privacy_flags": {
      "personalized_ads_allowed": true
    }
  },
  "page_context": {
    "query": "running shoes",
    "content_id": "post_456",
    "category": "sports"
  },
  "limit": 3
}
```

Response:

```json
{
  "request_id": "req_789",
  "ads": [
    {
      "ad_id": "ad_1",
      "creative_id": "cr_1",
      "ad_group_id": "ag_1",
      "campaign_id": "cmp_1",
      "rank_position": 1,
      "tracking_tokens": {
        "impression": "signed_impression_token",
        "click": "signed_click_token"
      }
    }
  ]
}
```

Why tracking tokens are returned:

1. They bind later events to this specific ranking decision.
2. They prevent clients from forging ad IDs, prices, or campaign IDs.
3. They carry or reference `request_id`, `ad_id`, `creative_id`, model version, rank position, and auction price.
4. They support deduplication and attribution in the event pipeline.

### Event APIs

```http
POST /v1/events/impression
POST /v1/events/click
POST /v1/events/conversion
```

Example:

```json
{
  "event_id": "evt_123",
  "tracking_token": "signed_impression_token",
  "timestamp": "2026-07-05T12:00:00Z"
}
```

### Campaign APIs

```http
POST /v1/campaigns
PATCH /v1/campaigns/{campaign_id}
POST /v1/ad-groups
POST /v1/ads
```

REST is fine for external APIs. Use gRPC for internal low-latency service-to-service calls.

## 5. Online Ranking Data Flow

A realistic ad ranking path:

1. Ads Gateway receives request, authenticates, and validates placement/context.
2. Eligibility Filtering removes ads that cannot legally or economically serve.
3. Candidate Generation retrieves a smaller set of promising ads.
4. Lightweight Ranking reduces candidates to a few hundred.
5. Feature Service fetches user, ad, context, and cross features.
6. Heavy Ranking predicts pCTR, pCVR, expected conversion value, and user utility.
7. Auction Service combines predictions, bids, quality, budget, and business rules.
8. Post-Rank Rules enforce diversity, frequency caps, advertiser limits, and creative constraints.
9. Response returns top N ads and signed tracking tokens.
10. Events flow back through Kafka/Flink to update budget, pacing, frequency caps, reporting, and training data.

## 6. High-Level Design

```text
Client
  |
  v
Ads Gateway
  |
  v
Ad Serving Orchestrator
  |
  +--> Eligibility Service
  |       +--> Campaign Metadata Cache
  |       +--> Creative Status Cache
  |       +--> Privacy / Policy Rules
  |       +--> Budget State Cache
  |
  +--> Candidate Generation Service
  |       +--> Targeting Index
  |       +--> Keyword / Search Index
  |       +--> Audience Segment Store
  |       +--> Vector Retrieval Index
  |
  +--> Lightweight Ranker
  |
  +--> Feature Service
  |       +--> Online Feature Store
  |       +--> Real-Time Feature Cache
  |
  +--> Heavy Ranking Service
  |       +--> Model Server
  |
  +--> Auction Service
  |       +--> Pacing Service
  |       +--> Budget State Cache
  |
  +--> Tracking Token Service

Events:

Impression / Click / Conversion
  |
  v
Event Collector
  |
  v
Kafka
  |
  +--> Flink: dedupe, attribution, budget spend, frequency caps
  |       +--> Budget State Cache
  |       +--> Frequency Cap Store
  |       +--> Real-Time Feature Store
  |
  +--> Real-Time Analytics Store
  |
  +--> Data Lake
          +--> Offline Training
          +--> Reporting
          +--> Model Registry
```

## 7. Ranking and Auction Logic

For click-optimized ads:

```text
expected_value = bid_CPC * pCTR
```

For conversion-optimized ads:

```text
expected_value = bid_CPA * pCTR * pCVR
```

For value-based optimization:

```text
expected_value = pCTR * pCVR * predicted_conversion_value
```

Final ranking score:

```text
final_score =
  expected_value
  * quality_score
  * pacing_multiplier
  * user_experience_factor
  * exploration_factor
```

Important interview nuance:

1. Ads ranking needs calibrated probabilities, not just relative ordering, because bids and billing depend on predicted event rates.
2. Auction can introduce selection bias because models train on shown ads, but infer over candidate ads that may not have been shown.
3. The model must handle fast-changing ad inventory where new ads and campaigns appear constantly.
4. Delayed conversions mean training labels may arrive days or weeks later, so model freshness and label completeness trade off.

### Useful Acronyms

```text
CTR  = Click-Through Rate
       clicks / impressions

CVR  = Conversion Rate
       conversions / clicks, or conversions / impressions depending on context

pCTR = predicted Click-Through Rate
       model-predicted probability that the user clicks the ad

pCVR = predicted Conversion Rate
       model-predicted probability that the user converts after clicking or seeing the ad

CPC  = Cost Per Click
       advertiser pays when the user clicks

CPM  = Cost Per Mille
       advertiser pays per 1,000 impressions

CPA  = Cost Per Action / Acquisition
       advertiser pays when the user converts

ROI  = Return on Investment
       advertiser value compared with ad spend

ROAS = Return on Ad Spend
       revenue attributed to ads / ad spend

RPM  = Revenue Per Mille
       platform revenue per 1,000 impressions

eCPM = effective Cost Per Mille
       normalized expected revenue per 1,000 impressions, useful for comparing CPC, CPM, and CPA ads
```

How they appear in ranking:

```text
CPC ad expected value:
  bid_CPC * pCTR

CPA ad expected value:
  bid_CPA * pCTR * pCVR

CPM ad expected value:
  bid_CPM / 1000

eCPM-style comparison:
  expected_value_per_impression * 1000
```

## 8. Eligibility Filtering

Eligibility should happen before expensive ranking.

Checks:

1. Campaign, ad group, and ad are active.
2. Campaign is within start and end time.
3. Creative is approved and compatible with placement.
4. Targeting rules match the request.
5. User privacy flags allow this type of ad personalization.
6. Budget remains above threshold.
7. Frequency cap is not exceeded.
8. Policy, brand safety, blocked categories, and advertiser conflicts pass.

Stores:

1. Campaign metadata cache for status, dates, objective, and budget config.
2. Creative status cache for review state and format compatibility.
3. Budget state cache for near-real-time spend.
4. Frequency cap store keyed by user plus ad/campaign/window.

## 9. Candidate Generation

Candidate sources:

1. Targeting index by country, device, language, placement, interest, age range, and segment.
2. Keyword or query matching for search ads.
3. Semantic retrieval using user/context and ad embeddings.
4. Retargeting based on recent user behavior.
5. Audience segments and lookalike audiences.
6. Controlled exploration for cold-start ads.

Design:

1. Retrieve candidates from multiple sources in parallel.
2. Deduplicate by `ad_id`.
3. Apply cheap quality and budget thresholds.
4. Keep high recall for the heavy ranker.
5. Cap candidates to protect latency and inference cost.

Storage:

1. Targeting index: OpenSearch, Elasticsearch, or custom inverted index.
2. Audience membership: bitmap index or distributed KV store.
3. Embedding retrieval: ANN/vector index.
4. Metadata: source-of-truth DB plus serving cache.

## 10. Feature Store

Feature categories:

1. User features: interests, location, historical CTR, recent behavior.
2. Ad features: historical CTR/CVR, quality score, category, creative type.
3. Context features: placement, query, content category, time, device.
4. Cross features: user-ad affinity, user-category CTR.
5. Real-time features: last 5-minute CTR, budget consumption, recent clicks.

Freshness targets:

```text
Budget spend: seconds
Frequency caps: seconds
Recent user actions: seconds to minutes
Ad CTR/CVR aggregates: minutes
User profile: hours to days
Model version: hours to days
```

Key design points:

1. Batch feature reads for many candidates.
2. Store timestamps with features and ignore stale values when needed.
3. Provide default values for missing features.
4. Keep online and offline feature definitions aligned to reduce training-serving skew.
5. Use streaming jobs for real-time features and batch jobs for slower aggregates.

## 11. Model Serving

Model lifecycle:

```text
Data Lake
  -> Training Pipeline
  -> Offline Evaluation
  -> Model Registry
  -> Shadow / Canary / A/B Test
  -> Online Ranking Service
```

Serving behavior:

1. Lightweight ranker uses cheaper models or embeddings to reduce the candidate set.
2. Heavy ranker uses richer features and models to predict pCTR, pCVR, conversion value, and user utility.
3. Model Server supports batch inference per request.
4. Every response logs model version, feature version, request ID, and candidate/ranking metadata.

Fallbacks:

1. Heavy model timeout -> use lightweight score.
2. Feature service timeout -> use cached/default features.
3. Candidate source failure -> use remaining sources or cached fallback ads.
4. Bad model release -> rollback by model version.

## 12. Budget and Pacing

Budget prevents overspend. Pacing controls how quickly budget is spent across the campaign lifetime.

Why pacing should be its own service:

1. Pacing parameters are shared by many requests for the same campaign.
2. Recomputing pacing per ad request wastes CPU.
3. Time-based pacing can compute campaign-level control signals periodically.
4. A standalone service lets pacing evolve independently from the ad server.

Pacing feedback loop:

```text
Budget config + live spend + campaign time window
  -> Pacing Service
  -> pacing_multiplier / throttle probability / bid modifier
  -> Auction and serving path
  -> Impression/click/conversion events
  -> Live spend update
```

Typical controls:

1. `pacing_multiplier`: lowers or raises rank score.
2. `throttle_probability`: probabilistically skips under/over-paced campaigns.
3. `bid_modifier`: adjusts effective bid.

Tradeoff:

1. Strongly consistent budget checks reduce overspend but add latency and contention.
2. Eventually consistent spend is fast but can overspend under lag or failures.
3. Token bucket or budget reservation is a practical middle ground for small budgets or high-risk campaigns.

## 13. Event Pipeline

Events are money-related data, not just logs.

Responsibilities:

1. Validate tracking tokens.
2. Deduplicate repeated events.
3. Attribute clicks and conversions to impressions or requests.
4. Generate billable records.
5. Update budget spend and frequency caps.
6. Produce real-time analytics.
7. Store training data.

Architecture:

```text
Event Collector
  -> Kafka
  -> Flink jobs
      -> Dedupe
      -> Sessionization / attribution
      -> Billing event generation
      -> Budget spend aggregation
      -> Real-time feature aggregation
  -> Analytics Store
  -> Data Lake
```

Correctness:

1. Use `event_id` and signed tracking token for dedupe and idempotency.
2. Use event time and watermarks for late events.
3. Use idempotent writes or upserts downstream.
4. Monitor event lag because stale spend can cause overspend.
5. Exactly-once is hard; in interviews, say the goal is effectively-once externally using Kafka/Flink transactions plus idempotent sinks.

## 14. Storage Choices

1. Campaign source of truth: PostgreSQL or MySQL.
2. Serving metadata cache: Redis, Aerospike, or in-memory replicated cache.
3. Targeting/search index: OpenSearch, Elasticsearch, or custom inverted index.
4. Audience store: bitmap store or distributed KV store.
5. Online feature store: Redis, Aerospike, DynamoDB, or Cassandra.
6. Budget state cache: strongly monitored distributed KV store.
7. Event log: Kafka or Pulsar.
8. Stream processing: Flink.
9. Real-time analytics: Pinot, Druid, or ClickHouse.
10. Offline lake: object storage with Iceberg, Hudi, or Delta.
11. Model registry: MLflow or internal registry.

## 15. Deep Dives

### Deep Dive 1: Low Latency

Example latency budget:

```text
Gateway/auth: 5 ms
Eligibility filtering: 10 ms
Candidate generation: 15 ms
Lightweight ranking: 10 ms
Feature fetch: 20 ms
Heavy ranking: 25 ms
Auction/post-rank/token: 10 ms
Network overhead: 10 ms
Total: about 100 ms
```

Techniques:

1. Keep serving-critical state in caches or local replicas.
2. Query candidate sources in parallel.
3. Use multi-stage ranking to control model cost.
4. Batch feature and model requests.
5. Apply strict timeouts and return a lower-quality result rather than fail the page.

### Deep Dive 2: Budget Overspend

Problem:

If event processing or spend cache updates lag, the ad server may believe a campaign still has budget and continue serving it. This creates overspend or lost revenue because the platform may not be able to charge beyond the advertiser budget.

Mitigations:

1. Live spend counters update budget cache from event streams.
2. Pacing service reduces delivery as spend approaches expected spend.
3. Add safety margin near budget exhaustion.
4. Use token bucket or budget reservation for campaigns likely to overspend.
5. Alert on event pipeline lag and stale `BudgetState.last_updated_at`.
6. Reconcile final billing from durable event logs.

### Deep Dive 3: Event Correctness

Events affect advertiser bills and reporting, so correctness matters.

Approach:

1. Tracking token signs the original serving decision.
2. Event collector validates token and emits to Kafka.
3. Flink deduplicates by event ID and token-derived identifiers.
4. Attribution jobs join impressions, clicks, and conversions within attribution windows.
5. Sinks use idempotent writes/upserts.
6. Data lake stores immutable event history for auditing and model training.

### Deep Dive 4: Model Quality and Feedback Bias

Ad ranking has ML issues that are more subtle than normal recommendation systems:

1. Models train mostly on ads that won auctions, causing selection bias.
2. Predicted probabilities need calibration because they interact with bids and billing.
3. Conversion labels arrive late and can repeat.
4. New campaigns and creatives create sparse IDs and cold-start problems.

Mitigations:

1. Use exploration traffic.
2. Use calibrated models and monitor calibration by segment.
3. Use delayed-label aware training windows.
4. Use advertiser/category/creative priors for cold start.
5. Use content embeddings from creative text, image, and landing page.

Cold-start smoothing:

```text
smoothed_ctr = (clicks + alpha * prior_ctr) / (impressions + alpha)
```

### Deep Dive 5: Reliability

Failure handling:

1. Feature Store timeout -> cached/default features.
2. Heavy Model timeout -> lightweight model.
3. Candidate index outage -> use remaining sources or cached ads.
4. Pacing Service unavailable -> use last known pacing parameters with short TTL and conservative throttling.
5. Event pipeline lag -> reduce risky campaign delivery and alert.
6. Bad model release -> rollback by model version.

Observability:

1. Request metrics: latency, candidate count, filter reasons, model latency, feature missing rate.
2. Marketplace metrics: CTR, CVR, revenue, fill rate, advertiser ROI, user hide/report rate.
3. Budget metrics: overspend, underspend, stale budget state, pacing error.
4. Event metrics: ingestion lag, dedupe rate, attribution lag, invalid token rate.
5. ML metrics: calibration, drift, A/B test lift, segment regressions.

## 16. Interview Narrative

Good opening:

```text
I will design the online ads ranking path first. For each ad request, we need
to filter eligible ads, retrieve candidates, run multi-stage ranking, execute
an auction with budget and pacing constraints, and emit tracking tokens so
events can update spend, reporting, and training data.
```

Recommended flow:

1. Requirements: clarify serving, tracking, budget, latency, and availability.
2. Core entities: campaign, ad group, ad, creative, budget state, events.
3. API: rank ads and record events.
4. High-level design: online serving path plus event feedback path.
5. Deep dives: low latency, budget/pacing, event correctness, ML feedback issues.

Good interviewer-facing tradeoffs:

1. Eligibility before ranking saves expensive inference cost.
2. Pacing as a separate service avoids per-request recomputation and enables time-based controls.
3. Online serving favors availability and bounded latency; event/billing pipelines favor correctness and auditability.
4. Tracking tokens make event attribution trustworthy without trusting client-provided ad metadata.
5. Multi-stage ranking balances recall, relevance, latency, and inference cost.

## 17. One-Sentence Summary

An ads ranking system filters eligible ads, retrieves high-recall candidates, predicts value with fresh features and ML models, runs an auction under budget and pacing constraints, returns signed tracking tokens, and uses event feedback to update spend, reporting, and future ranking quality.

## References

1. Hello Interview, "System Design Delivery Framework": https://www.hellointerview.com/learn/system-design/in-a-hurry/delivery
2. Snap Engineering, "Machine Learning for Snapchat Ad Ranking": https://eng.snap.com/machine-learning-snap-ad-ranking
3. Twitter/X Engineering, "How we built Twitter's highly reliable ads pacing service": https://blog.x.com/engineering/en_us/topics/infrastructure/2021/how-we-built-twitter-s-highly-reliable-ads-pacing-service
4. Twitter/X Engineering, "How we fortified Twitter's real time ad spend architecture": https://blog.x.com/engineering/en_us/topics/infrastructure/2020/how_we_fortified_twitters_real_time_ad_spend_architecture
5. InfoQ, "Building a Large Scale Real-Time Ad Events Processing System": https://www.infoq.com/presentations/streaming-pipeline-ad-platform/
6. Uber Engineering, "Real-Time Exactly-Once Ad Event Processing with Apache Flink, Kafka, and Pinot": https://www.uber.com/us/en/blog/real-time-exactly-once-ad-event-processing/
