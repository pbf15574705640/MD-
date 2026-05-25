# EXPERIMENT_CONTEXT

## 1. Overall Experimental Goal

The current experiment framework evaluates whether robust preprocessing can improve FPTD under malicious worker uploads.

The central experimental question is:

```text
If part of the worker-object matrix is poisoned by malicious workers,
does original FPTD degrade,
and can the robust preprocessing branch recover truth quality?
```

The main comparison is:

```text
clean baseline
attacked baseline
attacked ours
```

The project currently treats original FPTD without gateway as the main baseline. The robust branch, called `ours`, applies robust preprocessing before running original FPTD.

The old BooleanFilteringGateway variant is still present as `gateway-baseline`, but it is a reference only and should not be treated as the primary baseline in the current research narrative.

## 2. Baseline Branch

The baseline branch is the original no-gateway FPTD.

Conceptual flow:

```text
input worker-object matrix
→ original FPTD offline phase
→ original FPTD online phase
→ predicted truth
→ evaluation
```

In `PipelineFactory.baseline(...)`, the prepared matrix is simply the dataset's original matrix:

```text
preparedMatrix = dataset.getWorkerObjectMatrix()
retained_ratio = 1.0
reliable_worker_count = number of workers
restored_observation_count = 0
```

No robust bootstrap, gateway filtering, or support restoration is applied.

The baseline branch can be run on:

```text
clean input
attacked input
```

This distinction is important:

```text
clean baseline:
  original FPTD on clean data

attacked baseline:
  original FPTD on attacked data
```

## 3. Ours Branch

The `ours` branch uses robust preprocessing before original FPTD.

Conceptual flow:

```text
input worker-object matrix
→ robust bootstrap
→ adaptive gateway / filtering
→ minimum support constraint
→ prepared matrix
→ original FPTD
→ predicted truth
→ evaluation
```

Current default switches for full ours:

```text
enableRobustBootstrap = true
enableMadGateway = true
enableMinimumSupport = true
```

The robust bootstrap defaults to `MedianBootstrap` unless trimmed mean is explicitly enabled.

The gateway is data-type dependent:

```text
categorical dataset:
  CategoricalCollusionGateway

non-categorical / numerical dataset:
  MadGateway
```

The minimum support constraint can restore observations when an object has too few active observations after filtering.

Important current interpretation:

```text
Duck recovery is mainly from CategoricalCollusionGateway.
Weather numerical recovery, where observed, is from strict MAD filtering.
```

Robust bootstrap and minimum support exist in the pipeline, but current evidence does not show them as the main source of Duck recovery.

## 4. Attack Simulator in Experiments

Attack simulators operate on the worker-object matrix before evaluation.

General attack flow:

```text
clean worker-object matrix
→ select malicious workers according to attackerRatio and randomSeed
→ modify non-null observations of malicious workers
→ preserve honest workers and null cells
→ produce attacked matrix
```

The attacked matrix is wrapped into a new `ExperimentDataset` using the same worker IDs, object IDs, and ground truth.

The key fairness design is:

```text
attacked baseline and attacked ours use the same attacked matrix
```

This avoids comparing methods under different random attacks.

Current attack strategies include:

```text
CategoricalCollusionAttack
RandomNoiseAttack
ExtremeOutlierAttack
CollusionAttack
PersistentBiasedNoiseAttack
```

Current strongest categorical attack:

```text
CategoricalCollusionAttack
```

Current early numerical attack:

```text
PersistentBiasedNoiseAttack
```

## 5. Meaning of Clean Baseline, Attacked Baseline, and Attacked Ours

### Clean Baseline

Clean baseline means:

```text
clean dataset
→ original FPTD
→ predicted truth
```

It measures the original FPTD performance without malicious input.

### Attacked Baseline

Attacked baseline means:

```text
clean dataset
→ attack simulator
→ attacked dataset
→ original FPTD
→ predicted truth
```

It measures whether the attack actually degrades original FPTD.

If attacked baseline does not degrade relative to clean baseline, then the attack is not a useful robustness test for the current purpose.

### Attacked Ours

Attacked ours means:

```text
same attacked dataset
→ robust preprocessing
→ original FPTD
→ predicted truth
```

It measures whether the robust preprocessing branch can recover truth quality under the same attacked input.

The key robustness pattern is:

```text
clean baseline good
attacked baseline degraded
attacked ours recovered
```

## 6. Evaluation Metrics

Metrics are computed by `TruthDiscoveryEvaluator`.

### RMSE

RMSE is computed for all datasets:

```text
predicted truth vs ground truth
```

It is the primary metric for numerical datasets such as Weather.

### MAE

MAE is also computed for all datasets:

```text
average absolute error between predicted truth and ground truth
```

It is a secondary metric for numerical datasets.

### Accuracy

Accuracy is computed only when:

```text
dataset.isCategorical() == true
```

For categorical data, predicted numerical outputs are mapped to the nearest category among ground-truth categories.

Accuracy is the primary metric for Duck.

### Retained Ratio

Retained ratio measures how much of the input observation matrix remains after preprocessing:

```text
retained_ratio = countObserved(preparedMatrix) / countObserved(sourceMatrix)
```

For baseline:

```text
retained_ratio = 1.0
```

For ours, a lower retained ratio means the robust preprocessing filtered some observations.

### Reliable Worker Count

Reliable worker count is the number of workers retained as valid by the gateway result.

For baseline:

```text
reliable_worker_count = total worker count
```

For Duck categorical collusion, a drop in reliable worker count indicates that suspicious collusive workers were filtered.

### Restored Observation Count

Restored observation count measures how many observations were restored by minimum support constraint.

If this is zero, minimum support did not actively restore observations.

### Timing and Communication

Timing metrics:

```text
offline
online
total
```

Memory / communication metrics:

```text
before
after
delta
communication_bytes
```

`communication_bytes` is the server-to-server communication reported by `EdgeServer.getTotalBytesSent()` during protocol execution. It is not raw worker upload size.

Important limitation:

```text
Current timing mainly measures FPTD offline/online execution.
Defense preprocessing time is not separately instrumented.
```

## 7. Current Experimental Results and Meaning

### Duck Clean Baseline

Current clean Duck result:

```text
baseline accuracy ≈ 0.759
ours accuracy ≈ 0.759
```

Interpretation:

```text
The current robust preprocessing does not obviously harm clean Duck performance.
```

### Duck Categorical Collusion Attack

Current attack ratio results:

```text
10% attack:
  baseline accuracy ≈ 0.769
  ours accuracy ≈ 0.769

20% attack:
  baseline accuracy ≈ 0.759
  ours accuracy ≈ 0.750

30% attack:
  baseline accuracy ≈ 0.500
  ours accuracy ≈ 0.769

40% attack:
  baseline accuracy ≈ 0.259
  ours accuracy ≈ 0.769
```

Interpretation:

```text
10% and 20% attacks do not strongly degrade baseline.
30% and 40% attacks strongly degrade baseline.
Ours recovers accuracy under 30% and 40% highly consistent categorical collusion.
```

Defense behavior:

```text
30% attack:
  reliable workers reduced to 27
  retained ratio ≈ 0.692

40% attack:
  reliable workers reduced to 23
  retained ratio ≈ 0.590
```

This supports the claim that the categorical defense is filtering suspicious worker groups.

### Weather Persistent Biased-Noise Attack

Current Weather clean baseline:

```text
RMSE ≈ 29.86
MAE  ≈ 29.18
```

Weather 30% persistent biased-noise attack with `bias=200`, `noiseBound=20`:

```text
attacked baseline RMSE ≈ 93.82
attacked baseline MAE  ≈ 92.72
```

Interpretation:

```text
The new biased-noise attack can degrade Weather baseline.
```

MAD defense results:

```text
madCoefficient = 3.0:
  ours RMSE ≈ 93.82
  little recovery

madCoefficient = 1.0:
  ours RMSE ≈ 73.32
  partial recovery

madCoefficient = 0.5:
  ours RMSE ≈ 29.94
  near-clean recovery, but aggressive filtering
```

Clean Weather with `madCoefficient=0.5`:

```text
clean ours RMSE ≈ 29.88
retained ratio ≈ 0.462
restored observations = 903
```

Interpretation:

```text
Strict MAD can recover RMSE, but may filter heavily.
Weather numerical defense needs MAD sensitivity analysis before making strong claims.
```

## 8. Missing Experiments

### Attack Ratio Experiments

Completed for Duck categorical collusion at:

```text
10%, 20%, 30%, 40%
```

Still needed:

```text
Weather biased-noise attack ratio sweep
```

### Threshold / Parameter Sensitivity

Needed for Duck:

```text
agreement threshold sensitivity
group size min/max sensitivity
clean false-positive sensitivity
```

Needed for Weather:

```text
madCoefficient sweep
attack bias/noise strength sweep
clean side-effect analysis
```

### Ablation

Existing generic ablation modes:

```text
no robust bootstrap
no MAD gateway
no minimum support
```

Still needed:

```text
focused categorical collusion ablation
collusion filtering on/off
support restoration effect under categorical attack
MAD coefficient ablation under Weather biased-noise attack
```

### Robustness Tests

Still needed:

```text
noisy collusion attack
partial collusion attack
multi-target collusion attack
persistent single-worker fault
adaptive attack variants
additional categorical dataset
```

### Efficiency Tests

Current reports include timing and communication, but:

```text
preprocessing time is not separately measured
multiple repetitions are not systematically averaged
runtime variance is not reported
```

Needed:

```text
repeat runs
mean / standard deviation
preprocessing overhead instrumentation
larger dataset scalability
```

## 9. How to Run Current Experiments

Compile:

```powershell
mvn -q compile
```

Run default baseline batch using the dataset configured in `Params.java`:

```powershell
mvn -q exec:java '-Dexec.mainClass=fptd.evaluation.ExperimentRunner' '-Dexec.args=baseline'
```

Run Duck categorical attack batch:

```powershell
mvn -q exec:java '-Dexec.mainClass=fptd.evaluation.ExperimentRunner' '-Dexec.args=duck-attack'
```

Run other built-in modes:

```powershell
mvn -q exec:java '-Dexec.mainClass=fptd.evaluation.ExperimentRunner' '-Dexec.args=ablation'
mvn -q exec:java '-Dexec.mainClass=fptd.evaluation.ExperimentRunner' '-Dexec.args=attack-sweep'
mvn -q exec:java '-Dexec.mainClass=fptd.evaluation.ExperimentRunner' '-Dexec.args=mad-sweep'
mvn -q exec:java '-Dexec.mainClass=fptd.evaluation.ExperimentRunner' '-Dexec.args=shift-sweep'
```

Important:

```text
attack-sweep currently uses numerical CollusionAttack, not CategoricalCollusionAttack.
duck-attack uses CategoricalCollusionAttack at 30%.
Weather biased-noise experiments were run manually through JShell using PersistentBiasedNoiseAttack.
```

## 10. Output Files

Experiment reports are written to:

```text
target/eval-batches/<experiment-name>/
```

Each report directory contains:

```text
results.csv
results.json
summary.md
experiment.log
```

Common current output directories:

```text
target/eval-batches/baseline/
target/eval-batches/duck-attack/
target/eval-batches/duck-attack-ratio-10/
target/eval-batches/duck-attack-ratio-20/
target/eval-batches/duck-attack-ratio-30/
target/eval-batches/duck-attack-ratio-40/
target/eval-batches/weather-clean-baseline/
target/eval-batches/weather-clean-mad-0_5/
target/eval-batches/weather-biased-noise-ratio-30/
target/eval-batches/weather-biased-noise-ratio-30-mad-0_5/
target/eval-batches/weather-biased-noise-ratio-30-mad-1_0/
```

Figures are written to:

```text
target/eval-batches/figures/
```

Current figure examples:

```text
duck-defense-comparison.png / .svg
duck-overhead-comparison.png / .svg
duck-attack-ratio-sweep.png / .svg
```

## 11. Current Interpretation Boundaries

Safe current interpretation:

```text
The current experiment framework can fairly compare baseline and ours under the same attacked input.
Duck categorical collusion at 30% and 40% degrades baseline and is mitigated by current collusion-aware filtering.
Weather persistent biased-noise attack can degrade baseline, and strict MAD can recover in one tested setting.
```

Unsafe interpretation:

```text
The framework has proven general robustness against all malicious uploads.
The current defense handles noisy, partial, adaptive, or multi-target collusion.
The Weather numerical branch is fully validated.
Runtime overhead is fully characterized.
```

