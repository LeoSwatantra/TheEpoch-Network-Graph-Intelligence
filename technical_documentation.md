# Technical Documentation — AML Graph Intelligence Engine
### Track 3: Network & Graph Intelligence

## 1. Problem framing

Money laundering is rarely a single transaction. It is a process of splitting funds across
many accounts (smurfing), relaying them through chains of intermediaries (layering), and
circulating them back to the origin (round-tripping). These behaviours are invisible when
transactions are viewed individually but become visible as structural shapes once the
data is modelled as a graph. Our task is to model the transactions as a network, detect those
shapes, and produce a ranked, explainable list of suspicious accounts and subgraphs.

## 2. Data

| File | Role | Notes |
|---|---|---|
| `transactions.csv` | 100,222 transfers (edges) | sender, receiver, amount, date, time + engineered fields |
| `accounts.csv` | 65,339 accounts (nodes) | KYC: PEP flag, sanctions hit, risk grade, type, open date |
| `ml_features.csv` | per-transaction features | carries the `is_suspicious_tx` label |
| `graph_edges.csv` | edge subset | superseded by `transactions.csv` |
| `reports/` | STR XML |

**Amount handling:** we use `amount_local_npr` (NPR-normalized) as the edge weight so amounts
across currencies are comparable.

**Data quality (from EDA):** zero missing values, zero duplicate transactions, zero
self-loops, zero non-positive amounts, zero orphan account IDs. Time span ≈ 1 month
(2022-10-07 → 2022-11-06). The suspicious-transaction rate is 0.335%(336/100,222).

**Critical structural finding:** the transaction network is not one connected web. It
fragments into ~64,000 disconnected components, the largest of which has only 43
accounts, and the largest directed loop spans only 7 accounts. Only ~3,557 accounts
both send and receive (the "pass-through" accounts where layering can physically occur).


## 3. Graph construction

We maintain two synchronized representations:

- **Temporal multigraph** (`MultiDiGraph`): one edge per transaction, retaining the exact
  timestamp and amount. Used for time-sensitive detectors (fan-out/in bursts, forwarding,
  time-respecting chains).
- **Aggregated weighted digraph** (`DiGraph`): one edge per ordered account pair, weighted by
  total NPR flow, with transaction count and first/last timestamps. Used for centrality,
  connected components, and cycle enumeration.

KYC attributes are attached to nodes for the scoring stage.

## 4. Structural feature engineering

Per account: in/out degree, weighted in/out (total NPR), unique counterparties,
**flow ratio** (`total_out / total_in`; ≈1.0 = pass-through), net flow, reciprocity,
strongly/weakly connected component size, cycle membership, PageRank, k-sample approximate
betweenness, and clustering coefficient.

**Empirical signal strength** (top-200 lift over base rate, measured on our features):

| Signal | Lift |
|---|---|
| Flow-ratio ≈ 1.0 (pass-through) | 32.8× |
| Out-degree (fan-out) | 13.3× |
| In-degree (fan-in) | 10.2× |
| Betweenness (bridge) | 6.1× |
| PageRank | 3.1× |
| Raw volume | 2.0× |

The dominance of flow-ratio confirmed that, in this data, laundering is about money
moving through accounts, not accumulating in large ones and we weighted scoring
accordingly.

## 5. Pattern detection

All detectors are time-aware and, where relevant, amount-aware:

1. **Pass-through / rapid forwarding** — for each pass-through account, match each outgoing
   transfer to a recent incoming transfer of similar amount (70–105%, allowing fee erosion)
   within a 48h window. Repeated matches ⇒ mule behaviour.
2. **Temporal fan-out / fan-in** — a sliding 24h window finds the maximum number of distinct
   counterparties in any burst (a two-pointer algorithm, O(transactions)).
3. **Amount-conserving cycles** — enumerate simple cycles (restricted to the 1,525 accounts
   that can be in a loop), score each by `min(edge amount)/max(edge amount)`; high
   conservation ⇒ deliberate round-tripping.
4. **Time-respecting layering chains** — within each small island, enumerate paths from
   "source" accounts (send-only) to "sink" accounts (receive-only); keep paths whose hops are
   in chronological order and preserve amount. Reported for length ≥ 4 accounts.
5. **Dedicated-pair** — accounts whose activity concentrates (≥80%) on a single counterparty
   over ≥4 transfers — a fixed laundering pipe rather than ordinary spending.

## 6. Risk scoring (structure-first)

We rank accounts by confidence of structure, not volume. Exact detectors that rarely fire
on legitimate accounts receive the highest weight; broad heuristics that also flag legitimate
hubs receive the lowest:

```
conserved cycle  >  layering chain  >  rapid forwarder  >  dedicated pair
                 >  pass-through flow  >  fan-in / fan-out  >  centrality
```

Each account's score is the sum of points from the patterns it matches. A legitimate
high-fan-out payroll account with no cycle/chain/forwarding therefore ranks far below a quiet
4-account laundering ring — which is the correct behaviour. Subgraphs (islands) are scored
analogously by the worst behaviour they contain.

## 7. Results

- **Precision@100 = 20.0%**, a **41× lift** over the 0.488% account-level base rate.
- **Precision@200 = 16.0% (32.8×)**, **Precision@500 = 15.4% (31.5×)**.
- **74 of 171 injected laundering-ring accounts rediscovered using graph structure only** —
  no label, no account-ID shortcut.
- The top-ranked island is a 4-account loop (`900000000152→153→154→155→152`) circulating
  ~NPR 8M with the amount preserved at each hop, found blind by the
  cycle detector and 4/4 confirmed suspicious.

**Evaluation choice:** we report **precision@K** rather than accuracy, because at a 0.49% base
rate accuracy is meaningless (a "flag nothing" model scores 99.5%). Precision@K reflects the
real analyst workflow: only the top of the queue gets investigated.

## 8. Explainability

Every flagged account and island receives an auto-generated investigator narrative composed
purely from its structural features, e.g.:

> *"Account X received NPR 8.1M and sent NPR 7.9M over the period. It sits on a circular money
> flow where the amount that leaves returns almost unchanged (classic round-tripping), and
> forwarded money it had just received on 6 occasions, the fastest within 2.3 hours (flow
> ratio 0.98 — it keeps almost nothing). Pattern resembles layering and round-tripping."*

## 9. Limitations & next steps

- Synthetic, single-currency-normalized, ~1-month dataset; thresholds are tuned to it.
- Detectors are rule-based and explainable by design; an embedding-based anomaly detector
  (Node2Vec + Isolation Forest) is the planned extension to surface unknown typologies.
- The explainability + visualization layer is dashboard-ready. Productionizing it as a live
  Streamlit investigator console is the next engineering step.
