Technical Deep Dive · Single Customer Identifier (SCI) · bp

# How the Matching Works: From Raw Transactions to a Single Customer View

A walkthrough of the models behind SCI: how three disconnected datasets are cleaned, paired, scored, linked and finally rolled up into customer-level insight.

Agnivesh Das · Senior Data Analytics Engineer · bp

> **Note:** this describes the approach and the model logic. All parameters, transactions and figures in the examples are illustrative, not production values.

---

## The Pipeline at a Glance

| Stage | Model / technique | Output |
| ----- | ----------------- | ------ |
| 1. Standardise | Rule-based cleaning into a common data model | Clean EPOS, Loyalty and Payment transactions |
| 2. Generate candidates | Blocking + tolerance windows | Feasible record pairs |
| 3. Score | Multi-field similarity (cosine) | Similarity score per pair, 0–1 |
| 4. Penalise ambiguity | Dampening factor | Confidence score per pair, 0–1 |
| 5. Assign | Best-match selection with a confidence threshold | One accepted link per transaction |
| 6. Triangulate | Three-way reconciliation across dataset pairs | One linked transaction record |
| 7. Resolve identity | Token ↔ account linking across transactions | Customer-level identifiers |
| 8. Enrich | Fleet co-transaction models | Fleet-linked convenience spend |
| 9. Aggregate & segment | KPI aggregation + rule-based segmentation | Customer profiles and metrics |
| 10. Validate | Scope, coverage, confidence, accuracy | Trust in every number |

---

## Stage 1: Standardise the Sources

Before anything can be matched, the three datasets have to describe a purchase in the same way.

- **EPOS (till):** basket items, quantities, sale value, site, timestamp.
- **Loyalty:** account ID, sale value, quantity, site, timestamp. Only sales and refund events are kept.
- **Payment:** tokenised card, card type, sale value, site, timestamp.

Key cleaning rules:

- **Common keys and units.** Align site codes, country, dates and currency so the same purchase looks the same in every source.
- **Basket type.** Classify each transaction as *fuel*, *shop* or *fuel + shop*, so a fuel-only payment is never matched to a shop-only receipt.
- **Exclusions.** Remove transaction types that should never be linked to a person, such as tokens tied to non-customer or partner-scheme transactions.
- **Timestamp quirks.** Record known clock issues per market (for example, till timestamps with no seconds) so later stages can widen or shift their time windows.

---

## Stage 2: Generate Candidate Pairs (Blocking)

Comparing every transaction with every other one is impossible at this scale. SCI uses **blocking**: only records that share hard attributes are ever compared.

**Hard blocking keys (must match exactly):**
- Site
- Date
- Country
- Basket type and card type (for Payment ↔ Loyalty)

**Soft tolerances (must fall within a window):**

| Field | Tolerance (illustrative) |
| ----- | ------------------------ |
| Time difference | From exact, up to about 1–3 minutes depending on the dataset pair and market |
| Sale value | Exact match, or within a small range |
| Quantity | Exact match, or within a small range |

The tolerances are tuned per dataset pair and per market. For example, in a market where till timestamps lack seconds, the window extends *before* the till time as well as after it.

Every pair that survives blocking and tolerances is a **feasible match**. Matching is run separately for each dataset pair:

- EPOS ↔ Payment (cash transactions are out of scope)
- EPOS ↔ Loyalty
- Payment ↔ Loyalty

---

## Stage 3: Score Each Pair (Similarity Model)

Each feasible pair gets a **similarity score** between 0 and 1.

1. For each thresholded field (time, value, quantity), measure how far apart the two records are **relative to that field's tolerance**: zero difference scores as fully close, and a difference at the edge of the tolerance scores as least close.
2. Combine the fields into a vector and compute the **cosine similarity** of the pair against a perfect match.

The result:
- **1.0** = an exact match on every field (same time, same value, same quantity).
- **0.0** = a match that sits at the limit of every tolerance.

Because every field is normalised by its own tolerance, a 30-second gap and a 5p value difference can be compared on the same scale.

---

## Stage 4: Penalise Ambiguity (Confidence Model)

High similarity alone isn't enough. At a busy site, several transactions can look almost identical. SCI turns similarity into **confidence** by penalising matches that have competition:

```
dampening factor = similarity ÷ (sum of similarities of all feasible matches)
confidence       = similarity × dampening factor
```

- A pair with **no competing candidates** keeps its full similarity as confidence.
- A pair with **close competitors** has its confidence reduced in proportion to how strong those competitors are.

This is the key design choice of SCI: because there is no labelled ground truth, the model has to measure its own certainty.

---

## Stage 5: Assign the Best Match

For each transaction, SCI:

1. Ranks its feasible matches by confidence.
2. Keeps the highest-confidence match.
3. Accepts it only if it clears the **confidence threshold**.

**Choosing the threshold.** Plotting the confidence distribution against coverage showed that most matches cluster at very high confidence. A threshold around **0.7** kept almost all achievable coverage while discarding the ambiguous tail.

### Worked example

Two payments (P1, P2) and three till transactions (E1, E2, E3) at the same site, in the same few minutes:

| Pair  | Similarity | Competitors | Confidence | Decision |
| ----- | ---------- | ----------- | ---------- | -------- |
| E1–P1 | 0.85 | None | 0.85 | ✅ Accepted |
| E2–P2 | 0.94 | E3 | 0.61 = 0.94 × 0.94 / (0.94 + 0.52) | ⚠️ Best for P2, below 0.7 |
| E3–P2 | 0.52 | E2 | 0.19 = 0.52 × 0.52 / (0.94 + 0.52) | ❌ Rejected |

E2–P2 is clearly the better candidate for P2, but the model flags that it isn't *sure*. Whether it is kept depends on the threshold the business is willing to accept.

---

## Stage 6: Triangulate Across All Three Datasets

The three pairwise models each produce their own links. The final step joins them into a single transaction record:

```
EPOS basket ──(E↔P)── Payment token
     │                     │
   (E↔L)                 (P↔L)
     │                     │
     └──── Loyalty account ┘
```

- The **EPOS basket** is the anchor: it holds *what* was bought.
- The **payment token** and the **loyalty account ID** are attached to it from the accepted pairwise links.
- Where all three links agree, the record is very strong. Where only two exist, the third can be inferred. Where links conflict, the higher-confidence path wins.

The output is one row per purchase, carrying the basket, the card token and (if present) the loyalty account, along with the confidence of each link.

---

## Stage 7: Resolve Identity (From Transactions to Customers)

A linked transaction is still just one purchase. The customer emerges when links repeat over time.

- **Token ↔ account links.** If the same card token appears with the same loyalty account across many transactions, that card can be attributed to that loyalty customer, *even on visits where they didn't swipe their loyalty card*.
- **Card-only customers.** Tokens that never link to a loyalty account still form a customer record of their own, keyed by card.
- **Token dampening.** One person can pay with several cards, so a count of tokens overstates the number of people. A **card-token dampening factor** estimates how many tokens correspond to one customer, so token counts can be converted into customer counts.

---

## Stage 8: Enrich With Fleet Models

Fleet (business) fuel cards behave differently: a driver often pays for fuel on a fleet card and then buys a coffee or snack on a personal card or in cash.

Two models capture this:

- **Fleet → card convenience:** finds convenience payments made within a short time window and in sequential order after a fleet fuel transaction.
- **Fleet → cash convenience:** finds unmatched cash till transactions that follow a fleet transaction in the same window.

This reveals convenience spend that is driven by fleet customers but would otherwise be invisible.

---

## Stage 9: Aggregate and Segment

Linked transactions are rolled up to customer level (loyalty account or card token) and segmented.

**Loyalty engagement (illustrative definitions):**

| Segment | Meaning |
| ------- | ------- |
| Engaged | Loyalty members who regularly swipe their card |
| Semi-engaged | Loyalty members whose card payments are linked, but who often don't swipe |
| Unengaged | Card-only customers with no loyalty link |

**Other cuts:**
- Loyalty vs. non-loyalty
- Fleet vs. non-fleet
- Active vs. inactive
- Fuel only, shop only, or fuel + shop

**Outputs (Metrics layer):**
- **Base KPIs:** customers, transactions and spend per segment.
- **Top-down view:** customers, transactions, spend, margin and revenue by fuel, shop and both.
- **Margin pareto:** customers ranked by their share of total margin, split by loyalty/non-loyalty and fleet/non-fleet. This shows which small group of customers drives most of the value.

---

## Stage 10: Validate

Every number SCI produces comes with measures of how far it can be trusted.

| Metric | Question it answers |
| ------ | ------------------- |
| **Scope** | What share of transactions *could* be matched for this dataset pair? (For example, cash is out of scope for payment matching.) |
| **Coverage** | Of the in-scope transactions, what share *were* matched? |
| **Confidence** | How certain is each accepted match, and how is confidence distributed? |
| **Accuracy** (proposed) | Against a manually validated, randomly sampled set of known matches, how often is SCI right? |

Separating **scope** from **coverage** means the model is never blamed for transactions it could never match.

---

## End-to-End Example (Illustrative)

A customer buys fuel and a coffee at Site 0421 at 08:14.

1. **Standardise:** the till records a £48.20 fuel + shop basket; the payment system records £48.20 on a Visa token; the loyalty system records £48.20 against account A-7731.
2. **Candidates:** same site, same date, same basket type, all within the time window.
3. **Score:** EPOS ↔ Payment similarity 0.97; EPOS ↔ Loyalty 0.93; Payment ↔ Loyalty 0.95.
4. **Confidence:** no competing candidates within the window, so confidence equals similarity.
5. **Assign:** all three links clear the 0.7 threshold.
6. **Triangulate:** one record: basket + Visa token + account A-7731.
7. **Identity:** the same Visa token appears in 11 other transactions this quarter, only 4 of which had a loyalty swipe. All 12 are now attributed to A-7731.
8. **Segment:** A-7731 is classed as *semi-engaged*: a loyalty member whose real spend is three times what the loyalty data alone showed.

---

## Possible Extensions

The current model is deliberately transparent and rule-driven. Natural next steps:

- **Labelled accuracy set:** a manually verified sample of matches to measure accuracy directly.
- **Optimal one-to-one assignment:** using an assignment algorithm (for example, Hungarian matching) per site and time block so no record is claimed by two matches.
- **Probabilistic record linkage:** weighting each field by how informative it is (Fellegi–Sunter style) instead of treating fields equally.
- **Supervised scoring:** once a labelled set exists, training a classifier to predict match probability directly.
- **Graph-based identity resolution:** treating tokens and accounts as nodes in a graph and clustering them into households or multi-card customers.

---

Created by Agnivesh Das · bp Customer & Product Data · All examples and parameters illustrative.
