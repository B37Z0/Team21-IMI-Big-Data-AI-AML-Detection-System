# AML Hybrid Model Architecture

The essence of this detection pipeline is that neither machine learning nor deterministic rules are sufficient on their own - but combining them is difficult. Rule-based systems lack granularity and can be fairly easily bypassed by sophisticated bad actors. Conversely, pure unsupervised Machine Learning (in our case, Isolation Forests) flags anomalous behavior, but has a high false-positive rate because "unusual" doesn't necessarily mean "illegal".

The Hybrid Model resolves this by fusing both approaches (plus a little extra) into a **3-Pillar Ensemble**.

---

## Pillar 1: The Dynamic Score (Corroborated Risk)
**Weight: 50%**

This pillar broadly catches bad actors. It requires both the rule-based system and the ML model to "agree" on the suspiciousness of a customer (i.e. we multiply the scores).

For each of the five illicit typologies (Human Trafficking, Trade Based Shell, etc.), the pipeline calculates a **Dynamic Multiplier**:
```text
Dynamic Score = Rule Engine Score × Isolation Forest Probability
```
*   If a customer breaks a FINTRAC regulatory threshold, their rule score jumps.
*   If the Isolation Forest also determines their behavior shape matches an anomaly, their IF probability jumps.
*   Because the two are multiplied together, a customer would trip *both* the deterministic rule and the probabilistic ML boundary anomaly to score highly here.

This combination is greater than the sum of its parts. The final Dynamic Pillar is the normalized sum of these 5 multipliers.

---

## Pillar 2: Coverage-Based Fallback ('Zero-Day' Anomalies)
**Weight: 20%**

Dynamic scoring on its own is flawed (we found this out the hard way) - it is entirely dependent on the rule system. If no rules are triggered, the dynamic score is 0, even if the Isolation Forest detects highly anomalous behavior. Hence, this pillar should catch hypothetically sophisticated bad actors that evade all internal rule thresholds but behave highly suspiciously. In less flattering terms, it is a fallback. 

When a customer evades all rules for a given typology, their dynamic score is `0`. To offset this, we calculate `coverage`—the fraction of the 5 rule typologies that triggered. We then build the fallback equation:
```text
Fallback Score = (1.0 - coverage) × IF_Score_Weighted
```
*   **Coverage = 1.0 (All rules fired):** The IF score contributes `0` points here because it is already working perfectly as a multiplier in Pillar 1.
*   **Coverage = 0.0 (No rules fired):** The rules are blind to this customer. The equation pivots and gives 100% of the scoring weight to the Isolation Forest anomaly detection, acting as an unsanctioned net to catch zero-day typologies.

---

## Pillar 3: KMeans Behavioral Peer-Clustering
**Weight: 30%**

This pillar provides "relative severity". KMeans allows us to group similar customers into behavioral cohorts, determining how severely anomalous a customer is compared strictly to people who act like them. IF score distributions are heavily right-skewed across 61K customers — most customers score near zero with a long anomalous tail. Feeding raw scores into K-means would cause cluster geometry to degenerate into one large low-risk blob. RobustScaler flattens this distribution so K-means finds meaningful substructure across the full population rather than just identifying extreme outliers

### 1. Robust Scaled Feature Space
We feed all 61,000 customers into the KMeans algorithm using *only* their 5 pure Isolation Forest anomaly probabilities. Because anomalies are highly left-skewed, we need to aggressively flatten them using a `RobustScaler()` (using the Interquartile Range rather than strict variance). This prevents ultra-anomalies from dragging cluster centroids artificially outward.

### 2. The Thin-Cluster Refit Loop
KMeans begins by slicing the 61,000 customers into `K=8` behavioral archetypes. We must also institute a strict statistical safeguard: a percentile rank is useless if the sample size is insignificant. If any cluster contains fewer than `200` members, we reject the fit and dynamically drop `K` to `7`, and refit. This should iterate downwards until stable, macro-level cohorts are achieved.

### 3. Within-Cluster Percentile (`kmeans_score`)
Once clustered, the pipeline evaluates each customer against the rest of their cluster members using their average Isolation Forest anomaly score (`if_score_mean`).
If the customer has a more extreme anomaly score than 95% of their peers, their `kmeans_score` becomes `0.950`. 
By ranking purely on IF scores instead of Rule scores, Pillar 3 maintains total independence from the deterministic rule engine.

### 4. Risk-Tier Adjustment (`adjusted_kmeans_score`)

A pure within-cluster percentile is dangerous in isolation: being the 99th percentile inside a cluster of 50,000 legitimate retail users means something very different than being the 99th percentile in a suspected human-trafficking ring. To prevent false positives from inflating the score, the percentile is scaled by the cluster's base risk tier.

The cluster risk tier is calculated by taking the mean of all IF scores within the cluster, and min-max normalizing it across the *K* active clusters to a `[0.1, 1.0]` scale:
```text
cluster_risk_tier = 0.1 + 0.9 × Normalize(mean(if_score_mean across cluster))
```

By substituting the rules-based dynamic score out for pure IF scores, we can sever any circular dependency with Pillar 1. The `0.1` floor is added to ensure the safest cluster isn't entirely blinded. Should a bad actor land in a "normal" cluster, their within-cluster anomalies will still register at a maximum of 10% severity, rather than zeroing out.

```text
adjusted_kmeans_score = kmeans_score × cluster_risk_tier
```

---

## The Final Ensemble Equation

The final fusion combines all three pillars into a single `0.0` to `1.0` continuous feature.

> **Hybrid Risk =** 
> `0.50` × (Corroborated Dynamic Mix) + 
> `0.20` × (Zero-Day IF Fallback) + 
> `0.30` × (Adjusted Peer-Group Percentile)

This ensures that the top 1% of flagged customers have either tripped hard FINTRAC rules while behaving anomalously, circumvented rules while behaving irregularly, or are the single worst offending entity inside an already-suspicious network.

### Normalization & Hierarchy
The raw weights (`0.50`, `0.20`, `0.30`) do not dictate the absolute ceiling of the final score. After the raw sums are calculated, the entire population distribution is min-max normalized from 0.0 to 1.0. 

The weights have been structured primarily to ensure regulatory compliance, and reflect to our best ability the relative impartial importance of each pillar:
* **Regulatory Primacy:** Basic FINTRAC/FinCEN compliance should take precedence. If Customer A breaks explicit $10k reporting thresholds, the model should score a non-definite ML anomaly higher than them. The dynamic score is weighed (`0.50`) to ensure explicit rule-breakers loosely comprise a 'straightforward' half of the final score.
* **Throttling the Safety Net:** The fallback is necessary, but pure ML anomalies should have a high false-positive rate. By throttling the fallback to `0.20`, a 'maximum undetected anomaly' (Coverage=0) scores `0.50` raw points (`0.2 IF + 0.3 KMeans`). A 'maximum rule-breaker' (Coverage=1) scores `0.80` raw points (`0.5 Dynamic + 0.3 KMeans`). 
---