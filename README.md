# Wi-Fi-Deployment-Planning-Using-a-Multi-Metric-Decision-Layer-Function

## Overview
Traditional Wi-Fi deployment planning relies on simple heuristics — like capping the number of users per Access Point (AP) — which fail to capture the real, multi-dimensional nature of network congestion. 
This project proposes an alternative: a Network Health Index (NHI), a composite metric that combines five key performance indicators into a single, interpretable score.

The system simulates an IEEE 802.11 Wi-Fi infrastructure network in NetSim under increasing load (2 to 20 nodes transmitting CBR traffic), extracts performance metrics, and processes them in Python to classify the network's health as:

🟢 Suitable (Optimal)
🟡 Risky (Degraded)
🔴 Unsuitable (Failure)

## Key Features
* Multi-metric evaluation across throughput, delay, packet loss, collision rate, and backoff failure rate
* Dynamic weighting — metric importance adapts based on current network conditions rather than using static weights
* Min-max normalization with epsilon handling to avoid divide-by-zero and score collapse
* Adaptive thresholding using quantile-based cutoffs (70th/40th percentile) instead of fixed thresholds
* Critical point detection — automatically identifies the node count at which the network becomes unsuitable
* Visualization suite — NHI vs. nodes, metric trends, weighted contribution breakdown, weight evolution, and decision zone plots
  
## Theoretical Background
Why Wi-Fi Networks Degrade Under Load
IEEE 802.11 networks use a MAC-layer protocol called CSMA/CA (Carrier Sense Multiple Access with Collision Avoidance) to arbitrate shared-medium access. Since wireless nodes cannot detect collisions while transmitting (unlike wired Ethernet's CSMA/CD), 802.11 relies on collision avoidance rather than detection:

* Before transmitting, a node senses the channel and waits if it's busy: If the channel is free, the node waits a random backoff interval before transmitting, reducing the chance two nodes transmit simultaneously.
* If a collision does occur, the sender doesn't receive an ACK, assumes a collision happened, and doubles its contention window (binary exponential backoff) before retrying

This mechanism, formally analyzed in Bianchi's Distributed Coordination Function (DCF) model (Bianchi, 2000), is efficient at low node density but degrades non-linearly as more nodes contend for the same AP. As node count rises:

* The probability of simultaneous transmission attempts increases → collision rate rises
* Nodes spend more time backing off and retrying → delay and backoff failure rate rise
* Failed/retried packets consume airtime without delivering data → effective throughput falls
* Retry limits are eventually exceeded → packets are dropped, raising packet loss

These five effects are coupled — they don't degrade independently — which is precisely why a single metric (e.g., just throughput) gives an incomplete picture of network health, and motivates a composite index approach.

## Limitations of Heuristic Deployment Planning

Conventional deployment planning imposes a static cap (e.g., "25 users per AP") derived from rule-of-thumb estimates or vendor airtime-utilization guidelines. This approach has two theoretical weaknesses:

* It is metric-blind. A fixed user cap doesn't account for traffic pattern, packet size, or the specific mix of delay/loss/collision behavior a network is actually experiencing — two networks with the same node count can have very different health depending on application demand.
* It is non-adaptive. The relative importance of throughput vs. delay vs. loss changes depending on how congested the network already is; a static rule can't reflect this.
  
The Network Health Index (NHI): Theoretical Formulation
The NHI is designed to address both weaknesses using a weighted, normalized composite scoring function — a common approach in multi-criteria decision analysis (MCDA).

* Step 1 — Normalization. Each raw metric is rescaled to a common [0,1] range using min-max normalization: X_norm = (X − X_min) / (X_max − X_min).
For metrics where lower is better (delay, packet loss, collision rate, backoff failure), the score is inverted: Score = 1 − X_norm

This ensures all five metrics are directionally consistent — a higher score always means better network health — before they're combined. A small epsilon term is added to the denominator to avoid division-by-zero when a metric is constant across all samples, and scores are clipped to a minimum of 0.05 to prevent a metric from collapsing to zero and being effectively erased from the composite.

* Step 2 — Dynamic Weighting. Rather than fixed weights, each metric's weight is computed as a function of how degraded that metric currently is relative to its worst observed value. For example, the packet loss weight grows as packet loss itself grows: w_PL = 0.2 + 0.2 × (PacketLoss / PacketLoss_max)

All weights are then normalized so they sum to 1. The intuition: as a network becomes more congested, the metrics driving that congestion should count for more in the final score — the model self-adjusts its own sensitivity rather than treating all five factors as equally important at all times.

* Step 3 — Composite Score. The final Network Health Index combines the weighted scores and rescales to a 0–10 range for interpretability: NHI = Σ(w_i × score_i) × 10

* Step 4 — Adaptive Decision Thresholds. Instead of hardcoded cutoffs, thresholds are derived from the quantiles of the observed NHI distribution itself (70th and 40th percentile), making the classification self-calibrating to the specific dataset rather than relying on arbitrary fixed numbers:

  * NHI > 70th percentile → Suitable
  * NHI > 40th percentile → Risky
  * Otherwise → Unsuitable

This is conceptually similar to statistical process control, where thresholds are derived from a system's own behavior rather than imposed externally.

## Why This Matters

The theoretical contribution isn't the individual metrics (which are standard 802.11 performance indicators) — it's the decision layer on top of them: a reproducible, quantitative way to convert five interacting, sometimes contradictory signals into a single actionable classification, without relying on a network engineer's subjective judgment call.

## Methodology
* Configure and run Wi-Fi simulations in NetSim for varying node counts (2, 4, 6, 10, 14, 20)
* Collect performance metrics: throughput, delay, packet loss, collision rate, backoff failure rate
* Normalize each metric to a common [0.05, 1] scale (inverted for "lower is better" metrics)
* Compute dynamic weights per metric based on relative performance degradation
* Calculate the composite NHI score: NHI = Σ(weight_i × score_i) × 10
* Classify network health against adaptive quantile-based thresholds
* Visualize results and compare against traditional heuristic-based methods

## Tech Stack
NetSim — network topology design and simulation (AP, wireless nodes, switch, wired backbone, CBR traffic)
Python 3
pandas — data handling
numpy — numerical computation
matplotlib — visualization
