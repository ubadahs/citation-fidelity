# Reranker Model Evaluation — 2026-04-07

The evidence retrieval stage uses BM25 as a first pass over the cited paper's paragraph blocks, then an LLM reranker to semantically re-score and extract the key sentences a human reviewer would need. The reranker replaced a paragraph-start truncation approach that was producing false partial verdicts — the right section would surface but the decisive sentence would be cut off at a 600-character limit.

This evaluation compared six model configurations for the reranker to answer two questions: (1) does a more expensive reranker produce meaningfully better verdicts, and (2) does extended thinking help? We ran all six in parallel against the same extraction and classification artifacts, with an identical Opus adjudicator, then graded 7 of the 13 resulting tasks against domain-expert ground truth.

The short answer: **Haiku with thinking matches every larger model on accuracy at a fraction of the cost.** Plain Haiku without thinking is measurably worse — it misses evidence that all other configurations find — but the $0.14 thinking premium eliminates the gap entirely. There is no accuracy case for Sonnet or Opus as the reranker at current sample size.

Raw run artifacts live in `data/eval-reranker-20260407-004349/` (gitignored, local only).

## Purpose

Compare six LLM reranker configurations to determine the best cost/quality tradeoff for evidence reranking in the citation fidelity pipeline.

## Setup

- **Seed paper:** Sabbagh et al. (DOI: `10.1111/jnc.15101`) — "Diverse GABAergic neurons organize into subtype-specific sublaminae in the ventral lateral geniculate nucleus"
- **Tracked claim:** "the vLGN has at least six distinct GABAergic neuronal subtypes"
- **Shared stages:** screen → extract → classify (run once, reused across all variants)
- **Divergent stages:** evidence → curate → adjudicate (6× parallel, each with a different reranker)
- **Adjudication:** claude-opus-4-6 with extended thinking (identical across all variants)
- **Audit sample size:** 50 (13 tasks survived extraction and classification)
- **Ground truth:** 7 of 13 tasks graded by domain expert

### Variants

| Label | Reranker model | Thinking | Rerank cost | Total cost |
|---|---|---|---|---|
| haiku | claude-haiku-4-5 | off | $0.18 | $0.44 |
| haiku-thinking | claude-haiku-4-5 | on | $0.32 | $0.58 |
| sonnet | claude-sonnet-4-6 | off | $0.54 | $0.80 |
| sonnet-thinking | claude-sonnet-4-6 | on | $0.77 | $1.03 |
| opus | claude-opus-4-6 | off | $0.91 | $1.17 |
| opus-thinking | claude-opus-4-6 | on | $1.18 | $1.42 |

## Results

### Verdict Distribution

| Variant | Supported | Partial | Not Supported | Cannot Det. |
|---|---|---|---|---|
| haiku | 9 | 3 | 1 | 0 |
| haiku-thinking | 11 | 2 | 0 | 0 |
| sonnet | 11 | 2 | 0 | 0 |
| sonnet-thinking | 11 | 2 | 0 | 0 |
| opus | 11 | 2 | 0 | 0 |
| opus-thinking | 11 | 2 | 0 | 0 |

### Retrieval Quality (judge's assessment)

| Variant | High | Medium | Low |
|---|---|---|---|
| haiku | 9 | 4 | 0 |
| haiku-thinking | 12 | 1 | 0 |
| sonnet | 12 | 1 | 0 |
| sonnet-thinking | 13 | 0 | 0 |
| opus | 11 | 2 | 0 |
| opus-thinking | 13 | 0 | 0 |

### Ground-Truth Accuracy (7 expert-graded tasks)

| Variant | Correct | Wrong | Accuracy |
|---|---|---|---|
| haiku | 4-5 | 2 | 57-71% |
| haiku-thinking | 7 | 0 | **100%** |
| sonnet | 7 | 0 | **100%** |
| sonnet-thinking | 7 | 0 | **100%** |
| opus | 7 | 0 | **100%** |
| opus-thinking | 7 | 0 | **100%** |

### Per-Task Stability (13 tasks × 6 variants)

- **10 of 13 tasks:** Identical verdicts across all 6 variants (STABLE)
- **3 split tasks:** All splits were haiku-without-thinking deviating from the other 5 variants

### Split Case Analysis

| Task | haiku | All others | Root cause |
|---|---|---|---|
| Avertin protocol (methods_use) | not_supported | partially_supported | Haiku retrieved isoflurane paragraph, missed the Avertin methods section |
| Brn3c / Pvalb marker (methods_use) | partially_supported | supported | Haiku hedged on Pvalb as TRN marker; others recognized the paper as a legitimate source |
| Circadian / Calb1+Pvalb (ambiguous) | partially_supported | supported | Haiku missed the Calbindin evidence paragraph |

In all 3 cases, haiku-without-thinking's errors trace to the reranker (retrieval quality), not the adjudicator (reasoning). When any model gets the right evidence, it reasons correctly.

## Decision

**Default reranker: `claude-haiku-4-5` with extended thinking enabled.**

Rationale:
- Matches Sonnet/Opus on verdict accuracy (100% on graded tasks) at 40-70% lower cost
- Thinking compensates for Haiku's weaker semantic matching in the reranker stage
- The $0.14 thinking premium over plain Haiku buys 2 correct verdict flips
- No accuracy gain from Sonnet or Opus justifies the 2-3× cost increase at this sample size

### Cost comparison at chosen config

| Stage | Model | Cost per run |
|---|---|---|
| Seed grounding | claude-opus-4-6 | ~$0.22 |
| Evidence reranking | claude-haiku-4-5 + thinking | ~$0.32 |
| Adjudication | claude-opus-4-6 + thinking | ~$0.25 |
| **Total** | | **~$0.79** |

## Ground-Truth Calibration Notes

Expert feedback on grading philosophy (from domain scientist review):

1. **Conservative grading preferred.** Even if a compression is "acceptable scientific shorthand," flag it as `partially_supported` because detecting latent bias from shorthand is the project's purpose.
2. **Indirect method attribution** (citing intermediate protocol user instead of originator) is bad practice and should be `partially_supported`.
3. **Self-citations** from the same lab should be held to the same fidelity standard as external citations.
4. **Dual-role citations** (e.g., methods + anatomical reference) are acceptable as classified — `supported` is correct when the cited paper genuinely provides the relevant characterization.
5. **"Preferentially distributed" → "residing in"** is a meaningful compression, not acceptable shorthand — correctly flagged as `partially_supported`.

## Artifacts

All variant outputs are in subdirectories of this eval:
- `shared/` — screen, extract, classify (reused)
- `haiku/`, `haiku-thinking/`, `sonnet/`, `sonnet-thinking/`, `opus/`, `opus-thinking/` — each contains `04-evidence/`, `05-curate/`, `06-adjudicate/` and a `run.log`
