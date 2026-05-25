# DATASET_CONTEXT

## 1. Current Datasets

The current repository contains the following dataset directories under `datasets/`:

```text
datasets/d_Duck_Identification/
datasets/s4_Dog_data/
datasets/weather/
```

Each dataset directory contains:

```text
answer.csv
truth.csv
```

The current default dataset in `Params.java` is Duck:

```text
sensingDataFile = "datasets/d_Duck_Identification/answer.csv"
truthFile       = "datasets/d_Duck_Identification/truth.csv"
isCategoricalData = true
```

Weather and Dog are present and can be loaded explicitly or by changing `Params.java`, but they are not the current default in `Params.java`.

Known dataset roles in the current project:

```text
Duck:
  categorical dataset
  current main attack-defense evidence for categorical collusion

Weather:
  numerical dataset
  used for early numerical biased-noise / MAD defense exploration

Dog:
  categorical dataset present in the repository
  not yet used as a main validated robustness result in current experiments
```

## 2. Data Files and Their Roles

Each dataset has two CSV files.

### answer.csv

The `answer.csv` file stores worker-submitted observations.

Observed header:

```text
question,worker,answer
```

Examples:

Duck:

```text
question,worker,answer
36618,896,0
11619,896,1
```

Dog:

```text
question,worker,answer
1,1,3
1,2,2
```

Weather:

```text
question,worker,answer
11,1,151
12,1,135
```

In the code, `question` is used as the object/exam/task ID, `worker` is the worker ID, and `answer` is the worker-provided observation or label.

### truth.csv

The `truth.csv` file stores ground-truth values for evaluation.

Observed header:

```text
question,truth
```

Examples:

Duck:

```text
question,truth
36618,0
11619,1
```

Weather:

```text
question,truth
2,130.0
3,90.0
```

Ground truth is not used by the defense modules. It is loaded for final evaluation metrics such as accuracy, RMSE, and MAE.

## 3. Main Field Meanings

The current code assumes the following fields:

```text
question:
  object / task / exam ID

worker:
  worker ID

answer:
  worker-provided observation or categorical label

truth:
  ground-truth value for the corresponding question
```

The code does not read any other fields from the dataset CSV files.

## 4. Worker-Object Observation Matrix Construction

The worker-object matrix is constructed by `DataManager.changeToMatrix(...)`.

The input format before matrix conversion is:

```text
Map<workerId, Map<questionId, answer>>
```

The output matrix format is:

```text
List<List<BigInteger>> workerObjectMatrix
```

Matrix semantics:

```text
row index    = worker index
column index = question / object / exam index
cell value   = scaled worker answer, or null if missing
```

The matrix construction process is:

```text
1. Read all worker answers from answer.csv.
2. Build a worker-to-question-to-answer map.
3. Collect all question IDs into a sorted TreeSet.
4. Store the sorted question IDs in examIds / arrayIdx2ExamID.
5. Store worker IDs in workerIds / arrayIdx2WorkerID.
6. For each worker and each question:
   - if the worker has an answer, place it in the matrix
   - otherwise place null
```

Missing observations are represented as:

```text
null
```

This missing-value representation is important. Attack modules preserve null cells; they only modify non-null observations from selected malicious workers.

## 5. Scaling and Ground Truth Representation

Worker answers are read by `DataManager.readSensingData(...)`.

The code parses each `answer` as `Double`, multiplies it by `Params.PRECISE_ROUND`, and converts it to `BigInteger`:

```text
scaledAnswer = (long)(Double.valueOf(answer) * Params.PRECISE_ROUND)
BigInteger.valueOf(scaledAnswer)
```

Current precision:

```text
Params.PRECISE_ROUND = 100000
```

This means internal worker observations are scaled integer values.

Example:

```text
raw answer = 1
internal value = 100000

raw answer = 151
internal value = 15100000
```

Ground truth is read by `DataManager.readTruthData(...)` as:

```text
Map<String, Double>
```

Ground truth values are not scaled during loading. Scaling is applied later during metric computation when needed.

For categorical evaluation, `Tool.getAccuracy(...)` converts truth values to scaled long values using `PRECISE_ROUND`, then maps predicted numerical outputs to the nearest category among ground-truth categories.

## 6. Data Cleaning and Preprocessing in Current Code

The current data loading code performs limited preprocessing:

```text
skip header rows where worker == "worker"
skip truth header rows where truth == "truth"
parse answer/truth as numeric values
scale worker answers by PRECISE_ROUND
sort question IDs using TreeSet
fill missing worker-question cells with null
```

The current code does not perform:

```text
text normalization
duplicate resolution beyond map overwrite semantics
outlier removal during loading
label remapping beyond numeric parsing
train/test split
data imputation for missing answers
```

Worker subset selection exists in `ExperimentDataset.selectWorkers(...)` and in the older `DataManager` constructor when `requiredWorkerNum > 0`.

Observed special cases:

```text
Dog / weather:
  worker IDs are sorted numerically if a worker subset is requested.

Duck:
  a fixed worker ordering is used if a worker subset is requested.
```

When `requiredWorkerNum <= 0`, all workers are used.

## 7. offline_data

The `offline_data/` directory is not a dataset directory. It stores generated offline protocol data for FPTD jobs.

Relevant parameter:

```text
Params.FAKE_OFFLINE_DIR = "./offline_data/"
```

Files in `offline_data/` are generated and consumed by the FPTD offline/online protocol execution. They are not read by `DataManager` as experimental datasets and should not be treated as raw worker-answer data.

## 8. How Attack Modules Modify Data

Attack modules operate on the in-memory `workerObjectMatrix`.

General attack flow:

```text
1. Select malicious workers according to attackerRatio and randomSeed.
2. Iterate over each worker row.
3. If the worker is honest, keep the row unchanged.
4. If the worker is malicious:
   - modify non-null observations according to the attack strategy
   - preserve null cells
5. Return attackedMatrix plus masks.
```

AttackResult contains:

```text
attackedMatrix
attackMask
honestMask
maliciousWorkerMask
```

### Categorical Collusion Attack

Used mainly for Duck.

It modifies selected malicious workers' categorical labels. For each object, malicious workers are pushed toward a coordinated target label when possible.

This creates:

```text
fixed malicious workers
→ coordinated false labels
→ false consensus in attacked worker-object matrix
```

### Persistent Biased Noise Attack

Used for early Weather numerical exploration.

It modifies selected malicious workers' numerical observations:

```text
malicious_value = original_value + scaled_bias + scaled_random_noise
```

The bias and noise are specified in real units and scaled internally by `Params.PRECISE_ROUND`.

### Existing Numerical Attacks

The repository also contains numerical attacks such as random noise, collusion shift, and extreme outlier attacks. Their values operate on the already scaled internal `BigInteger` observations unless the specific attack class rescales values internally.

This distinction matters. A shift of `50` in scaled space is much smaller than a real-unit shift of `50`.

## 9. How Defense Modules Process Data

Defense modules also operate on the worker-object matrix.

The current `ours` pipeline is:

```text
input matrix
→ robust bootstrap
→ gateway / filtering
→ minimum support constraint
→ prepared matrix
→ original FPTD
```

### Robust Bootstrap

Default:

```text
MedianBootstrap
```

For each object:

```text
collect non-null observations
sort
take median as pilotTruth
record support count
```

Trimmed mean bootstrap exists but is not the default in current experiments.

### Categorical Defense

For categorical datasets, current `PipelineFactory.enhanced(...)` uses:

```text
CategoricalCollusionGateway
```

This gateway detects suspicious worker groups based on cross-task label agreement.

Current hard-rule logic:

```text
pairwise agreement >= 0.98
common tasks >= 10
group size in [20%, 50%] of worker count
→ suspicious collusive group
```

Suspicious workers are filtered from the matrix before original FPTD.

### Numerical Defense

For non-categorical datasets, current `PipelineFactory.enhanced(...)` uses:

```text
MadGateway
```

MAD filtering compares each observation against the pilot truth for that object:

```text
deviation = |value - pilotTruth|
accept if deviation <= madCoefficient * MAD
```

The default `madCoefficient` is currently `3.0`. Early Weather biased-noise results indicate that this default can be too loose for some attacks; stricter values such as `0.5` recovered RMSE in one tested setting but filtered aggressively.

### Minimum Support

After gateway filtering, `MinimumSupportConstraint` can restore observations if an object has too few active observations.

It restores inactive observations closest to pilot truth until:

```text
active observations >= minimumSupport
```

Current default:

```text
minimumSupport = 3
```

This module preserves data availability but is not the main attack detector.

## 10. Dataset Suitability for Experimental Claims

### Duck

Duck is the current default and the strongest validated categorical robustness case.

Suitable for claims about:

```text
categorical truth discovery
binary false-label collusion
worker-level false consensus
collusion-aware filtering
attack ratio robustness under highly consistent malicious workers
```

Current supported evidence:

```text
clean baseline accuracy ≈ 0.759
30% categorical collusion baseline accuracy ≈ 0.500
30% categorical collusion ours accuracy ≈ 0.769
40% categorical collusion baseline accuracy ≈ 0.259
40% categorical collusion ours accuracy ≈ 0.769
```

Limitations:

```text
binary categorical setting
current attack is highly coordinated
not enough to prove general categorical robustness
```

### Weather

Weather is numerical and suitable for numerical robustness experiments.

Suitable for claims about:

```text
numerical false data injection
random / biased numerical noise
MAD-based observation filtering
RMSE / MAE robustness
```

Current early evidence:

```text
clean baseline RMSE ≈ 29.86
30% persistent biased-noise baseline RMSE ≈ 93.82
MAD coefficient 0.5 ours RMSE ≈ 29.94
```

Limitations:

```text
Weather numerical branch is preliminary
default MAD coefficient 3.0 did not recover in the biased-noise setting
strict MAD coefficient 0.5 filters aggressively
needs MAD sweep and clean side-effect analysis
```

### Dog

Dog is present and appears categorical based on `Params.java` comments and numeric class-like labels in the CSV. However, the current strongest experiments have not used Dog as a validated robustness result.

Suitable future use:

```text
additional categorical dataset
possible generalization beyond Duck
potential multi-class categorical robustness depending on label distribution
```

Current status:

```text
present in repository
not yet part of the main supported evidence
```

## 11. Safe Claims Based on Current Data Handling

Safe current claims:

```text
The project converts raw answer.csv files into a worker-object matrix with null missing values.
Worker answers are scaled by PRECISE_ROUND and stored as BigInteger.
Ground truth is loaded separately and used for evaluation.
Attack modules modify selected malicious workers' non-null observations.
Defense modules operate on the attacked or clean worker-object matrix before original FPTD.
Duck supports current categorical collusion robustness evidence.
Weather supports early numerical biased-noise exploration.
```

Claims not yet supported:

```text
All datasets support the same robustness conclusions.
Duck results generalize to all categorical datasets.
Weather MAD defense is robust under all numerical noise settings.
Dog has validated attack-defense results.
Ground truth is used by defense modules.
offline_data contains raw dataset observations.
```

