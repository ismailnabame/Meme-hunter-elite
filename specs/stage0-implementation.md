# MEME HUNTER ELITE — Stage 0 Implementation Spec

**Version:** 0.1 (frozen on sign-off)
**Depends on:** v1 Hypothesis Spec v0.1
**Purpose:** Build the minimum infrastructure to prove or disprove the v1 hypothesis, from GitHub Codespaces, on a phone, at ~$0/month starting cost.

---

## 0. What Stage 0 delivers (and does not)

**Delivers:**
1. A working Solana data ingestion pipeline (Helius + backup).
2. A feature extractor that turns any token mint into a T+60s feature vector.
3. A historical dataset of ~10k–50k labeled tokens (feature vectors + 24h outcome).
4. A replay harness that runs the v1 entry/exit rules against that dataset.
5. A results report answering: *did the hypothesis hold?*

**Does NOT deliver:**
- Any live trading
- Any dashboard
- Any scoring beyond the v1 rules
- Any wallet intelligence
- Any execution code

If any of these creep in, Stage 0 has failed even if the code works.

---

## 1. Repo structure (freeze this — everything lives here)
