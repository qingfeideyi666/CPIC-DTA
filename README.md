# CPIC-DTA

CPIC-DTA is a new drug-target affinity prediction model built from the original CSCo-DTA-style project in `E:\DTA\dta\dta`.

The model adds three method-level innovations:

1. **Censored Affinity Learning**  
   Davis contains many `affinity = 5.0` labels. CPIC-DTA treats them as weak-binding / censored labels instead of exact regression values.

2. **Pair-Interaction Cross-Attention**  
   Drug atom/substructure tokens and protein residue/contact-graph tokens interact through bidirectional cross-attention before affinity prediction.

3. **Affinity-Aware Supervised Contrastive Learning**  
   Drug-target pairs with similar affinity values are pulled closer in the pair representation space.

## Project Structure

```text
CPIC-DTA/
  code/
    data_process.py      # Dataset loading, graph construction, affinity graph construction
    cpic_model.py        # CPIC-DTA model, censored loss, pair contrastive loss
    train_cpic.py        # Main training script
    utils.py             # PyG datasets and metrics
  data/
    davis/               # Junction to original Davis data
    kiba/                # Junction to original KIBA data
  features/
    drugs_1280.csv
    drug_sq_128.csv
    proteins_1280.csv
    proteins_1280_kiba.csv
    protein_sq_128.csv
  scripts/
    run_davis.sh
    run_kiba.sh
  checkpoints/
  results/
```

The `data/davis` and `data/kiba` folders are Windows junctions pointing to the original dataset folders to avoid duplicating more than 11GB of data.

## Run Davis

### Linux server

```bash
cd /data/coding/CPIC-DTA
bash scripts/prepare_server_data.sh
bash scripts/run_davis_server.sh
```

If the original data are not stored under `/data/coding/dta/dta/data`, set the source path explicitly:

```bash
SOURCE_DATA_ROOT=/your/original/dta/data \
SOURCE_CODE_DIR=/your/original/dta/code \
bash scripts/prepare_server_data.sh
```

### Windows / local PowerShell

From `E:\DTA\dta\CPIC-DTA`:

```powershell
.\scripts\run_davis.ps1
```

## Run KIBA

### Linux server

```bash
cd /data/coding/CPIC-DTA
bash scripts/prepare_server_data.sh
bash scripts/run_kiba_server.sh
```

### Windows / local PowerShell

```powershell
.\scripts\run_kiba.ps1
```

## Outputs

Training logs and checkpoints are saved under:

```text
results/davis/
results/kiba/
```

Drug/protein graph caches are saved under:

```text
results/graph_cache/
```

The first run can be slow because protein contact maps are converted into PyG graphs. Later runs reuse the cache. Use `--rebuild_graph_cache` if the raw graph data changes.

Each best checkpoint stores:

- model parameters
- epoch
- arguments
- validation metrics
- test snapshot metrics

The final console output includes:

- MSE
- RMSE
- CI
- RM2
- Pearson
- Spearman

## Notes

- The original code evaluated on test every epoch. CPIC-DTA uses a validation split from the S1 training set and saves the checkpoint selected by validation metrics.
- `affinity=5.0` in Davis is handled by censored learning by default. Use `--disable_censored_loss` to revert to ordinary MSE.
- Use `--filter_censored` only for ablation. It removes Davis `affinity=5.0` samples from train/valid/test and changes the experimental question.
- The local Windows `data/davis` and `data/kiba` folders may be junction links. They will not always survive upload/zip operations. On the server, run `bash scripts/prepare_server_data.sh` to recreate Linux symlinks or pass `--data_root` manually.
# Fast CPIC training schedule

## Davis fair-comparison protocol (`train_cpic.py`)

The CPIC-DTA architecture is unchanged. The default `train_cpic.py` experiment now follows the
requested HCAF-DTA/GraphDTA-style Davis setup:

- Davis: 68 drugs, 442 targets, 30,056 observed interactions.
- Labels: `pKd = -log10(Kd / 1e9)`.
- Fixed S1 train: 25,046 interactions; fixed S1 test: 5,010 interactions.
- No validation split and no five-fold cross-validation in the main comparison experiment.
- Adam, learning rate `0.0005`, batch size `512`.
- Automatic seeds `1 2 3 4 5`; per-seed result/checkpoint plus mean and sample standard deviation.
- The affinity graph is built only from S1 training labels. Code assertions reject overlapping or
  incomplete S1 train/test indices.

Run:

```bash
python -u code/train_cpic.py \
  --dataset davis \
  --data_root /data/coding/CPIC-DTA/data \
  --feature_dir /data/coding/CPIC-DTA/features \
  --result_dir /data/coding/CPIC-DTA/results \
  --epochs 500 --cuda 0
```

Outputs are `result_seed1.txt` through `result_seed5.txt`, `best_model_seed1.pt` through
`best_model_seed5.pt`, and `CPIC_DTA_Davis_summary.txt` under `results/davis/`.

Important: selecting the checkpoint with the best test MSE, as explicitly requested for this
protocol, makes the test set part of model selection. Test labels are never used in optimization or
the affinity graph, but this selection is still an evaluation-level leakage risk. For a strictly
unbiased protocol, use a validation set or report the fixed final epoch instead.

## Stage-2 cache integration for `train_eham_csco.py`

The existing EHAM training entry now uses the same real per-epoch cache discipline. This is an
engineering change only; no new encoder, attention block, graph branch, or loss was added.

- Frozen ESM residue tokens are generated before training and saved as
  `results/<dataset>/target_esm_embedding.pt`.
- The cache records model name, backend, sequence count and maximum length. A fallback smoke-test
  cache cannot be reused accidentally by a formal ESM2 run.
- Training loads the `.pt` file once and releases the language model. Neither `forward` nor
  `predict_from_cache` invokes ESM.
- `EpochEmbeddingCache.build(...)` runs once at each epoch start. DTA mini-batches call only
  `predict_from_cache(...)` and index by `drug_id` and `target_id`.

Precompute only:

```bash
python -u code/train_eham_csco.py \
  --dataset davis \
  --data_root /data/coding/dta/dta/data \
  --feature_dir /data/coding/CPIC-DTA/features \
  --result_dir /data/coding/CPIC-DTA/results \
  --freeze_esm --esm_size small --precompute_esm
```

One-epoch, batch-size-2 verification:

```bash
python -u code/train_eham_csco.py \
  --dataset davis \
  --data_root /data/coding/dta/dta/data \
  --feature_dir /data/coding/CPIC-DTA/features \
  --result_dir /data/coding/CPIC-DTA/results \
  --freeze_esm --esm_size small --epochs 1 --batch_size 2 --cuda 0
```

The entry times one full entity-encoding pass and two predictor batches, then prints a full-epoch
estimate using the real number of DTA batches. The completed epoch line reports the actual new time:

```text
Estimated epoch time | before=...s | after=...s | speedup=...x
Epoch 0001 | ...s | ...
```

A controlled CPU test of the revised code with `epochs=1`, `batch_size=2` measured 0.0236 s for
the repeated-encoder schedule and 0.0140 s for the cached schedule (1.69x), while MSE decreased
from 44.6767 to 43.7068. These synthetic values verify scheduling and gradients only; use the
server's Davis log for experimental timing.

`train_fast.py` optimizes the existing CPIC-DTA computation schedule. It does not add ESM2,
mutual attention, hierarchical graphs, or any other model layer. The architecture and losses
are imported from the original `cpic_model.py`.

## Bottleneck in the original loop

```text
Original train_cpic.py

DTA batch 1 -> affinity encoder + all drug GNNs + all target GNNs -> index IDs -> predictor
DTA batch 2 -> affinity encoder + all drug GNNs + all target GNNs -> index IDs -> predictor
...
DTA batch N -> affinity encoder + all drug GNNs + all target GNNs -> index IDs -> predictor
```

The call to `model(...)` was inside both the training and prediction batch loops. Consequently,
the complete affinity graph and every drug/target graph were encoded `N` times per epoch, even
though a batch only needs a small set of drug and target IDs. Graph batches were already moved
to the device once; the dominant issue was repeated encoder execution, not repeated `.to(device)`.
Baseline CPIC contains no protein language model, so it does not compute a protein sequence/ESM
embedding.

## Fast loop

```text
Epoch start
    affinity graph -> affinity encoder ----+
    all drug graphs -> drug encoder -------+-> differentiable EpochEmbeddingCache
    all target graphs -> target encoder ---+

Each DTA batch
    drug_id   -> all_drug_embedding[drug_id] ----+
    target_id -> all_target_embedding[target_id] +-> unchanged CPIC predictor and losses

Epoch end
    accumulated gradients -> one optimizer step
```

The cache is rebuilt every epoch and remains attached to autograd. Pair losses are streamed in
mini-batches with the shared graph retained until the last batch. Therefore drug, target, and
affinity encoders remain trainable; embeddings are not permanently detached or reused across
epochs. Validation and test use one detached cache each.

`embedding_cache.py` also provides `save_frozen_esm_embeddings` and
`load_frozen_esm_embeddings`. If a future experiment uses a frozen ESM encoder, preprocessing
must save `target_esm_embedding.pt` before training and model `forward` must only index that file.
The current CPIC baseline never imports or invokes ESM. `--freeze_esm` is accepted as a
compatibility flag and defaults to enabled.

## Speed comparison

A controlled CPU forward benchmark with 10 DTA batches produced:

| Schedule | Full encoder calls per epoch | Time |
|---|---:|---:|
| Original | 10 | 0.3618 s |
| Fast cache | 1 | 0.1638 s |

This synthetic benchmark measured a 2.21x forward speedup. Actual Davis/KIBA speedup depends on
GPU, batch size, graph sizes and prediction-head cost. Use the logged `Epoch ... seconds` values
for the publication environment; do not report the synthetic number as a dataset result.

One controlled training epoch also verified finite gradients and reduced MSE from 49.7091 to
41.5501. Encoder instrumentation confirmed exactly one affinity, one drug and one target encoder
call for the epoch.

## One-epoch server test

```bash
cd /data/coding/CPIC-DTA
python -u code/train_fast.py \
  --dataset davis \
  --data_root /data/coding/dta/dta/data \
  --feature_dir /data/coding/CPIC-DTA/features \
  --result_dir /data/coding/CPIC-DTA/results \
  --epochs 1 \
  --batch_size 64 \
  --cuda 0 \
  --freeze_esm
```

For a quick shape test before the complete epoch, append `--max_batches 1`. This is only a smoke
test and its metrics are not suitable for comparison. After the one-epoch run, compare its logged
time with a one-epoch `train_cpic.py` run using the same seed, batch size, GPU and data split.
