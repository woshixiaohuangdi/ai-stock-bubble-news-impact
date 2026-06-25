# Direction Prediction: AI Stock Next-Day Return Forecasting

This module predicts the **next-day close-to-close return direction** (up or down) of an
equal-weight AI stock portfolio using Bloomberg news sentiment and an expert committee framework.

**Stocks:** NVDA, MSFT, META, AMZN, GOOGL  
**News source:** Bloomberg (2021–2025)  
**Price data:** Yahoo Finance daily OHLCV  
**Train period:** 2021–2024  
**Out-of-sample test window:** 2025 (strict holdout, no look-ahead)

---

## Pipeline Overview

```
Bloomberg News (Title + Summary)
        │
        ├──► BART zero-shot classification ──► Narrative category score per article
        │                                       (chip / power / algorithm / regulation / earnings)
        │
        └──► FinBERT sentiment scoring ──► Score = P(positive) − P(negative)
                                           Confidence = max(P(positive), P(negative))
                        │
                        ▼
        Filter: BART score ≥ 0.6 AND FinBERT confidence ≥ 0.6
                        │
                        ▼
        Daily sentiment aggregation per (ticker, category)
                        │
                        ▼
        Local Projections (LP) — estimated on 2021–2024 training period
                        │
                        ▼
        Expert committee weighted vote ──► Portfolio direction prediction
```

---

## Models

### BART — Narrative Classification

**Model:** `facebook/bart-large-mnli`  
**Task:** Zero-shot multi-label classification  
**Reference:** Lewis et al. (2020), NeurIPS

BART is a sequence-to-sequence model fine-tuned on the Multi-Genre Natural Language Inference
(MNLI) corpus. Category assignment is framed as a natural language inference task: given an
article and a candidate label description, BART estimates the probability that the article
entails the label. No task-specific training data is required, making it suitable for
domain-specific taxonomies such as AI narrative categories.

**Input:** Concatenated title and summary (truncated to 512 tokens)  
**Output:** Probability score for each of the five narrative categories  
**Threshold:** A category is assigned if its BART score ≥ 0.6. An article may receive
multiple categories simultaneously (multi-label).

**Five narrative categories and their label descriptions:**

| Category | Label description used for inference |
|----------|--------------------------------------|
| `algorithm` | artificial intelligence algorithm, large language model, generative AI, foundation model, transformer architecture, AI model training and research |
| `chip` | chip and GPU hardware, semiconductor manufacturing, processor supply chain, silicon wafer fabrication |
| `power` | data center power consumption, electricity demand for computing, energy infrastructure for AI servers, cooling capacity |
| `regulation` | government regulation, export ban, antitrust lawsuit, trade sanction, congressional policy, court ruling, geopolitical restriction |
| `earnings` | quarterly earnings report, revenue results, profit guidance, EPS forecast, financial performance, analyst estimate |

---

### FinBERT — Sentiment Scoring

**Model:** `ProsusAI/finbert`  
**Task:** Financial sentiment classification  
**Reference:** Araci (2019); Devlin et al. (2018)

FinBERT is a BERT-based model fine-tuned on a large corpus of financial news and disclosures.
It outputs three class probabilities: positive, negative, and neutral.

**Input:** Concatenated title and summary (truncated to 512 tokens)  
**Sentiment score:** `P(positive) − P(negative)` ∈ [−1, 1]  
**Confidence:** `max(P(positive), P(negative))`  
**Filter:** Articles with confidence < 0.6 are excluded from daily aggregation.

---

## Methodology

### Data Preparation

Bloomberg news articles are matched to trading days by date. Weekend articles
(Saturday and Sunday) are shifted to the preceding Friday. The prediction target
`Direction_tomorrow` is the sign of the next trading day's close-to-close return
(1 = up, 0 = down).

### Daily Sentiment Aggregation

For each (ticker, category) pair, the daily sentiment signal is the **sum of FinBERT
scores** across all articles passing both filters: BART score ≥ 0.6 for the category
and FinBERT confidence ≥ 0.6.

### Local Projections (LP)

For each (ticker, category) pair, LP regresses next-day return on daily sentiment,
controlling for the current day's return:

```
next_return_t = α + β · sentiment_t + γ · return_t + ε_t
```

Estimated via OLS on the 2021–2024 training period. The sign of β determines the
prediction direction. Only pairs with statistically significant β (p < 0.1) are
retained in the committee.

### Expert Committee Voting

Three aggregation layers:

1. **Article level:** FinBERT scores are summed per (ticker, category, day) after filtering.
2. **Ticker level:** Category-level predictions within each ticker are aggregated by
   accuracy weights into a single ticker vote.
3. **Portfolio level:** Ticker votes are aggregated by weighted majority voting.
   Days with no active signal are excluded from evaluation (treated as no-signal days).

---

## Three-Method Comparison

All methods share the same expert committee architecture and 2025 out-of-sample test window.
The ablation isolates the contribution of each component.

| Method | Text Representation | Direction Accuracy | Baseline | Significant |
|--------|--------------------|--------------------|----------|-------------|
| Method 1 | Loughran-McDonald dictionary | < baseline | ~53% | ✗ |
| Method 2 | TF-IDF + LinearSVC | 51.7% | 51.7% | ✗ |
| **Method 3** | **FinBERT + LP** | **see below** | **52.9%** | **✓** |

The comparison confirms that **text representation quality** (FinBERT over TF-IDF) and
**econometric estimation** (LP over raw scores) are the two primary drivers of predictive
performance, not the committee architecture itself.

---

## Results (Method 3, 2025 Out-of-Sample)

Two committee configurations were evaluated, differing only in which tickers are included.

### 3-Stock Committee (NVDA + MSFT + META)

| Metric | Value |
|--------|-------|
| Signal days | 141 / 249 trading days |
| Directional accuracy | **61.7%** |
| Baseline (up-day rate on signal days) | 51.1% |
| Bootstrap 90% CI (5,000 resamples) | [54.6%, 68.8%] |
| Statistically significant | **Yes** |
| Large-move day accuracy | **67.6%** (37 days) |

### 5-Stock Committee (NVDA + MSFT + META + AMZN + GOOGL)

| Metric | Value |
|--------|-------|
| Signal days | 167 / 249 trading days |
| Directional accuracy | **58.7%** |
| Baseline (up-day rate on signal days) | 50.3% |
| Bootstrap 90% CI (5,000 resamples) | [52.7%, 64.7%] |
| Statistically significant | **Yes** |
| Large-move day accuracy | 59.6% |

The 3-stock committee achieves higher per-signal accuracy by concentrating on the
ticker-category pairs where sentiment has the strongest and most consistent LP coefficients.
The 5-stock committee trades some accuracy for broader daily coverage (67% vs ~45% of
trading days).

---

## Why Two Committee Configurations?

The 3-stock and 5-stock committees are not two separate models — they reflect different
levels of signal filtering within the same framework.

The expert committee retains only ticker-category pairs where the LP coefficient β is
statistically significant (p < 0.1) on the 2021–2024 training data. When this filter is
applied, NVDA, MSFT, and META consistently contribute reliable signals across multiple
narrative categories. AMZN and GOOGL pass the significance threshold on fewer pairs,
meaning their inclusion adds trading day coverage but dilutes per-signal accuracy.

The 3-stock committee prioritises **accuracy** (61.7%, signal on 57% of trading days).
The 5-stock committee prioritises **coverage** (58.7%, signal on 67% of trading days).
Both are statistically significant. Which configuration is preferable depends on the
intended use: the 3-stock version is better suited to high-conviction applications, while
the 5-stock version provides more consistent daily signal generation.

---

## Comparison with GPT-4 Based Approaches

The closest comparable study is Lopez-Lira & Tang (2025, *Journal of Financial Economics*),
which uses GPT-4 to classify news headlines and predict stock price movements.

| Dimension | Lopez-Lira & Tang (2025) | This Study |
|-----------|--------------------------|------------|
| Model | GPT-4 (proprietary, closed) | FinBERT + BART (open-source) |
| Prediction target | **Initial reaction** (open-to-open, ~30 min after news) | **Next-day close-to-close return** |
| Hit rate reported | ~90–93% (initial reaction) | 61.7% (3-stock) / 58.7% (5-stock) |
| Tradability | Essentially non-tradable for most investors | Tradable at market close |
| Training required | Zero-shot, no fine-tuning | LP estimated on 2021–2024 |
| News universe | Broad US equities (4,123 stocks) | AI/tech sector focus (5 stocks) |
| Narrative structure | Topic modelling post-hoc | Five categories defined a priori |
| Monetary policy context | Not modelled | Three FOMC phases annotated |

**Key distinction:** Lopez-Lira & Tang's high hit rate captures the market's **immediate
open-to-open price adjustment** after overnight news — a window that is largely
non-tradable for most market participants. This study targets the **subsequent
close-to-close drift**, a strictly harder prediction problem because the market has
already partially priced in the news by open. Our results are therefore not directly
comparable in magnitude, but address a more practically relevant forecasting horizon.

A further distinction is interpretability: our five narrative categories (chip, power,
algorithm, regulation, earnings) are defined a priori from domain knowledge, enabling
direct analysis of which types of AI news drive return predictability. Lopez-Lira & Tang
derive topics post-hoc via topic modelling.

---

## Limitations and Forward-Looking Notes

The results reported here are based on a single out-of-sample year (2025) and five
AI/tech stocks. Several factors may affect the stability of the framework over time:

**Market regime dependence.** The LP coefficients are estimated on 2021–2024 data,
a period spanning three distinct monetary policy phases (accommodation, tightening,
and easing). The predictive relationship between news sentiment and returns may shift
as macroeconomic conditions change. Regular re-estimation of LP coefficients on a
rolling window is advisable in any live application.

**Narrative relevance may evolve.** The five narrative categories (chip, power,
algorithm, regulation, earnings) were defined based on the dominant AI news themes
of 2021–2025. As the AI sector matures, new narrative types — such as AI safety
regulation, sovereign AI investment, or inference-cost competition — may emerge and
require category updates.

**Signal sparsity.** The dual filtering (BART ≥ 0.6 and FinBERT confidence ≥ 0.6)
ensures high-quality signals but limits coverage to roughly 57–67% of trading days.
On no-signal days the framework makes no prediction, which is the correct behaviour
but limits practical utility.

**Stock universe.** The committee composition (which tickers contribute reliable
signals) is likely to change as the AI stock landscape evolves. The current 3-stock
core (NVDA, MSFT, META) reflects the signal structure of the 2021–2024 training
period and should be re-evaluated periodically.

Despite these limitations, the core finding is robust: **Bloomberg news sentiment,
decomposed by AI narrative category and processed through domain-adapted language
models, contains statistically significant directional information about next-day
AI stock returns.** The method is sound and the framework is designed to be
re-estimated and updated as new data becomes available.

---

## References

- Araci, D. (2019). FinBERT: Financial Sentiment Analysis with Pre-trained Language Models. *arXiv:1908.10063*.
- Jordà, Ò. (2005). Estimation and Inference of Impulse Responses by Local Projections. *American Economic Review*, 95(1), 161–182.
- Lewis, M., Liu, Y., Goyal, N., Ghazvininejad, M., Mohamed, A., Levy, O., Stoyanov, V., & Zettlemoyer, L. (2020). BART: Denoising Sequence-to-Sequence Pre-training for Natural Language Generation, Translation, and Comprehension. *ACL 2020*.
- Lopez-Lira, A. & Tang, Y. (2025). Can ChatGPT Forecast Stock Price Movements? Return Predictability and Large Language Models. *Journal of Financial Economics*, forthcoming.
