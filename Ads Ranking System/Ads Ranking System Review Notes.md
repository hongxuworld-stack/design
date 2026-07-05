# Ads Ranking System Review Notes

This file records the important follow-up points from our discussion. Use it as a quick review sheet before an interview.

## 1. Entity Hierarchy

Typical ads hierarchy:

```text
Advertiser
  -> Campaign
      -> AdGroup
          -> Ad
              -> Creative
```

Meaning:

1. `Advertiser`: the business buying ads.
2. `Campaign`: high-level objective, budget, and flight dates.
3. `AdGroup`: targeting, bid, pacing strategy, and optimization settings.
4. `Ad`: the serving and ranking unit.
5. `Creative`: the actual asset shown to the user.

Recommended interview simplification:

```text
One Ad maps to one Creative.
Multiple creatives can be represented as multiple Ads under the same AdGroup.
```

This makes ranking, stats, experimentation, and billing easier to explain.

## 2. What Is an AdGroup?

An `AdGroup` groups ads that share the same delivery strategy.

It usually owns:

1. Targeting rules.
2. Bid type and bid value.
3. Pacing strategy.
4. Optimization goal.

Example:

```text
Campaign: Nike Summer Sale
  daily_budget: $100,000
  objective: conversions

  AdGroup A: US iOS runners
    targeting: country = US, device = iOS, interest = running
    bid: $2 CPC
    ads: creative_1, creative_2

  AdGroup B: Canada Android fitness users
    targeting: country = Canada, device = Android, interest = fitness
    bid: $1.50 CPC
    ads: creative_3, creative_4
```

Interview line:

```text
Campaign controls the high-level budget and goal. AdGroup controls the concrete
delivery strategy: targeting, bid, pacing, and optimization settings.
```

## 3. What Is a Creative?

`Creative` is the user-visible ad asset.

Example:

```text
Ad:
  ad_id = ad_123
  ad_group_id = us_ios_runners
  creative_id = cr_456

Creative:
  creative_id = cr_456
  type = image
  title = "Nike Summer Sale"
  image_url = "..."
  landing_page_url = "..."
  call_to_action = "Shop Now"
```

Creative usually contains:

```text
creative_id
type: image / video / carousel / text
title
description
image_url / video_url
landing_page_url
call_to_action
review_status
format
size
```

Why creative matters:

1. Eligibility: the creative must be approved.
2. Placement compatibility: image/video/size must match the ad slot.
3. Quality: creative quality affects CTR, conversion, and user experience.
4. Rendering: client uses `creative_id` to display the final ad.

Interview line:

```text
Ad is the serving/ranking unit. Creative is what the user actually sees.
```

## 4. One Ad to One Creative or Many Creatives?

Both models exist.

Simpler model:

```text
Ad -> Creative
```

For A/B testing:

```text
AdGroup
  -> Ad 1 -> Creative A
  -> Ad 2 -> Creative B
  -> Ad 3 -> Creative C
```

More complex model:

```text
Ad
  -> Creative A
  -> Creative B
  -> Creative C
```

In the complex model, after ranking the ad, the system also needs creative selection.

Recommended interview answer:

```text
I will model one Ad as mapping to one Creative. If advertisers want to test
multiple creatives, we create multiple Ads under the same AdGroup. This keeps
ranking, metrics, and billing straightforward.
```

## 5. Why Return Tracking Tokens in the Rank API?

`tracking_tokens` are signed receipts for the serving decision.

Rank API response example:

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

Why not just let client report `ad_id`?

1. Client may be buggy or malicious.
2. Client-provided `ad_id` does not prove the ad was actually returned.
3. Server needs original context: request ID, rank position, model version, auction price, campaign, ad group, and creative.
4. Billing and attribution need trustworthy data.
5. Event deduplication needs stable event/request identifiers.

Token can contain or reference:

```text
request_id
impression_id
user_id
ad_id
creative_id
ad_group_id
campaign_id
placement_id
rank_position
auction_price
bid_type
model_version
expires_at
event_type
signature
```

Interview line:

```text
The client should not be trusted to report ad_id, campaign_id, price, or rank
position directly. The tracking token binds later events back to the original
signed serving decision.
```

## 6. How the Client Uses Tracking Tokens

After calling:

```http
POST /v1/ads:rank
```

The client receives ads and tokens.

When the ad becomes viewable:

```http
POST /v1/events/impression
```

```json
{
  "event_id": "evt_imp_001",
  "tracking_token": "signed_impression_token",
  "timestamp": "2026-07-05T12:00:00Z"
}
```

When the user clicks:

```http
POST /v1/events/click
```

```json
{
  "event_id": "evt_clk_001",
  "tracking_token": "signed_click_token",
  "timestamp": "2026-07-05T12:00:05Z"
}
```

When the user converts:

```http
POST /v1/events/conversion
```

```json
{
  "event_id": "evt_cvr_001",
  "click_token": "signed_click_token",
  "conversion_type": "purchase",
  "conversion_value": 59.99,
  "timestamp": "2026-07-05T12:10:00Z"
}
```

Server-side event flow:

```text
Validate token
  -> Check expiry
  -> Reconstruct serving context
  -> Deduplicate event
  -> Write to Kafka
  -> Update budget, frequency caps, reporting, features, and training data
```

## 7. Self-Contained Token vs Reference Token

### Option A: Self-Contained Signed Token

The token contains most serving fields.

Pros:

1. No extra lookup during event ingestion.
2. Simple and fast.

Cons:

1. Token can become large.
2. Sensitive data may be exposed unless encrypted.
3. Harder to change token schema.

### Option B: Signed Reference Token

The token contains an ID and signature. The full serving decision is stored server-side.

Example:

```text
token = signed_ref:imp_abc
```

Server stores:

```text
ServingDecisionStore:
  key: imp_abc
  value:
    request_id: req_789
    user_id: u123
    ad_id: ad_1
    campaign_id: cmp_1
    rank_position: 1
    auction_price: 0.82
    bid_type: CPC
    model_version: ranker_v42
    expires_at: ...
```

Pros:

1. Small token.
2. Sensitive details stay server-side.
3. Easier debugging and schema evolution.

Cons:

1. Event ingestion needs a store lookup.
2. ServingDecisionStore needs high availability and TTL.

Recommended interview answer:

```text
For simplicity and security, I would use a signed reference token. The token
contains request_id or impression_id, event_type, expiry, and signature. The
full serving decision is stored with a short TTL for attribution, billing, and
debugging.
```

## 8. What Is an Impression?

`Impression` means the ad was actually shown to the user, not just returned by the ranking API.

It is not mouse over.

Typical definition:

```text
The ad is rendered and viewable to the user.
```

Common standard:

```text
At least 50% of the ad pixels are in the viewport for at least 1 second.
```

For video ads:

```text
At least 50% of the video is visible and plays for at least 2 seconds.
```

Event differences:

```text
Ad returned:
  Ranking API returned the ad. User may not have seen it.

Impression:
  Ad was rendered and viewable. User had an opportunity to see it.

Click:
  User clicked the ad.

Conversion:
  User completed the advertiser's goal, such as purchase, signup, or install.

Hover / mouse over:
  User moved the mouse over the ad. This can be an engagement signal, but it is
  not normally counted as an impression.
```

Example:

```text
User opens feed.
System returns ad_1, ad_2, ad_3.

ad_1 is shown in the viewport.
  -> report impression

ad_2 is below the fold and user never scrolls to it.
  -> no impression

ad_3 fails to render.
  -> no impression

user hovers over ad_1.
  -> optional hover/engagement event, not impression

user clicks ad_1.
  -> report click
```

Interview line:

```text
An impression should fire only when the ad is actually rendered and viewable,
not merely when the ranking service returns it. This matters because impressions
drive CPM billing, reporting, frequency caps, and model training labels.
```

## 9. Key Fields to Highlight

Most important serving-time fields:

```text
Campaign.status
Campaign.daily_budget
Campaign.start_time
Campaign.end_time
Campaign.objective

AdGroup.bid_type
AdGroup.bid_value
AdGroup.targeting_rules
AdGroup.pacing_strategy

Ad.status
Ad.quality_score
Ad.category

Creative.review_status
Creative.type
Creative.format

BudgetState.remaining_budget
BudgetState.pacing_multiplier
BudgetState.last_updated_at
```

Why these matter:

1. They decide eligibility.
2. They affect auction math.
3. They protect user experience.
4. They prevent overspend.
5. They help ranking choose the right objective.

## 10. Useful Acronyms

Quick glossary:

```text
CTR  = Click-Through Rate
       clicks / impressions

CVR  = Conversion Rate
       conversions / clicks, or conversions / impressions depending on context

pCTR = predicted Click-Through Rate
       probability that the user clicks this ad

pCVR = predicted Conversion Rate
       probability that the user converts after click/view

CPC  = Cost Per Click
       advertiser pays per click

CPM  = Cost Per Mille
       advertiser pays per 1,000 impressions

CPA  = Cost Per Action / Acquisition
       advertiser pays per conversion/action

ROI  = Return on Investment
       advertiser value compared with ad spend

ROAS = Return on Ad Spend
       revenue attributed to ads / ad spend

RPM  = Revenue Per Mille
       platform revenue per 1,000 impressions

eCPM = effective Cost Per Mille
       normalized expected revenue per 1,000 impressions
```

Ranking formulas:

```text
CPC ad:
  expected_value = bid_CPC * pCTR

CPA ad:
  expected_value = bid_CPA * pCTR * pCVR

CPM ad:
  expected_value = bid_CPM / 1000

Final score:
  expected_value * quality_score * pacing_multiplier
```

Interview line:

```text
I convert different bid types like CPC, CPM, and CPA into a common expected
value, often eCPM or expected value per impression, so the auction can compare
them fairly.
```

## 11. High-Value Interview Sound Bites

Use these lines when explaining tradeoffs:

```text
Eligibility should happen before expensive ranking so we do not waste model
inference on ads that cannot serve.
```

```text
Ad ranking needs calibrated probabilities, not just relative ordering, because
predictions are multiplied by bids and affect auction outcomes.
```

```text
Tracking tokens make event attribution trustworthy without trusting
client-provided ad metadata.
```

```text
Online serving favors bounded latency and graceful degradation. Event and billing
pipelines favor correctness, deduplication, and auditability.
```

```text
Pacing is a feedback loop: spend events update live spend, pacing computes
control signals, and serving uses those signals to avoid over- or under-delivery.
```

```text
An impression is not when the ad is returned by the server. It is when the ad is
actually rendered and viewable to the user.
```
