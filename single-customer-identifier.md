Field-Tested Innovation · Fuel & Convenience Retail · bp

# Creative Solutions: The Single Customer Identifier (SCI)

How linking three transaction datasets that were never designed to talk to each other (till receipts, loyalty swipes and card payments) turned millions of anonymous forecourt transactions into a single, measurable view of the customer.

Agnivesh Das · Senior Data Analytics Engineer · bp · 2022–2024

---

## My Role

### End to end: from raw source data to the matching model

As Senior Data Analytics Engineer on SCI, I owned the path from raw data to a linked customer view:

- **Sourcing.** Brought in raw EPOS, Loyalty and Payment data from each of the source systems.
- **Cleaning and modelling.** Cleaned and standardised that data into a common data model so the three sources could be compared like for like, market by market.
- **Matching.** Built the statistical models that find approximate matches between records across the three datasets, creating the links that join a basket, a card and a loyalty account into one picture of the customer.

---

## The Problem

### Three records of the same purchase, and no way to connect them

A single purchase at a bp forecourt can create up to three separate records, each in a different system:

- **EPOS** (the till) knows *what* was bought: the basket, items, quantities and value.
- **Loyalty** knows *who* bought it, but only if the customer swiped a loyalty card.
- **Payment** knows *how* it was paid for, through a tokenised card.

None of these systems shared a common key. The business could see a basket, *or* a loyalty member, *or* a card, but never the customer behind all three.

That left the most basic commercial questions without a reliable answer. How many customers do we actually have? How many are engaged with loyalty, and how many shop with us regularly but never swipe? Which customers drive our margin? Loyalty data covered only a fraction of spend, and everything outside it was effectively invisible.

---

## The Constraint Nobody Was Naming

### There was no ground truth to learn from

Linking the datasets was not a simple join. It was an **unsupervised matching problem**: there was no labelled list of "this till receipt is definitely this card payment" to train or test against. Any approach had to answer three questions at once:

**1. Which records are actually the same purchase?** Clocks across systems didn't agree. In one market the till timestamps were missing seconds entirely, and payment timestamps had known accuracy issues. Values and quantities could differ slightly between systems.

**2. How sure are we about each match?** At a busy site, several transactions of a similar value can happen within the same minute. Picking the closest one isn't enough if two or three others were almost as close.

**3. Which transactions can be matched at all?** A cash purchase will never appear in payment data. Counting it as "unmatched" makes the model look worse than it is, so the in-scope population had to be defined for every pair of datasets.

The business didn't need a stricter join. It needed a matching model that could say how confident it was in every link it made.

---

## The Fix

### Probabilistic linking, a confidence score, and a layered data product

SCI links EPOS, Loyalty and Payment transactions through a scored, tolerance-based matching model, then rolls the linked transactions up into customer-level KPIs. Three tunable controls sit at the heart of it:

**X: Match Window**
The time tolerance between two records (from exact match up to a few minutes, depending on the market and the dataset pair) within which they can be considered the same purchase.

**Y: Value & Quantity Tolerance**
How closely the sale value and item quantity must agree, either an exact match or a small range, to remain a feasible candidate.

**Z: Confidence Threshold**
The minimum confidence a link must reach before it is accepted, balancing match quality against coverage.

### How it works

1. **Refine:** Clean EPOS, Loyalty and Payment source data into one common data model, per market.
2. **Block:** Generate candidate pairs only within the same site, date and country, and also basket type (fuel / shop / both) and card type where relevant.
3. **Score:** Calculate the similarity of each candidate pair (cosine similarity across the thresholded fields), on a scale from 0 (at the edge of every tolerance) to 1 (an exact match).
4. **Dampen:** Penalise matches that have competing alternatives: each similarity is scaled by its share of the total similarity of all feasible candidates.
5. **Link:** Keep the best match above the confidence threshold, then chain EPOS–Payment–Loyalty into a single transaction record carrying both the payment token and the loyalty account ID.
6. **Measure:** Aggregate linked transactions into customer KPIs: loyalty vs. non-loyalty, engaged / semi-engaged / unengaged, fleet vs. non-fleet, and margin contribution. Coverage and confidence are reported alongside every result.

**Confidence = Similarity × Dampening Factor**
**Dampening Factor = Similarity ÷ Sum of similarities of all feasible matches**

Every link is therefore self-explanatory: which records were joined, how close they were, and how much competition that match had. A high-confidence link means "very close, and nothing else came near."

---

## Worked Example

### What the confidence score actually does

Below is the shape of how SCI resolves a match between till (EPOS) and payment records.

> **Note:** the method shown reflects how the model actually worked. The transactions and scores are illustrative, not real production data.

**Scenario:** two card payments (P1 and P2) and three till transactions (E1, E2 and E3) at the same site, within the same few minutes.

| Pair  | Similarity | Feasible alternatives | Confidence                       |
| ----- | ---------- | --------------------- | -------------------------------- |
| E1–P1 | 0.85       | None                  | **0.85**                         |
| E2–P2 | 0.94       | E3                    | **0.61** = 0.94 × 0.94 / (0.94 + 0.52) |
| E3–P2 | 0.52       | E2                    | **0.19** = 0.52 × 0.52 / (0.94 + 0.52) |

- **E1–P1** isn't a perfect match (the timestamps differ slightly), but nothing else competes for it, so confidence stays high.
- **E2–P2** is the closest match for P2, but E3 is also a plausible candidate, so SCI deliberately lowers its confidence. A raw "best match" approach would have treated it as certain.
- **E3–P2** is rejected.

**The trade-off:** a higher confidence threshold gives cleaner links but fewer of them. Plotting coverage against threshold showed that most matches clustered at very high confidence, so a threshold around 0.7 retained almost all achievable coverage while filtering out ambiguous links.

---

## Built to Scale

### From a UK proof of concept to a multi-market data product

SCI started as a UK proof of concept in 2022. It was engineered to grow:

- **Four layers, each smaller than the last.** Refine → Integrate → Aggregate → Metrics. Data volume falls at each step, so downstream queries get cheaper and faster, and the early layers load incrementally.
- **Traffic-light change rules.** Experiments are free (green), the Refine layer follows the rules where possible (amber), and the Integrate, Aggregate and Metrics layers must follow strict modelling rules (red): no duplicated fields in fact tables, no dimension tables that already exist elsewhere.
- **Market-specific tuning.** Match windows and tolerances were adapted to each market's data quality, for example by checking *before* the till time in a market where till timestamps lacked seconds.
- **A development framework.** In 2024 the team moved SCI from *model development* (frequent model changes) to *model refinement* (frequent data refreshes, fewer model changes), with batch deployments, code-review acceptance criteria and a clear onboarding path.

---

## Why This Beat The Old Approach

### Same data, fundamentally different view of the customer

|                              | Before SCI                                         | With SCI                                                   |
| ---------------------------- | -------------------------------------------------- | ---------------------------------------------------------- |
| View of the customer         | Loyalty members only                               | Loyalty and non-loyalty customers, linked by card token    |
| Linking EPOS, Loyalty, Payment | Not possible: no common key                      | Probabilistic linking across all three                     |
| Trust in a match             | n/a                                                | Every link carries a 0–1 confidence score                  |
| Measuring the gaps           | Unmatched transactions looked like failures        | Scope vs. coverage separates "unmatchable" from "missed"   |
| Customer segmentation        | Loyalty engagement only                            | Engaged / semi-engaged / unengaged, fleet, margin pareto   |
| Rolling out to a new market  | Rebuild from scratch                               | Same layered model, tuned per market                       |

---

## Impact

### What changed

**3 → 1**
Three disconnected transaction datasets joined into a single customer view.

**[~90%]**
Of in-scope UK transactions successfully linked. *[Confirm OK to publish]*

**3 markets**
UK, Germany and Poland running on the same layered model.

**Single source of truth**
From 2023, bp's Customer Lifetime Value (CLV) model consumed SCI as its single source of customer truth.

The bigger shift wasn't just joining three tables. It was turning "we know our loyalty members" into "we know our customers, including the ones who never swipe a card, and we know how confident we are in every link." That gave pricing, marketing and loyalty teams a customer base they could size, segment and value, instead of a fraction of it.

---

Created by Agnivesh Das · bp Customer & Product Data · Illustrative example: transactions and scores fictionalised. EPOS = electronic point of sale (till) data.
