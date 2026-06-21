# AML Graph Intelligence Engine

A graph-based anti-money-laundering (AML) system that models financial transactions as a
network, detects the structural shapes of laundering (smurfing, layering, round-tripping),
and produces a ranked, explainable list of suspicious accounts and subgraphs.

> # Team:TheEpoch

---

## Results

- Modelled 100,222 transactions across 65,339 accounts as a directed, time-aware graph.
- Built 5 structural laundering detectors (pass-through forwarding, temporal fan-out,
  temporal fan-in, time-respecting layering chains, amount-conserving cycles) plus a
  dedicated-pair detector.
- Combined them into a structure-first risk score** that ranks accounts by the *confidence
  of the laundering structure* they sit in — not by how busy they are.
- Precision@100 = 20% (a 41× lift over the 0.49% base rate).
- Rediscovered 74 of 171 injected laundering-ring accounts using graph structure alone —
  never using the suspicious-transaction label or the accounts' IDs.

---

## Why this is not a "degree centrality" project

High-degree accounts are easy to find and mostly legitimate(payroll, popular merchants).
We verified this directly: our highest fan-out / fan-in accounts were not suspicious. Real
signal comes from flow direction, amount conservation, timing, and multi-hop paths. Our
detectors are built around those four ideas, and our scoring deliberately ranks exact,
time-aware structures above volume heuristics**.

---

## The five laundering patterns we detect

| Pattern | Plain-English shape | Why it's suspicious |

|---|---|---|
| Pass-through / forwarding | money received is forwarded almost immediately, keeping ~nothing | mule / relay account (our strongest single signal — 32.8× lift) |
| Temporal fan-out | one sender → many receivers within hours| smurfing / structuring |
| Temporal fan-in | many senders → one receiver within hours | funnelling into a collector |
| Layering chain | A→B→C→D, each hop after the last, amount preserved | distancing dirty money from its origin |
| Circular flow | money loops back to its origin, amount preserved | round-tripping to fake legitimacy |

All temporal detectors use a sliding time window so "sent to 10 accounts over a month"
is not confused with "sent to 10 accounts in an hour."

---

## Pipeline

1. Data understanding — profiled all files; found the data is clean (no missing values,
   duplicates, self-loops, or orphan accounts) and that the network shatters into ~64,000
   tiny disconnected islands (largest = 43 accounts). This shaped the whole approach:
   each island is analysed exactly as its own "case."

2. Graph construction — two synchronized views: a temporal multigraph(one edge per
   transaction, keeps timestamps) for time-sensitive patterns, and an aggregated weighted
   digraph for centrality, components, and cycles.

3. Structural features — in/out degree, weighted flow, flow-ratio, reciprocity,
   SCC/WCC membership, PageRank, approximate betweenness, clustering.

4. Pattern detection — the five detectors above, each time- and amount-aware.

5. Risk scoring — structure-first additive score, weighted by each signal's measured
   lift; exact detectors (cycles, chains) outrank heuristics (fan size).

6. Explainability — every flagged account/island gets an auto-generated
   investigator-style narrative built only from its structural features.

7. Investigator visuals — top suspicious islands rendered as annotated flow diagrams.

## How to run

Open the notebooks in Google Colab in order (02 → 03 → 04), with the dataset folder mounted
at `/content/drive/MyDrive/STUDENT_DATASET`. Each notebook saves artifacts to `outputs/`
that the next one loads.

## Key deliverables

- `outputs/ranked_accounts.csv` — every account, its risk score, and a human-readable
  explanation of why it was flagged.
- `outputs/ranked_subgraphs.csv` — suspicious account clusters ("islands"), ranked.
- `outputs/figures/` — annotated diagrams of the top suspicious structures.

## Limitations & next steps

- Dataset is synthetic and covers ~1 month, thresholds are tuned to it.
- Next: Node2Vec embeddings + Isolation Forest to catch unknown typologies, and a live
  Streamlit investigator dashboard wrapping the existing explainability layer.
