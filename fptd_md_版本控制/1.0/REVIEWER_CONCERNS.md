# REVIEWER_CONCERNS

This document analyzes the current project from a strict advisor / reviewer perspective. It is intentionally conservative and should not be read as a defense of the current approach. Its purpose is to identify weak points, unsupported claims, and experiments needed before making stronger robustness claims.

## 1. Is the Current Innovation Strong Enough?

The current strongest methodological idea is:

```text
worker-level collusion-aware filtering for categorical truth discovery
```

It is meaningful because it matches the observed Duck attack structure:

```text
attack: malicious workers coordinate false labels
defense: detect abnormal cross-task worker agreement
```

However, at the current stage, the innovation is not yet fully strong as a general robustness contribution.

Reasons:

- The main validated case is Duck, a binary categorical dataset.
- The attack is highly coordinated and relatively easy for agreement-based detection.
- The current detector uses hand-designed thresholds.
- The method has not yet been tested on noisy, partial, multi-target, or adaptive collusion.
- The numerical Weather branch is still preliminary.

Conservative assessment:

```text
The current project has a plausible core idea and initial evidence.
It is not yet a mature general robust truth discovery method.
```

The innovation may be strong enough for a focused contribution if the scope is narrowed to:

```text
robust preprocessing for highly coordinated categorical worker-level collusion
```

It is not yet strong enough to claim:

```text
general defense against malicious worker uploads
```

## 2. Which Modules May Be Heuristic?

Several modules or design choices are currently heuristic.

### Categorical Collusion Detection Thresholds

Current hard rules:

```text
pairwise agreement >= 0.98
common tasks >= 10
group size between 20% and 50% of workers
```

These thresholds are not theoretically derived in the current project. They are reasonable for the current Duck attack, but a reviewer may ask:

- Why 0.98?
- Why 20%-50%?
- What happens at 10% attack?
- What happens above 50% attack?
- Does this work on other categorical datasets?

Current status:

```text
heuristic, partially validated only on Duck high-consistency collusion
```

### Whole-Worker Filtering

Current categorical defense filters suspicious workers at the worker level. This is simple but coarse.

Potential issue:

```text
If a worker is only malicious on some objects, whole-worker filtering removes potentially useful observations too.
```

Current status:

```text
works for current full-row collusion attack
not proven for partial-task attacks
```

### MAD Coefficient for Numerical Data

Current default:

```text
madCoefficient = 3.0
```

Weather biased-noise results show:

```text
3.0: no recovery
1.0: partial recovery
0.5: near-clean recovery but aggressive filtering
```

This suggests the numerical branch is sensitive to the MAD threshold.

Current status:

```text
parameter-sensitive, needs systematic sweep
```

### Minimum Support Restoration

Minimum support is reasonable as a stability mechanism, but its effect is not fully isolated.

Potential issue:

```text
restoring observations based on distance to pilot truth may reintroduce malicious values if pilot truth is polluted
```

Current status:

```text
supporting mechanism, not yet a proven contribution
```

## 3. What Evidence Is Missing for Robustness Claims?

The current evidence supports only a narrow robustness claim.

Supported:

```text
Duck highly consistent categorical collusion at 30% and 40%
→ baseline degrades
→ ours recovers
```

Missing evidence:

### Noisy Collusion

Current attack is too consistent. Need:

```text
malicious workers coordinate most of the time but inject random label noise
```

Purpose:

```text
test whether agreement-based detection survives imperfect coordination
```

### Partial Collusion

Need:

```text
malicious workers attack only a subset of objects
```

Purpose:

```text
test whether whole-worker filtering is too coarse or ineffective
```

### Threshold Sensitivity

Need:

```text
agreement threshold sweep
group-size threshold sweep
```

Purpose:

```text
show the method is not tuned to a single parameter setting
```

### Clean False-Positive Analysis

Need:

```text
run defense on clean data under different thresholds
```

Purpose:

```text
show that defense does not remove honest users in normal data
```

### Additional Categorical Dataset

Need:

```text
Dog / Face / Product if compatible
```

Purpose:

```text
show the categorical defense is not Duck-specific
```

### Numerical Robustness

Weather branch needs:

```text
MAD coefficient sweep
attack strength sweep
attacker ratio sweep
clean side-effect analysis
```

Purpose:

```text
validate numerical robustness beyond one strict threshold result
```

### Ablation

Need focused ablation:

```text
without collusion filtering
without robust bootstrap
without minimum support
collusion filtering only
full pipeline
```

Purpose:

```text
identify which component causes recovery
```

## 4. Is the Threat Model Reasonable?

The threat model is reasonable but currently narrow.

Reasonable assumptions:

- The attacker controls a subset of workers.
- The attacker modifies uploaded observations.
- Servers and FPTD protocol are not compromised.
- Attack happens through the worker-object matrix.

This is realistic for crowdsensing / truth discovery systems where data sources may be malicious or faulty.

Limitations:

- Current categorical attack assumes strong coordination.
- Current defense assumes collusion leaves high cross-task similarity.
- The model does not yet cover adaptive attackers.
- The model does not yet cover stealthy long-term low-amplitude attacks.
- The model does not cover server-side adversaries.

Safe formulation:

```text
We study input-level poisoning by fixed malicious workers.
```

Unsafe formulation:

```text
We solve malicious attacks against FPTD.
```

## 5. Is the Attack Simulation Convincing?

### Duck Categorical Collusion

The current Duck attack is convincing for demonstrating a clean failure mode:

```text
coordinated false labels
→ false consensus
→ baseline degradation
```

Evidence:

```text
30% attack baseline accuracy ≈ 0.500
40% attack baseline accuracy ≈ 0.259
```

But it may be considered too clean.

Likely reviewer critique:

```text
The attack is designed in a way that exactly matches the detector.
```

Needed improvement:

```text
noisy collusion
partial collusion
multi-target collusion
```

### Weather Persistent Biased Noise

The new Weather attack is more realistic for numerical data because it simulates persistent biased workers.

Evidence:

```text
clean baseline RMSE ≈ 29.86
30% biased-noise baseline RMSE ≈ 93.82
```

But defense evidence is preliminary:

```text
default MAD 3.0 fails
strict MAD 0.5 recovers but filters aggressively
```

Needed improvement:

```text
parameter sweep and repeated runs
```

## 6. Can the Defense Mechanism Be Questioned?

Yes. The defense mechanism can be questioned on several grounds.

### It May Be Overfitted to Highly Consistent Collusion

Current categorical defense detects high agreement. If malicious workers reduce agreement below 0.98, it may fail.

Needed response:

```text
soft suspiciousness scoring
```

### It May Confuse True Agreement with Collusion

If honest workers naturally agree strongly on simple tasks, high agreement does not necessarily mean attack.

Current mitigation:

```text
group size upper bound
```

But this is heuristic.

Needed evidence:

```text
clean false-positive tests
additional datasets
```

### It Filters Whole Workers

Whole-worker filtering may be too aggressive when attacks are partial.

Needed method:

```text
object-level selective filtering
```

### Numerical MAD Filtering Is Parameter-Sensitive

Weather results show that the default coefficient can fail.

Needed evidence:

```text
MAD sweep and clean side-effect analysis
```

## 7. Missing Key Comparisons

Current experiments should add the following comparisons.

### Focused Categorical Ablation

Compare:

```text
baseline
attacked baseline
ours without categorical collusion gateway
ours with categorical collusion gateway
ours with / without minimum support
```

Goal:

```text
prove recovery comes from collusion-aware filtering
```

### Hard Threshold vs Soft Score

Not implemented yet, but important if soft scoring is added.

Compare:

```text
hard threshold detector
soft suspiciousness scoring
```

under:

```text
clean
highly consistent collusion
noisy collusion
partial collusion
```

### MAD Coefficient Sweep

For Weather:

```text
0.5, 0.75, 1.0, 1.5, 2.0, 3.0
```

Goal:

```text
find robustness-utility tradeoff
```

### Runtime / Communication with Repetitions

Current timing can vary. Need repeated runs.

Compare:

```text
mean and standard deviation
preprocessing overhead
online/offline/communication
```

## 8. Conclusions That Cannot Be Overstated

Do not claim:

```text
The method defends against all malicious uploads.
The method solves general FPTD robustness.
The method works for all categorical datasets.
The method works for all numerical attacks.
The method handles adaptive attackers.
The robust bootstrap is the main cause of Duck recovery.
The Weather numerical branch is fully validated.
The overhead is fully characterized.
```

Safe current claims:

```text
Original FPTD is vulnerable to strong fixed-worker categorical collusion on Duck.
The current worker-level collusion filtering can recover Duck accuracy under highly consistent 30%-40% collusion attacks.
Clean Duck performance is not obviously harmed.
Weather persistent biased-noise attack can degrade baseline.
Strict MAD filtering can recover Weather RMSE in one tested setting, but needs parameter analysis.
```

## 9. Highest-Priority Experiments to Address Concerns

### Priority 1: Duck Threshold Sensitivity

Run:

```text
agreement threshold = 0.90, 0.95, 0.98, 1.00
group min/max ratio variations
```

Purpose:

```text
address heuristic-threshold concern
```

### Priority 2: Duck Clean False-Positive Test

Run:

```text
clean Duck under different thresholds
```

Purpose:

```text
show honest high agreement is not over-filtered
```

### Priority 3: Noisy Collusion

Run:

```text
10%, 20%, 30% random perturbation within malicious labels
```

Purpose:

```text
test whether the defense only works on perfectly coordinated attacks
```

### Priority 4: Partial Collusion

Run:

```text
attack only 30%, 50%, 70% of objects
```

Purpose:

```text
test whole-worker filtering limitation
```

### Priority 5: Weather MAD Sweep

Run:

```text
MAD coefficient sweep under persistent biased-noise attack
clean and attacked settings
```

Purpose:

```text
validate numerical branch and quantify filtering side effects
```

### Priority 6: Additional Categorical Dataset

Run:

```text
Dog or another compatible categorical dataset
```

Purpose:

```text
address Duck-specific generalization concern
```

## 10. Final Reviewer-Style Assessment

The project has a coherent and promising direction:

```text
input-level poisoning threat
→ FPTD degradation
→ robust preprocessing
→ recovered truth quality
```

The current strongest evidence is the Duck categorical collusion result. However, the current method is still closer to a validated prototype than a complete robustness framework.

The main concern is not whether the current result is real. It appears real for the tested attack. The main concern is scope:

```text
How far beyond highly consistent Duck collusion does the method work?
```

The next stage should therefore focus on:

- threshold sensitivity
- less clean attacks
- clean false positives
- additional datasets
- numerical parameter sweeps
- focused ablation

Only after these are addressed should the contribution be framed as a broader robust truth discovery method.

