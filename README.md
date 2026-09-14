# Odyn multi-ventor prediction engine

This repository demonstrates a repeatable cross-OEM training workflow: train on one vendor's hardware, transfer checkpoint artifacts, and resume on a different machine or vendor, while profiling key metrics along the way.

Analytical models of AI/ML training cost exist, but the noise in real systems is too large for them to be deterministic. Instead, we profile a representative sample with a small set of key metrics and fit a regression to the observed behaviour. The working assumption is that the relevant latent factors (hardware characteristics, interconnect, workload shape) can be captured by a low-dimensional representation that the regressor learns from the profiled data.

**What the repository covers:**

- Train, transfer checkpoint artifacts, and resume on a different machine or vendor
- Profile runtime and hardware metrics during training
- Fit a regression model on those metrics to predict training time
- Hardware exercised so far: AMD MI300X, AMD Radeon, and NVIDIA DGX Spark, connected via QSFP

The current scope is intentionally sequential, with a single active training phase at a time. This is a deliberate choice to prove portability and recovery semantics first; data-parallel and multi-job orchestration can be layered on top later.

## Prerequisites

- Python 3.12
- SSH access to participating machines (key-based authentication preferred)
- `conda` or another environment manager

Optional:

- `sshpass` (only if password-based SSH is unavoidable)
- Vendor GPU telemetry libraries (`nvidia-ml-py` and/or the AMD SMI Python bindings)

## Quickstart

```bash
python3 --version
conda create -y -n cross-oem-migration python=3.12
conda activate cross-oem-migration

dagster dev -w workspace.yaml
```

Expected output includes:

```text
Serving dagster-webserver on http://127.0.0.1:3000
```

Open `http://127.0.0.1:3000`, materialize the job/assets, and inspect the run metadata.

<img width="1348" height="930" alt="Dagster UI showing the cross-OEM migration job" src="https://github.com/user-attachments/assets/8426355f-8115-4d16-a444-ef70170f92f1" />

## Which metrics, and why?

Metric selection is currently the most important part of this repository. The goal is a set of comparable, normalised metrics across NVIDIA and AMD hosts.

Core metrics:

- Training tokens processed
- Runtime (seconds)
- Throughput (tokens/second)
- Average and peak GPU power
- GPU energy (joules)
- Efficiency (tokens/joule)
- Checkpoint transfer throughput

Suggested packages:

- Host metrics: `psutil`
- Export format: `prometheus-client`
- NVIDIA: `nvidia-ml-py`
- AMD: ROCm AMD SMI Python bindings

On NVIDIA, sample NVML counters at a fixed interval and integrate power over time to obtain energy. On AMD, use the AMD SMI equivalents; field names differ between ROCm versions, so they should be verified on the target environment.

---

Two things worth checking on your side: the Quickstart creates the conda environment but never installs the project's dependencies (no `pip install -r requirements.txt` or `pip install -e .` step) before running `dagster dev`, so a fresh user will likely hit an import error. And I rephrased the "hope the regressor has suitably PCA'd" line as a stated working assumption rather than removing it entirely, since it's an honest caveat about the method; tighten or expand it depending on how much you want to claim.
