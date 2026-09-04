# Capstone Report — Severe Decline Prioritization

- **Author:** Saman Ashfaq
- **Lane:** Decline Recovery
- **Repo:** https://github.com/samanashfaq05/flyrank-ml-internship
- **Date:** September 4, 2026

## 0. Abstract

This capstone investigates whether observable search performance, traffic, engagement, and content characteristics can help prioritize already-declining content records that are experiencing severe decline. The analysis uses the FlyRank ML Internship decline recovery dataset containing 56,253 anonymized content records across 46 clients. A Random Forest classifier was trained using a client-level grouped train/test split to predict severe decline, defined as an impression decrease of 60% or more between the previous and recent 30-day periods. On the held-out test clients, the Random Forest achieved Precision@50 of 1.000 compared with 0.720 for the transparent baseline, while the severe-decline base rate was 0.524. The output is intended as a decision-support tool that helps FlyRank editors prioritize which declining content records should be reviewed first for possible refresh or recovery action.

## 1. Problem framing

The decision supported by this project is which already-declining content records a FlyRank editor should investigate first. The unit of analysis is a content record.

The output is a probability-based priority score and ranked recommendation list. Higher-scoring records are placed earlier in the list so that an editor can focus attention on the records most likely to be experiencing severe decline.

The action supported is editorial investigation for possible content refresh or recovery. A false positive may result in unnecessary editor review, while a false negative may cause a genuinely severe decline to receive less immediate attention.

Data and machine learning are useful because multiple search performance, traffic, engagement, and content characteristics may interact in ways that are difficult to represent with a single manual rule.

An important dataset observation is that all 56,253 records were already labeled as declining. Therefore, this project does not attempt to predict general decline. Instead, it focuses on distinguishing **severe decline from less severe decline among already-declining records**.

## 2. Data safety

The analysis uses the FlyRank ML Internship dataset in the decline recovery lane. The dataset contains 56,253 anonymized records across 46 pseudonymous clients.

The target `severe_decline` was defined using observed impression change:

- Severe decline: impression change of 60% or more (`<= -60%`)
- Non-severe decline: decline less than 60%

This produced 26,093 severe-decline records and 30,160 non-severe-decline records, giving an overall severe-decline rate of approximately 0.464.

Identifier columns such as client, content, keyword, URL, and title hash IDs were deliberately excluded from model features. The client hash was used only for grouped splitting so that the same client could not appear in both training and test data.

Potential leakage fields were also excluded, including:

- `trend_direction`
- `trend_pct`
- `is_declining`
- Other outcome-derived or recommendation-style labels

The modeling features included observable content characteristics, search performance, traffic, engagement, and availability indicators. Nothing client-identifying was included in the `work/` outputs.

## 3. Baseline

A transparent rule-based baseline was created before training the machine learning model. The baseline assigned a score using observable signals related to recent performance:

1. Recent impression drop
2. Weak average search position
3. Meaningful previous visibility

The resulting baseline score ranged from 1 to 3. Records with higher scores were prioritized first.

This baseline provides a fair comparison because it uses the same held-out test split and the same Precision@50 evaluation metric as the Random Forest model.

On the test set:

- **Baseline Precision@50: 0.720**
- **Test severe-decline base rate: 0.524**

The baseline therefore performed better than selecting records randomly according to the overall test-set rate, while remaining transparent and easy to interpret.

## 4. Model / analysis

The main method used was a Random Forest Classifier.

Random Forest was selected because the severe-decline prioritization problem contains multiple numerical, categorical, traffic, engagement, and content-related signals that may interact in non-linear ways. The method can model these interactions without assuming a simple linear relationship and also provides feature importance estimates for interpretation.

The target was:

**`severe_decline = 1` when the impression change between the previous and recent 30-day periods was less than or equal to -60%; otherwise `0`.**

The model used 25 observable features, including:

- keyword character count
- keyword token count
- public URL character count
- public URL path depth
- content title character count
- content title token count
- impressions over 90 days
- clicks over 90 days
- summed search position over 90 days
- sessions over 90 days
- pageviews over 90 days
- AI sessions over 90 days
- scroll events over 90 days
- clicks in the last 30 days
- sessions in the last 30 days
- clicks in the previous 30 days
- sessions in the previous 30 days
- content age
- CTR over 90 days
- average position over 90 days
- AI traffic percentage
- scroll rate
- client_has_gsc
- client_has_ga4
- content_type

Identifier columns and leakage-prone outcome columns were intentionally excluded from the feature set.

## 5. Evaluation

Evaluation used a client-level grouped train/test split with a fixed random seed of 42 and a test size of 20%.

The split produced:

- Training rows: 48,219
- Test rows: 8,034
- Training clients: 36
- Test clients: 10
- Clients appearing in both sets: 0

The grouped split was chosen to provide a more honest evaluation. Content records from the same client were not allowed to appear in both training and testing, reducing the possibility that the model could benefit from client-specific patterns seen during training.

The main metric was Precision@50, which measures the proportion of truly severe-decline records among the top 50 prioritized recommendations.

### Results

| Method | Precision@50 |
|---|---:|
| Test base rate | 0.524 |
| Transparent baseline | 0.720 |
| Random Forest | 1.000 |

The observed model lift over the baseline was:

**1.000 - 0.720 = 0.280**

On this specific grouped holdout, all of the top 50 records selected by the Random Forest were severe-decline records. This should be interpreted as an observed result on this split rather than proof that the model will always achieve perfect Precision@50 on future data.

The main limitation of error analysis is that the top 50 predictions contained no false positives on this particular test split. Performance should therefore be checked on additional client groups or future data before operational use.

## 6. Interpretation

The Random Forest results suggest that observable search-performance and content characteristics can help distinguish more severe declines among records that are already declining.

The top feature importances included:

1. `impressions_90d`
2. `sum_position_90d`
3. `content_age_days`
4. `avg_position_90d`
5. `clicks_last_30d`
6. `public_url_char_count`
7. `content_title_char_count`
8. `keyword_char_count`
9. `ctr_90d`
10. `clicks_90d`

In plain terms, the model relied strongly on historical visibility, search position, content age, and recent click performance.

A notable finding is that the dataset contained only records already classified as declining. Because of this, predicting `is_declining` would not have been meaningful because the target had no class variation. The severe-decline target was therefore created to support a more useful prioritization question.

Feature importance indicates association with the model's ranking behavior, not causal evidence. For example, a feature being important does not prove that changing that feature will prevent or reverse a decline.

## 7. Recommendation

The model output is intended as a decision-support tool for content prioritization.

Records are ranked by their predicted probability of severe decline. The highest-ranked records should be reviewed first by a FlyRank editor for possible refresh or recovery action.

Priority levels are assigned as follows:

- **High priority:** predicted probability >= 0.80
- **Medium priority:** predicted probability >= 0.60 and < 0.80
- **Low priority:** predicted probability < 0.60

For the highest-ranked records, the recommended action is immediate editorial review for possible content refresh or recovery action.

The top recommendations showed very high model scores and substantial differences between previous and recent 30-day impressions. These recommendations should not be treated as automatic instructions to modify content. An editor should review the underlying content and performance context before taking action.

The model is most useful for helping editors decide **where to investigate first**. Confidence reflects the model's ranking score rather than certainty that a specific editorial intervention will improve performance.

## 8. Reproducibility

The analysis was developed using Python with a fixed random seed of 42.

Environment highlights:

- Python: 3.13.15
- Pandas: 2.2.3
- NumPy: 2.1.3
- Scikit-learn: 1.6.1

Dataset:

- FlyRank ML Internship dataset
- Lane: decline_recovery

Main method:

- Random Forest Classifier

Evaluation design:

- Client-level grouped train/test split
- Test size: 20%
- Random seed: 42
- Main metric: Precision@50

To reproduce the work from a fresh clone:

```bash
git clone https://github.com/samanashfaq05/flyrank-ml-internship.git
cd flyrank-ml-internship
