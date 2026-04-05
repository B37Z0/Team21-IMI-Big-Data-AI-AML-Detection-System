# 05 — Explanation Engine Architecture

## Purpose

`04-explanation-model.py` is the final stage of the AML pipeline. It takes the structured, ML-scored evidence produced by `03-hybrid-model.py` and converts it into plain-language investigative narratives with the intention of providing a clear, concise summary of a customer's risk profile without going in depth on any one aspect of the risk profile.

The script does not re-score customers, apply new rules, or alter rankings. Its sole function is intended to be a faithful, grounded translation. Although the LLM should not infer information beyond what is provided in the evidence, it would ideally produce a narrative by synthesizing connections in a 'compound' manner not immediately obvious from the evidence alone.

---

## Two-Layer Evidence Design

This is the most architecturally important decision in the script. The LLM receives **two distinct categories of evidence** for each customer, serving different roles in the explanation:

### Layer 1 — Indicator Trace (`rule_indicator_trace`)
A text string such as `"HT: HT-SEX-12(H), HT-SEX-07(H) | Struct: STRUCT-001(M)"`.

- Contains the **specific rule indicators** that fired, ranked by dynamic impact (`raw_rule_score × IF_score`)
- Truncated globally to the **top 5 indicators** across all typologies - as determined to fit the 2,000-character explanation limit
- This is the LLM's **primary evidence source** for citing specific indicator IDs (STRUCT-001, HT-SEX-12, etc.) in the narrative
- The trace format was designed to give the LLM natural language cues (`HT::HT-SEX-12(H)`) with severity codes (H/M/L) baked in

> Note this is first and foremost due to the character budget. Indicators ranked 6–10 exist in the underlying scores but are excluded from the trace.

### Layer 2 — Per-Typology Rule Scores (`rule_{typology}`)
Five float values in [0, 1], one per typology:
- `rule_structuring_layering`
- `rule_behavioural_profile`
- `rule_trade_shell`
- `rule_cross_border_geo`
- `rule_human_trafficking`

These are read **directly from `high_risk_evidence.csv`**, not derived from the trace.

> **These are not Isolation Forest scores.** `rule_{typology}` values come entirely from the deterministic rule engine in `03-hybrid-model.py` — weighted sums of hard FINTRAC/FinCEN threshold checks, clipped to [0, 1]. The ML-derived equivalents are the `if_score_{typology}` columns. The two are combined into the dynamic score (`if_score × rule_score` per typology), which is what ranks the top-5 trace indicators, but they remain distinct signals.

**Why not derive from the trace?**  
Because the global top-5 cut can eliminate an entire typology's indicators if they rank below 5. A customer could have `rule_structuring_layering = 0.45` (meaningful activity) but zero structuring indicators in the trace (all beaten out by higher-impact HT or GEO signals). If the LLM only sees the trace-derived count, it concludes structuring is silent — which is wrong. 

The direct score columns answer a different question: **"Is this typology meaningfully active at all?"** — independent of whether any of its indicators happened to crack the top 5. This is what drives Rule 7 in the system prompt (secondary typology detection).

**The two layers are complementary:**
- Trace → what to cite in text (specific indicators with severity)
- Scores → whether a typology is active at all (breadth signal for co-occurrence detection)

---

## Evidence Fields Sent to the LLM

| Field | Source | Purpose |
|-------|--------|---------|
| `customer_id` | Evidence CSV | Identity anchor |
| `final_hybrid_score` | Evidence CSV | Overall risk magnitude |
| `hybrid_risk_category` | Evidence CSV | Band label (Very High / High) |
| `coverage` | Evidence CSV | Fraction of 5 typologies where rules fired (0=none, 1=all) |
| `primary_rule_typology` | Evidence CSV | Dominant typology by dynamic score argmax |
| `rules_triggered` | Evidence CSV | Count of typologies with any rule score > 0 |
| `rule_indicator_trace` | Evidence CSV | Top-5 indicators by dynamic impact (primary narrative source) |
| `rule_{typology}` × 5 | Evidence CSV (direct) | Per-typology rule scores for breadth detection |
| `if_score_max` | Evidence CSV | Strongest single anomaly signal |
| `if_score_{typology}` × 5 | Evidence CSV | Per-typology IF anomaly scores |
| `cluster_primary_typology` | Evidence CSV | Dominant typology of the customer's peer cluster |
| `cluster_risk_tier` | Evidence CSV | Cluster baseline risk (0.1 = safest, 1.0 = riskiest) |

---

## Coverage Semantics

`coverage` = fraction of the 5 rule typologies where at least one rule fired.

| Value | Meaning | Interpretation |
|-------|---------|----------------|
| `0.0` | No rules fired at all | Score is entirely anomaly-driven (Pillars 2 and 3 only) |
| `0.2–0.4` | 1–2 typologies with rules | Partially corroborated; anomaly signal carries significant weight |
| `0.6–0.8` | 3–4 typologies with rules | Well-corroborated across multiple domains |
| `1.0` | All 5 typologies fired | Complete multi-typology rule corroboration |

**System Prompt Rule 5:** If `coverage <= 0.40`, the LLM is instructed to include a mandatory verbatim caveat noting that the score is primarily anomaly-driven. This is the correct direction — **low** coverage means **less** rule corroboration, not "limited data."

> Note: `coverage` is not about transaction history availability. It is a signal-quality indicator for how much of the final score is rule-confirmed vs. purely anomaly-driven.

---

## System Prompt Design

The system prompt (`SYSTEM_PROMPT`) is embedded as a module-level constant and passed to the LLM as a role instruction. It has four major sections:

### `== INPUT FORMAT ==`
Defines every field the LLM will receive and its semantics. Critical: LLMs follow field descriptions literally, so any inaccuracy here propagates into every explanation produced.

### `== AML INDICATOR REFERENCE ==`
A curated table of 36 indicator IDs across 5 typologies, with short descriptions and risk levels. The LLM is explicitly instructed:
- Only cite indicators genuinely supported by the evidence
- Do not invent indicator IDs
- Do not cite IDs absent from this table

This prevents hallucinated indicator references (e.g., the LLM inventing `STRUCT-042` because it sounds plausible).

### `== OUTPUT FORMAT ==`
Specifies CSV output with exactly two columns: `customer_id` and `explanation`. Double-quotes wrap the explanation. Internal quotes are escaped as `""`. No commentary before or after rows. This strict format enables deterministic `csv.reader` parsing on the Python side.

### `== EXPLANATION RULES ==`
Ten behavioural constraints governing how each explanation is written:

| Rule | Purpose |
|------|---------|
| 1 — Lead with dominant concern | Anchors the narrative to the primary typology |
| 2 — Cite indicators by ID | Grounds text in auditable evidence |
| 3 — Translate scores to language | Prevents raw decimals leaking into analyst-facing output |
| 4 — Use peer context | Incorporates cluster information (alignment or divergence) |
| 5 — Flag low coverage | Mandatory caveat when score is primarily anomaly-driven |
| 6 — No speculation | Maintains defensible language ("suggests", "warrants investigation") |
| 7 — Mention co-occurring typologies | Surfaces multi-scheme profiles using the per-typology scores |
| 8 — Never expose internals | Prevents field names, cluster IDs, model names appearing in output |
| 9 — Anomaly-only language | When rules_triggered = 0, explanation must reflect this honestly |
| 10 — Null handling | Missing fields are silently omitted, never fabricated |

---

## Batch Processing Architecture

Processing is structured in three loops:

```
evidence_df (up to MAX_CUSTOMERS_PER_RUN = 600, sorted by score descending)
    └── batches of LLM_BATCH_SIZE = 300 customers each
            └── single JSON array → single LLM call → single CSV response
```

Each batch sends a JSON array of records (built by `build_input_record()`) and expects back a CSV with exactly one row per customer. The LLM processes all 300 customers in a single call rather than one call per customer — this is a deliberate design choice for:
- **Cost efficiency**: One call per batch vs. 300 calls
- **Coherence**: The model sees the full batch distribution, helping it calibrate relative language ("the strongest signal in this batch")
- **Rate limiting**: Gemini has per-minute request limits

After each batch, a local checkpoint file is written so a partial run can be inspected if the process fails mid-way.

---

## LLM I/O Parsing

### Input Construction (`build_input_record`)
Converts a pandas `Series` row into a dict with normalized typed values (`None` for NaN, `int`/`float` for numeric, `str` otherwise). The dict is then serialized as JSON.

### Output Parsing — Strict Path (`parse_llm_csv`)
Expects the model to return a perfectly-formed CSV. Validates:
1. Header is `["customer_id", "explanation"]`
2. Exactly `len(expected_ids)` data rows
3. The set of returned customer IDs exactly matches the request batch

If any check fails, raises `ValueError` and falls through to the partial parser.

### Output Parsing — Partial Path (`parse_llm_csv_partial`)
Tolerates malformed or incomplete responses. Accepts any rows that parse correctly and match an expected customer ID. Fills unmatched customers with `"unsuccessful"`. Used as the fallback when the strict parser raises.

### Output Rendering (`render_output_csv`)
Always returns a valid CSV string with proper double-quote escaping, `\r` and `\n` stripped from explanation text. Used for both the final upload and intermediate checkpoints.

---

## Backend Support

| Backend | Model | Config |
|---------|-------|--------|
| `gemini` (default) | `gemini-2.5-flash` | `temperature=0.2`, 3 retries with 429 backoff |
| `local_llm` | GGUF (default: Llama-3.2-3B-Q4_K_M) | `n_ctx=4096`, `n_batch=256`, CPU by default |

Backend is selected via `MODEL_BACKEND` env var. Gemini requires `GEMINI_API_KEY`. Local LLM auto-downloads the GGUF file from Hugging Face if not found locally.

The rate limiting pause (`SECONDS_BETWEEN_CALLS = 10`) only applies to the Gemini backend and is skipped on the last batch.

---

## Error Handling Summary

| Failure scenario | Behaviour |
|-----------------|-----------|
| LLM returns `[ERROR: ...]` | Batch logged as failed; customers get `"unsuccessful"` |
| LLM returns malformed CSV | Strict parser fails; partial parser recovers what it can |
| Missing evidence columns | `normalize_value(row.get(...))` returns `None`; LLM omits the field per Rule 10 |
| Gemini rate limit (429) | Regex extracts retry-after, sleeps, retries up to 3× |
| Empty evidence CSV | Writes a header-only output file, exits cleanly |
