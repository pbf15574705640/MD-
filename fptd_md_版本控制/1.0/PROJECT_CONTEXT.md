# PROJECT_CONTEXT

## 1. Project Overview

This project studies robustness for the original FPTD truth discovery pipeline under malicious worker uploads.

The original FPTD system focuses on privacy-preserving truth discovery. It can estimate truths from worker-provided observations while preserving privacy through the original protocol. However, the original pipeline assumes that the uploaded worker observations are usable as inputs. If a group of workers intentionally uploads poisoned data, the secure computation protocol may still execute correctly while the final truth estimates become unreliable.

The current project therefore shifts the research focus from only privacy-preserving computation to malicious-input robustness:

```text
Can FPTD still produce reliable truths when part of the worker-object matrix is poisoned by malicious workers?
```

The current goal is not to replace the original FPTD core. The main design principle is low-intrusion robust preprocessing:

```text
Raw / attacked worker-object matrix
→ robust preprocessing
→ prepared matrix
→ original FPTD
→ predicted truth
```

The current strongest result is in the Duck categorical dataset under fixed-worker collusive false-label attack. A second numerical robustness direction has recently started on Weather using persistent biased-noise attack.

## 2. Main Story

The main research narrative is:

```text
Attack → Degradation → Defense → Recovery
```

### Attack

The attacker controls a subset of workers and modifies their uploaded observations. The attack happens at the input layer, before FPTD execution. The server-side FPTD protocol and truth discovery logic are not directly compromised.

For Duck, the main effective attack is categorical collusion:

```text
fixed malicious workers
→ coordinated false labels
→ false consensus in the worker-object matrix
```

For Weather, a newer attack direction is persistent biased noise:

```text
fixed malicious workers
→ continuous numerical bias + random noise
→ persistent deviation from normal observations
```

### Degradation

Original FPTD does not distinguish honest observations from poisoned observations before truth discovery. If malicious workers form a sufficiently strong false consensus or inject strong numerical bias, the initial truth estimation and subsequent truth discovery process can be polluted.

Observed degradation:

```text
Duck clean baseline accuracy ≈ 0.759
Duck 30% categorical collusion baseline accuracy ≈ 0.500
Duck 40% categorical collusion baseline accuracy ≈ 0.259
```

Weather biased-noise attack also begins to show degradation:

```text
Weather clean baseline RMSE ≈ 29.86
Weather 30% persistent biased-noise baseline RMSE ≈ 93.82
```

### Defense

The proposed approach inserts a robust preprocessing layer before original FPTD. Depending on the data type, the preprocessing layer can use:

```text
robust pilot estimation
→ gateway / filtering
→ support preservation
→ original FPTD
```

For Duck categorical collusion, the current key defense is worker-level collusion-aware filtering. It detects suspicious worker groups based on abnormal cross-task agreement.

For Weather numerical attacks, the current numerical defense branch is median bootstrap + MAD filtering + minimum support. This branch is still under early validation.

### Recovery

The current strongest recovery evidence is Duck:

```text
30% attack:
baseline accuracy ≈ 0.500
ours accuracy ≈ 0.769

40% attack:
baseline accuracy ≈ 0.259
ours accuracy ≈ 0.769
```

Weather has early evidence that stricter MAD filtering can recover RMSE under persistent biased-noise attack:

```text
30% biased-noise attack:
baseline RMSE ≈ 93.82
ours with default MAD coefficient 3.0 RMSE ≈ 93.82
ours with MAD coefficient 1.0 RMSE ≈ 73.32
ours with MAD coefficient 0.5 RMSE ≈ 29.94
```

This Weather evidence is preliminary because the effective threshold may be aggressive and needs sensitivity analysis.

## 3. Threat Model

The current threat model assumes malicious workers, not malicious servers.

The attacker can:

- Control a fraction of workers.
- Modify the observations submitted by those workers.
- Coordinate the malicious workers.
- Poison the worker-object matrix before FPTD runs.

The attacker cannot:

- Modify server-side FPTD code.
- Directly corrupt the secure computation protocol.
- Use ground truth during defense.

The input to FPTD is a worker-object matrix:

```text
rows    = workers
columns = objects / tasks
cells   = uploaded observation or label
```

The attack modifies only the rows/cells corresponding to selected malicious workers. Honest workers remain unchanged.

### Categorical Collusion Attack

In Duck, malicious workers submit coordinated false labels. The attack creates a misleading false consensus:

```text
honest workers: normal label distribution
malicious workers: coordinated false label distribution
→ original FPTD may treat false labels as valid consensus
```

This attack is best described as:

- collusive false-label injection
- data poisoning
- worker-level collusion attack
- categorical consensus manipulation

Current attack limitation: the malicious workers are highly coordinated. This is useful for proving vulnerability and initial defense feasibility, but it is a relatively clean attack pattern.

### Numerical Biased-Noise Attack

In Weather, a new attack class simulates persistent biased numerical uploads:

```text
malicious_value = original_value + bias + random_noise
```

This models fixed workers that frequently deviate from normal values, such as faulty sensors, calibrated-bad devices, or persistently malicious numerical uploads.

This Weather attack is newly added and has only early validation.

## 4. Defense Pipeline

The current robust pipeline contains the following conceptual stages.

### 4.1 Pilot Truth Estimation

The robust bootstrap module estimates a pilot truth for each object before filtering.

Current default:

```text
MedianBootstrap
```

For each object:

```text
collect non-null worker observations
→ sort values
→ take median
→ record support count
```

Trimmed mean bootstrap also exists but is not the default in current experiments.

Important limitation: in the current Duck categorical collusion result, the pilot truth is not the main driver of success. The categorical collusion gateway receives `pilotTruth` but does not use it for the current collusion decision. Duck recovery is mainly due to worker-level collusion filtering.

### 4.2 Suspicious Observation Detection

For numerical data, the current gateway is MAD-based:

```text
deviation = |observation - pilotTruth|
MAD = median of deviations per object
accept if deviation <= madCoefficient * MAD
```

This is suitable for observation-level numerical outliers, but its effectiveness depends heavily on the MAD coefficient and attack magnitude.

### 4.3 Collusive Worker Analysis

For categorical data, the current gateway uses worker-level agreement.

For each pair of workers:

```text
agreement = same labels on common tasks / number of common tasks
```

Current hard-rule detector:

```text
agreement >= 0.98
common tasks >= 10
group size in [20%, 50%] of workers
→ suspicious collusive group
```

For Duck with 39 workers:

```text
minimum suspicious group size = 8
maximum suspicious group size = 19
```

This detector is effective for highly consistent collusive workers. It is not yet proven for noisy, partial, adaptive, or multi-target collusion.

### 4.4 Unreliable Observation Filtering

The gateway returns a participation mask. Suspicious observations or suspicious workers are filtered before original FPTD.

For current Duck collusion filtering, the filtering is worker-level:

```text
if worker is in suspicious coalition
→ filter that worker's observations
```

This is intentionally simple but can be too coarse for partial-task attacks.

### 4.5 Support Recovery

Minimum support constraint exists to reduce over-filtering risk.

Conceptually:

```text
after filtering, check active observations per object
if active count < minimumSupport
restore closest observations according to pilot truth
```

In Duck current results, restoration count is usually 0, so support recovery is not the primary source of recovery.

In Weather, support recovery can trigger more often, especially under strict MAD filtering.

## 5. Current Innovation Modules

This section separates potential contributions from engineering and experiment support.

### 5.1 Potential Contribution Modules

#### Worker-Level Collusion-Aware Filtering

This is the most plausible current methodological contribution.

It addresses categorical false consensus attacks by detecting abnormal cross-task agreement among workers. The core idea is:

```text
attack uses worker-level coordination
→ defense detects worker-level behavioral similarity
```

Current evidence supports this module under highly consistent Duck collusion attack.

#### Support-Preserving Robust Preprocessing

Minimum support is a useful robustness-utility component. It helps prevent over-filtering from making FPTD unstable.

Current evidence does not show it as the main reason for Duck recovery. It should be described as a supporting mechanism, not the primary innovation.

#### Numerical MAD-Based Robust Filtering

This module is relevant for Weather numerical attacks. Early Weather biased-noise results show that strict MAD thresholds can recover RMSE, but default settings do not.

This branch is promising but still preliminary.

### 5.2 Engineering Optimization Modules

These are important for project reliability but should not be overclaimed as paper contributions:

- Fair attacked-input construction.
- `AttackResult` storing malicious worker mask.
- Batch experiment runner modes.
- CSV / JSON / Markdown report output.
- Plot generation scripts.
- Dataset wrappers for attacked matrices.

### 5.3 Experiment Support Modules

These modules support evaluation but are not themselves the main method contribution:

- Categorical collusion attack simulator.
- Persistent biased-noise attack simulator.
- Attack ratio sweep scripts / JShell runs.
- Runtime and communication plots.
- Result aggregation scripts.

## 6. Current Experimental Framework

### Baseline vs Ours

Current naming convention:

```text
baseline = original FPTD without gateway
ours     = robust preprocessing + original FPTD
gateway-baseline = old BooleanFilteringGateway reference only
```

Main paper comparisons should focus on:

```text
clean baseline
attacked baseline
attacked ours
```

The old `gateway-baseline` should not be treated as the primary baseline.

### Fair Robustness Evaluation

The attacked input is generated once and shared across methods:

```text
clean worker-object matrix
→ attack simulator
→ attacked worker-object matrix
→ baseline branch
→ ours branch
```

This ensures that baseline and ours face the same malicious worker selection and the same attacked dataset.

### Attack Ratio Experiments

Duck categorical collusion attack has been run at:

```text
10%, 20%, 30%, 40%
```

Observed:

```text
10%, 20%: baseline not strongly degraded
30%, 40%: baseline strongly degraded, ours recovers
```

### Ablation

Some ablation modes exist in the code, but current strongest Duck result still needs a focused ablation around:

- categorical collusion filtering on/off
- robust bootstrap on/off
- minimum support on/off
- hard threshold vs future soft score

Existing generic ablations are not enough to fully explain the new categorical defense.

### Evaluation Metrics

Categorical data:

```text
accuracy
RMSE / MAE also reported but accuracy is primary
retained_ratio
reliable_worker_count
restored_observation_count
runtime
communication_bytes
```

Numerical data:

```text
RMSE
MAE
retained_ratio
reliable_worker_count
restored_observation_count
runtime
communication_bytes
```

## 7. Current Evidence

### 7.1 Experimentally Supported

The following claims are currently supported by experiments:

1. Original FPTD is vulnerable to Duck categorical collusion at sufficiently high attack ratios.

```text
30% attack baseline accuracy ≈ 0.500
40% attack baseline accuracy ≈ 0.259
```

2. Current worker-level collusion filtering recovers Duck truth quality under 30% and 40% highly consistent categorical collusion.

```text
30% attack ours accuracy ≈ 0.769
40% attack ours accuracy ≈ 0.769
```

3. Clean Duck performance is not obviously harmed by the current categorical defense.

```text
clean baseline accuracy ≈ 0.759
clean ours accuracy ≈ 0.759
```

4. Duck defense behavior is interpretable.

```text
30% attack: reliable workers reduced from 39 to 27
40% attack: reliable workers reduced from 39 to 23
```

5. Weather persistent biased-noise attack can degrade baseline RMSE.

```text
clean Weather baseline RMSE ≈ 29.86
30% biased-noise baseline RMSE ≈ 93.82
```

6. Weather strict MAD filtering can recover RMSE in one tested setting.

```text
MAD coefficient 0.5:
attacked ours RMSE ≈ 29.94
```

### 7.2 Reasonable but Not Yet Proven

The following are plausible but not fully verified:

- Soft suspiciousness scoring may handle noisy collusion better than hard thresholds.
- Object-level selective filtering may handle partial collusion better than whole-worker filtering.
- Reputation-aware filtering may help with persistent individual worker faults.
- MAD-based numerical defense may be effective across a range of Weather attack strengths and thresholds.
- The categorical collusion defense may generalize to non-Duck categorical datasets.

### 7.3 Not Yet Supported

Current evidence does not support claiming:

- Defense against all malicious upload attacks.
- Defense against adaptive attackers.
- Defense against noisy / partial / multi-target collusion.
- Defense against single-worker persistent faults.
- Generalization to all categorical truth discovery datasets.
- Generalization to all numerical datasets.
- The robust bootstrap module is the main source of Duck recovery.

## 8. Reviewer Concerns

Likely reviewer concerns:

### Attack Is Too Clean

The current Duck attack produces highly consistent malicious workers. Reviewers may argue that real colluders may add noise, attack only part of the tasks, or split into multiple groups.

Current response:

```text
The current experiment validates a clear fixed-worker collusion threat model.
It does not yet prove robustness against noisy or adaptive collusion.
```

Needed evidence:

- noisy collusion attack
- partial collusion attack
- multi-target collusion attack

### Hard Thresholds Are Heuristic

Current collusion detection uses:

```text
agreement >= 0.98
group size between 20% and 50%
```

These thresholds are currently heuristic.

Needed evidence:

- threshold sensitivity
- group size sensitivity
- clean false positive tests

### Duck Is Too Limited

Duck is a binary categorical dataset. Reviewers may question whether results generalize to multi-class categorical truth discovery.

Needed evidence:

- another categorical dataset, preferably multi-class
- Dog / Face Sentiment / Product if compatible

### Numerical Branch Is Preliminary

Weather results are early. Default MAD coefficient 3.0 failed to recover under biased-noise attack, while coefficient 0.5 worked but filtered aggressively.

Needed evidence:

- MAD coefficient sweep
- clean side-effect analysis
- attack strength sweep

### Engineering vs Methodological Contribution

Some changes improve experimentation but are not methodological contributions. The paper should not overclaim experiment infrastructure as core innovation.

## 9. Current Project Stage

The project is currently between prototype stage and experiment stage.

More precise assessment:

```text
Duck categorical collusion branch:
prototype + initial experiment evidence

Weather numerical noise branch:
early feasibility stage

general multi-attack robust framework:
theoretical design stage
```

The project is not yet at a mature full-paper robustness evaluation stage. The core attack-defense chain has been demonstrated for one strong categorical scenario, and a second numerical line has started showing feasibility.

## 10. Future Research Direction

### Highest Priority Experiments

1. Duck threshold sensitivity:

```text
agreement threshold: 0.90, 0.95, 0.98, 1.00
group min/max ratio sweep
```

2. Duck clean false-positive test:

```text
ensure clean data is not over-filtered under different thresholds
```

3. Duck noisy / partial collusion:

```text
malicious workers not perfectly aligned
attack only part of objects
```

4. Weather MAD sweep:

```text
madCoefficient = 0.5, 0.75, 1.0, 1.5, 2.0, 3.0
```

5. Weather attack strength sweep:

```text
bias = 100, 200, 500
noiseBound = 20 or 10% of bias
attackerRatio = 10%, 20%, 30%, 40%
```

### Most Promising Innovation Strengthening

The most promising next methodological direction is soft suspiciousness scoring:

```text
worker similarity
group compactness
attack coverage
deviation from population
group size score
→ continuous suspiciousness score
```

This would generalize the current hard-threshold collusion detector and directly address the concern that current attacks are too clean.

The second promising direction is object-level selective filtering:

```text
detect suspicious group
→ localize suspicious objects
→ filter only suspicious cells
```

This would address partial collusion and reduce over-filtering.

The third direction is reputation-aware filtering:

```text
track long-term worker reliability
→ handle persistent faults and non-collusive malicious workers
```

This is important but should be introduced only after the current collusion branch is better validated.

### Most Dangerous Current Weakness

The most dangerous weakness is overclaiming.

The currently defensible claim is:

```text
The current method improves robustness against highly consistent worker-level categorical collusion in Duck.
```

The currently unsafe claim is:

```text
The method generally defends FPTD against malicious uploads.
```

Future work should broaden the evidence before broadening the claim.

