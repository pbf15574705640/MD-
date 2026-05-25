# ATTACK_MODEL

This document describes the current attack model based strictly on the implemented attack simulators and experiment setup.

## 1. Attacker Identity: Malicious Workers

The attacker is modeled as a subset of malicious workers.

These malicious workers are selected from the worker rows of the worker-object matrix. The attack does not compromise FPTD servers, offline data generation logic, MPC protocol execution, or evaluation code.

The attack surface is the uploaded worker observations:

```text
worker-object matrix rows controlled by malicious workers
```

Current attack simulators assume:

```text
honest workers remain unchanged
malicious workers' non-null observations may be modified
null cells remain null
```

## 2. Attack Goal

The attack goal is to degrade truth discovery quality by poisoning the input matrix.

For categorical data such as Duck:

```text
goal = push final predicted labels toward false labels
```

For numerical data such as Weather:

```text
goal = increase RMSE / MAE by biasing numerical truth estimates
```

The attacker aims to affect the final predicted truth without modifying the original FPTD algorithm or protocol.

The target degradation path is:

```text
malicious worker uploads
→ polluted worker-object matrix
→ biased initial / iterative truth estimation
→ degraded predicted truth
```

## 3. Data Controlled by the Attacker

The attacker can control observations submitted by selected malicious workers.

For each malicious worker:

```text
if cell value is non-null:
  attack simulator may replace or transform it

if cell value is null:
  remains null
```

The attacker does not change:

```text
worker IDs
object IDs
ground truth
honest worker observations
matrix dimensions
server-side protocol
```

The attacked matrix has the same shape as the original worker-object matrix.

## 4. Malicious Worker Selection

Malicious workers are selected by `AttackSupport.selectMaliciousWorkers(...)`.

Inputs:

```text
workerCount
attackRatio
seed
```

Selection process:

```text
1. Validate attackRatio is in [0, 1].
2. Compute maliciousWorkerCount = ceil(workerCount * attackRatio).
3. Clamp maliciousWorkerCount to [1, workerCount] if attackRatio > 0.
4. Shuffle worker indices using Random(seed).
5. Mark the first maliciousWorkerCount shuffled indices as malicious.
```

If:

```text
workerCount == 0 or attackRatio == 0.0
```

then no worker is selected as malicious.

The attack result records the selected malicious workers in:

```text
maliciousWorkerMask
```

This mask is used for analysis but is not available to the defense as ground-truth knowledge.

## 5. Attack Ratio and Attack Strength

### Attack Ratio

The attack ratio controls the fraction of workers selected as malicious:

```text
attackRatio = malicious workers / total workers
```

Because the code uses `ceil(workerCount * attackRatio)`, small nonzero ratios still select at least one malicious worker.

Examples:

Duck has 39 workers in the current default dataset.

```text
10% → ceil(39 * 0.10) = 4 malicious workers
20% → ceil(39 * 0.20) = 8 malicious workers
30% → ceil(39 * 0.30) = 12 malicious workers
40% → ceil(39 * 0.40) = 16 malicious workers
```

Weather has 9 workers in the current full dataset.

```text
30% → ceil(9 * 0.30) = 3 malicious workers
```

### Attack Strength

Attack strength depends on the specific attack class.

Implemented strength controls include:

```text
CollusionAttack:
  collusiveShift

RandomNoiseAttack:
  noiseBound

ExtremeOutlierAttack:
  outlierMagnitude

PersistentBiasedNoiseAttack:
  biasInRealUnits
  noiseBoundInRealUnits
```

Important scaling detail:

Most worker observations are stored internally as:

```text
raw answer * Params.PRECISE_ROUND
```

`CollusionAttack`, `RandomNoiseAttack`, and `ExtremeOutlierAttack` operate directly on the internal scaled `BigInteger` values. Therefore, a small shift such as `50` is very small in real units when `PRECISE_ROUND = 100000`.

`PersistentBiasedNoiseAttack` is different. It accepts bias and noise in real units and scales them internally by `Params.PRECISE_ROUND`.

## 6. Implemented Attack Types

## 6.1 CategoricalCollusionAttack

This is the current main categorical attack for Duck.

Purpose:

```text
simulate fixed malicious workers coordinating false categorical labels
```

Process:

```text
1. Select malicious workers.
2. Build a target label for each object.
3. For malicious workers:
   - if their current label differs from the target label, replace it with the target label
   - otherwise keep it unchanged
4. Honest workers remain unchanged.
```

Target label construction:

For each object:

```text
count observed labels
rank labels by frequency, descending
if at least two labels exist:
  target label = second most frequent label
else:
  target label = an alternative global label if available
```

If no label exists for an object:

```text
target label = null
```

In that case the attack does not modify that object.

This attack can make malicious workers submit coordinated false labels, creating a false consensus or competing consensus.

Current validated dataset:

```text
Duck
```

Current strong evidence:

```text
30% attack baseline accuracy ≈ 0.500
40% attack baseline accuracy ≈ 0.259
```

## 6.2 CollusionAttack

This is a numerical shift attack.

Process:

```text
select malicious workers
for each non-null malicious observation:
  value = value + collusiveShift
```

The `collusiveShift` is applied directly to the internal scaled value.

Current limitation:

Earlier Weather shift experiments using small shift values did not strongly degrade baseline. This was partly because the shift values were small relative to the internal scaling.

## 6.3 RandomNoiseAttack

This attack adds random signed noise to malicious workers' observations.

Process:

```text
select malicious workers
for each non-null malicious observation:
  delta ∈ [-noiseBound, +noiseBound]
  value = value + delta
```

The `noiseBound` is applied directly to the internal scaled value.

Current status:

```text
implemented
not currently the main validated attack-defense result
```

## 6.4 ExtremeOutlierAttack

This attack adds a fixed outlier magnitude to malicious workers' observations.

Process:

```text
select malicious workers
for each non-null malicious observation:
  value = value + outlierMagnitude
```

The `outlierMagnitude` is applied directly to the internal scaled value.

Current status:

```text
implemented
not currently the main validated attack-defense result
```

## 6.5 SignFlipAttack

This attack negates malicious workers' observations.

Process:

```text
select malicious workers
for each non-null malicious observation:
  value = -value
```

Current status:

```text
implemented
not currently used as a main reported result
```

## 6.6 PersistentBiasedNoiseAttack

This is the current numerical biased-noise attack used for early Weather exploration.

Purpose:

```text
simulate fixed malicious workers that frequently deviate from normal numerical values
```

Process:

```text
select malicious workers
scale bias by PRECISE_ROUND
scale noiseBound by PRECISE_ROUND
for each non-null malicious observation:
  value = value + scaledBias + randomSignedNoise
```

Formula:

```text
malicious_value = original_value + bias * PRECISE_ROUND + random(-noiseBound, +noiseBound) * PRECISE_ROUND
```

Current tested setting:

```text
Weather
attackRatio = 0.30
biasInRealUnits = 200
noiseBoundInRealUnits = 20
```

Observed baseline degradation:

```text
Weather clean baseline RMSE ≈ 29.86
Weather attacked baseline RMSE ≈ 93.82
```

This attack is newly added and should be treated as early evidence, not a mature validated benchmark yet.

## 7. Collusion Attack and False Consensus Formation

The strongest current false-consensus attack is `CategoricalCollusionAttack`.

False consensus forms as follows:

```text
1. A fixed group of malicious workers is selected.
2. For each object, a target label is chosen.
3. Malicious workers are pushed toward the same target label for that object.
4. Their coordinated labels form a consistent group signal.
5. Original FPTD treats those labels as ordinary observations.
6. The estimated truth can be pulled toward the coordinated false signal.
```

In Duck, this is effective when the malicious worker group is large enough.

Observed:

```text
10% and 20% attacks do not strongly degrade baseline
30% and 40% attacks strongly degrade baseline
```

This suggests the attack needs sufficient malicious support to form a meaningful false consensus.

## 8. Worker-Object Matrix Pollution

All attacks modify the worker-object matrix before FPTD execution.

General pollution pattern:

```text
original worker-object matrix
→ attack simulator
→ attacked worker-object matrix
```

The attacked matrix keeps:

```text
same number of workers
same number of objects
same worker IDs
same object IDs
same missing-value structure for null cells
```

Only selected malicious workers' non-null observations are changed.

The attacked matrix is then wrapped into a dataset:

```text
dataset.withWorkerObjectMatrix(...)
```

This ensures that:

```text
attacked baseline
attacked ours
```

use the same attacked input.

## 9. Why Original FPTD Is Affected

Original FPTD directly consumes the prepared worker-object matrix.

The baseline branch does not perform:

```text
malicious worker detection
collusion analysis
robust filtering
minimum support correction
```

Therefore, poisoned observations can participate in:

```text
initial truth estimation
worker weight computation
iterative truth update
```

For categorical collusion, coordinated false labels can be interpreted as a legitimate consensus signal.

For numerical attacks, biased or noisy observations can shift the estimated truth and increase RMSE / MAE.

The important point is:

```text
FPTD protocol execution can be correct,
but the input matrix can be poisoned,
so the output truth can be wrong.
```

## 10. Reasonableness of the Current Attack Model

The current attack model is reasonable for studying input-level poisoning because:

- Worker uploads are a natural attack surface in truth discovery.
- A malicious user or faulty device can submit incorrect observations.
- Fixed malicious workers are more realistic than random cell-level corruption.
- Collusive false-label injection is a plausible threat for categorical crowdsensing.
- Persistent biased noise is a plausible model for faulty numerical sensors or long-term biased users.

The fair attacked-input design is also reasonable:

```text
generate one attacked matrix
use it for both baseline and ours
```

This avoids comparing methods under different attack samples.

## 11. Limitations of the Current Attack Model

The current attack model has important limitations.

### Categorical Collusion Is Highly Coordinated

Current Duck attack creates highly consistent malicious worker behavior. This is useful for demonstrating vulnerability but may be cleaner than real attacks.

Not yet implemented:

```text
noisy collusion
partial collusion
multi-target collusion
adaptive collusion
```

### No Server-Side Adversary

The current threat model does not cover malicious servers or compromised FPTD protocol execution.

### No Identity-Level Sybil Metadata

The code treats workers as rows in the matrix. There is no separate identity metadata or account-level signal for Sybil detection.

### Attack Strength Scaling Can Be Subtle

Some numerical attacks operate directly on scaled internal values. Small parameter values may be too weak in real units.

This matters for:

```text
CollusionAttack
RandomNoiseAttack
ExtremeOutlierAttack
```

### Persistent Biased Noise Is Early

`PersistentBiasedNoiseAttack` has shown initial Weather degradation, but it has not yet been evaluated across:

```text
multiple attacker ratios
multiple bias values
multiple noise bounds
multiple MAD coefficients
```

### Current Attack Models Are Not Adaptive

No implemented attack currently tries to evade the defense by controlling:

```text
worker similarity
group size
attack coverage
MAD deviation
```

Therefore, current results should not be interpreted as defense against adaptive attackers.

## 12. Safe Current Attack Claims

Safe claims:

```text
The project models malicious workers as selected worker rows in the worker-object matrix.
The attack simulator modifies selected malicious workers' non-null observations.
CategoricalCollusionAttack can create a strong false-label collusion attack on Duck.
PersistentBiasedNoiseAttack can degrade Weather baseline under one tested setting.
The same attacked matrix is used for attacked baseline and attacked ours.
```

Unsafe claims:

```text
The current attacks cover all realistic malicious worker behaviors.
The current defense has been tested against adaptive attacks.
The categorical attack model fully represents real-world collusion.
The numerical attack model has been exhaustively validated.
```

