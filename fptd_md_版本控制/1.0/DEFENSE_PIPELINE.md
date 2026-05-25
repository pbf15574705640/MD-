# DEFENSE_PIPELINE

This document describes the current robust preprocessing and defense pipeline from a system-mechanism perspective. It is based on the current implementation and experiment progress.

## 1. Overall Defense Pipeline

The current defense is implemented as a preprocessing layer before original FPTD.

The key design is:

```text
do not rewrite original FPTD core
clean or attacked worker-object matrix
→ robust preprocessing
→ prepared matrix
→ original FPTD
```

The full `ours` branch currently follows:

```text
Input worker-object matrix
→ Pilot truth estimation
→ Gateway-based detection and filtering
→ Minimum support recovery
→ Prepared matrix
→ Original FPTD offline + online execution
→ Predicted truth
```

The gateway is data-type dependent:

```text
categorical dataset:
  CategoricalCollusionGateway

numerical dataset:
  MadGateway
```

The current default `ours` switches are:

```text
enableRobustBootstrap = true
enableMadGateway = true
enableMinimumSupport = true
```

The term `enableMadGateway` is historical in the switch name. In the current enhanced pipeline, categorical data uses `CategoricalCollusionGateway` when this gateway switch is enabled.

## 2. Pilot Truth Estimation

Pilot truth estimation is the first step of robust preprocessing.

Its purpose is to produce a preliminary per-object reference before filtering. This reference can be used by numerical filtering and support recovery.

Current default:

```text
MedianBootstrap
```

For each object:

```text
1. Collect all non-null worker observations for the object.
2. Sort the values.
3. Take the median.
4. Store it as pilotTruth[object].
5. Record the support count.
```

If no values exist for an object:

```text
pilot truth = BigInteger.ZERO
```

Alternative implemented but not default:

```text
TrimmedMeanBootstrap
```

If robust bootstrap is disabled, the pipeline uses a mean-based bootstrap.

Important current interpretation:

```text
Pilot truth is central for numerical MAD filtering.
Pilot truth is not currently used by CategoricalCollusionGateway for Duck collusion detection.
```

Therefore, in current Duck results, pilot truth should not be claimed as the main recovery mechanism.

## 3. Suspicious Observation / Suspicious Worker Detection

The current system has two different detection mechanisms.

## 3.1 Numerical Suspicious Observation Detection

For non-categorical datasets such as Weather, detection is observation-level.

Module:

```text
MadGateway
```

For each object:

```text
1. Use pilotTruth[object].
2. Compute each observation's absolute deviation from pilot truth.
3. Compute MAD as the median of these deviations.
4. Accept or reject each observation based on MAD threshold.
```

Acceptance rule:

```text
deviation = |value - pilotTruth|

if MAD == 0:
  accept only observations with deviation == 0
else:
  accept if deviation <= madCoefficient * MAD
```

The numerical gateway therefore detects suspicious observations, not suspicious workers as a primary object.

It also reports `validWorkerCount`, where a worker is valid if at least one of its observations remains active.

Current limitation:

```text
MAD effectiveness depends strongly on madCoefficient and on whether pilot truth is sufficiently robust.
```

Weather early evidence shows:

```text
madCoefficient = 3.0 may be too loose for persistent biased-noise attack
madCoefficient = 0.5 can recover RMSE but filters aggressively
```

## 3.2 Categorical Suspicious Worker Detection

For categorical datasets such as Duck, detection is worker-level.

Module:

```text
CategoricalCollusionGateway
```

The gateway analyzes whether multiple workers show abnormal cross-task agreement.

For each pair of workers:

```text
common = number of objects both workers answered
same   = number of common objects where labels are equal
agreement = same / common
```

Current hard-rule condition:

```text
common >= GATEWAY_CATEGORICAL_COLLUSION_MIN_COMMON_OBJECTS
agreement >= GATEWAY_CATEGORICAL_COLLUSION_AGREEMENT_THRESHOLD
```

Current parameter values:

```text
minimum common objects = 10
agreement threshold = 0.98
```

Pairs satisfying this condition are connected through union-find. Connected components become candidate worker groups.

A candidate group is suspicious if:

```text
group size >= max(3, ceil(workerCount * 0.20))
group size <= floor(workerCount * 0.50)
```

Current parameter values:

```text
min ratio = 0.20
max ratio = 0.50
```

For Duck with 39 workers:

```text
suspicious group size range = 8 to 19 workers
```

This module detects suspicious workers, not individual suspicious cells.

Current strength:

```text
effective for highly consistent categorical collusion
```

Current limitation:

```text
not yet validated for noisy, partial, multi-target, or adaptive collusion
```

## 4. Unreliable Observation Filtering

Both numerical and categorical gateways output a participation mask.

The participation mask marks each worker-object cell as:

```text
active = retained
inactive = filtered
```

### Numerical Filtering

For numerical data:

```text
each observation is filtered independently based on MAD deviation
```

This means a worker can have some observations retained and others filtered.

### Categorical Filtering

For categorical data:

```text
if a worker belongs to a suspicious collusive group:
  all non-null observations from that worker are marked inactive
else:
  observations are retained
```

This is currently whole-worker filtering.

This is appropriate for the current fixed-worker full-row collusion attack, but it may be too coarse for partial-task attacks.

The filtered matrix is generated by applying the participation mask:

```text
active cell   → keep original value
inactive cell → null
```

## 5. Support Recovery

Support recovery exists in the current implementation.

Module:

```text
MinimumSupportConstraint
```

Purpose:

```text
avoid over-filtering that leaves too few observations for an object
```

Process:

```text
for each object:
  count active observations
  if active count >= minimumSupport:
    do nothing
  else:
    collect inactive non-null observations
    sort them by distance to pilot truth
    restore closest observations until active count reaches minimumSupport
```

Current default:

```text
minimumSupport = 3
```

Support recovery is not an attack detector. It is a stability mechanism.

Current interpretation:

```text
Duck:
  restored_observation_count is often 0 in main results,
  so support recovery is not the main source of Duck recovery.

Weather:
  support recovery can trigger under strict MAD filtering.
```

Potential concern:

```text
if pilot truth is polluted, restoring closest observations may restore bad observations
```

This needs experiment-level validation.

## 6. Prepared Matrix Input to Original FPTD

After gateway filtering and optional support recovery, the pipeline constructs the final prepared matrix.

If minimum support is enabled:

```text
prepared matrix = supportResult.correctedMatrix
```

If minimum support is disabled:

```text
prepared matrix = gatewayResult.filteredMatrix
```

The `RobustFptdAdapter` returns the corrected matrix if support recovery exists; otherwise it returns the filtered matrix.

The prepared matrix is then passed to original FPTD:

```text
TDOfflineOptimal(workerCount, objectCount, jobName).runTDOffline()
TDOnlineOptimal(workerCount, objectCount).buildTDCircuit(preparedMatrix, jobName)
```

Original FPTD then produces predicted truth values, which are evaluated against ground truth.

## 7. Which Attack-Chain Problem Each Module Addresses

### Pilot Truth Estimation

Attack-chain problem:

```text
raw observations may be polluted before filtering
```

Defense role:

```text
produce a more robust object-level reference than simple raw aggregation
```

Current evidence:

```text
implemented
important for numerical MAD logic
not proven as main Duck recovery source
```

### MAD Numerical Filtering

Attack-chain problem:

```text
malicious numerical observations can bias truth estimates
```

Defense role:

```text
filter observations far from pilot truth
```

Current evidence:

```text
early Weather evidence under strict MAD
not yet fully validated across parameters
```

### Categorical Collusion Filtering

Attack-chain problem:

```text
malicious categorical workers coordinate false labels
→ false consensus
```

Defense role:

```text
detect abnormal cross-task worker agreement
filter suspicious collusive workers
```

Current evidence:

```text
strongest current evidence on Duck 30% and 40% attacks
```

### Minimum Support Recovery

Attack-chain problem:

```text
filtering can make some objects too sparse
```

Defense role:

```text
restore enough observations to keep FPTD executable and stable
```

Current evidence:

```text
implemented
not main Duck recovery source
needs more ablation
```

## 8. Core Innovation vs Engineering Support

## 8.1 Most Plausible Core Innovation

The most plausible current core innovation is:

```text
worker-level categorical collusion-aware filtering
```

Reason:

```text
It directly targets the attack mechanism observed in Duck:
coordinated malicious workers form false consensus.
```

This module is not just engineering glue. It changes the detection unit from:

```text
single observation deviation
```

to:

```text
cross-task worker behavior similarity
```

## 8.2 Supporting Robustness Modules

Supporting modules:

```text
MedianBootstrap / TrimmedMeanBootstrap
MadGateway
MinimumSupportConstraint
```

These are important for a robust preprocessing framework, especially for numerical data and stability, but current evidence does not show all of them as primary innovations.

## 8.3 Engineering / Experiment Infrastructure

Engineering support includes:

```text
PipelineFactory branch construction
ExperimentRunner batch modes
ExperimentIo result output
PipelineArtifacts metadata
RobustFptdAdapter
```

These are necessary for reproducible experiments but should not be overclaimed as paper-level innovation.

## 9. Current Defense Limitations

### 9.1 Hard Thresholds

The categorical detector currently uses hard thresholds:

```text
agreement >= 0.98
group size in 20%-50%
```

These are heuristic and need sensitivity analysis.

### 9.2 Highly Consistent Attack Assumption

The current Duck defense is validated only against highly consistent categorical collusion.

Not yet validated:

```text
noisy collusion
partial collusion
multi-target collusion
adaptive collusion
```

### 9.3 Whole-Worker Filtering

Current categorical filtering removes all observations from suspicious workers.

This may be too aggressive for attacks where:

```text
workers are malicious only on some objects
workers mix honest and malicious behavior
```

Object-level selective filtering is not yet implemented.

### 9.4 Numerical Parameter Sensitivity

Weather results show that MAD filtering can be highly sensitive to `madCoefficient`.

Default:

```text
madCoefficient = 3.0
```

worked poorly in the tested biased-noise attack, while:

```text
madCoefficient = 0.5
```

recovered RMSE but filtered aggressively.

### 9.5 No Long-Term Reputation Defense

The current `ours` branch does not yet include a new validated reputation-aware defense for persistent low-quality workers.

This means current defense is not yet proven against:

```text
single persistent malicious worker
faulty device with mild long-term bias
non-collusive low-quality worker
```

### 9.6 Defense Does Not Use Ground Truth

This is correct and necessary, but it also means the defense cannot directly know whether a worker is wrong. It relies on observable patterns:

```text
numerical deviation
categorical agreement
support counts
```

If malicious behavior does not produce these patterns, the current defense may not detect it.

## 10. Safe Interpretation of Current Defense

Safe current statement:

```text
The current robust preprocessing branch can mitigate highly consistent worker-level categorical collusion on Duck by detecting abnormal cross-task worker agreement and filtering suspicious workers before original FPTD.
```

Also safe but preliminary:

```text
For Weather numerical biased-noise attack, strict MAD filtering can recover RMSE in an initial tested setting, but parameter sensitivity remains unresolved.
```

Unsafe current statement:

```text
The defense pipeline solves general malicious worker attacks.
The defense is robust to noisy, partial, adaptive, or long-term low-amplitude attacks.
The numerical defense is fully validated.
```

