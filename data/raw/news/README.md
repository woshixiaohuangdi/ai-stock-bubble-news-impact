## News Data

**Sources:** GDELT Project, The Guardian API, Yahoo Finance  
**Period:** 2021 ~ 2026

### Files

| File | Rows | Source |
|------|------|--------|
| `gdelt_news_v1.csv` | 21,902 | GDELT |
| `gdelt_news_checkpoint.csv` | 16,676 | GDELT |
| `yahoo_finance_news.csv` | 140 | Yahoo Finance |

> ATTENTION: `guardian_news.csv` (109MB) exceeds GitHub limit and is excluded.

### Fields

| Column | Description |
|--------|-------------|
| date | Publication datetime |
| title | Article headline |
| url | Source URL |
| source | Media outlet |
| language | Language |
| query | Search keyword |

### To Be Added

- [ ] Chinese financial news (Sina Finance / THUCNews)
- [ ] FOMC statements full text
- [ ] Expanded GDELT coverage post-2024
- [ ] Sentiment scores (FinBERT) → see `data/processed/`
