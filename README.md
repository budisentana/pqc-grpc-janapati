# Computational cost of ML-DSA signing and verification for gRPC

Benchmark code and data for the Lontar Komputer paper on the computational cost
of ML-DSA (FIPS 204) signing and verification for credentials carried in gRPC.
It contains two experiments:

| Folder | What it measures |
| --- | --- |
| `experiments/e1_crypto/` | E1: ML-DSA-65 key generation, signing and verification time (default CPU settings) |
| `experiments/e1b_signatures/` | E1b: ML-DSA-44/65/87 vs ECDSA P-256 and Ed25519, fixed CPU frequency |
| `experiments/common/` | `cpu_setup.sh` (fix/restore the CPU frequency) and `environment.txt` (test machine) |
| `interceptors/mldsa_credentials.py` | ML-DSA-65 key generation, signing and verification (liboqs-python), used by E1 |

Each experiment folder has its script(s) and a `results/` folder with the raw
data and the analysis output used in the paper.

## E1: ML-DSA-65 cost

`experiments/e1_crypto/benchmark_crypto.py` times ML-DSA-65 key generation,
signing and verification through `interceptors/mldsa_credentials.py`:

- the credential is a fixed 66-byte JSON claim set
- 10 warm-up + 500 measured iterations per operation, in the order keygen, sign, verify
- every call creates a new liboqs signer/verifier object
- the key pair and the signature used for verification are made once, outside the timed loop
- one launch = one run (new `run_id`); the paper uses 5 runs (data from 2 Nov 2026),
  under the default CPU settings (no fixed frequency)

Output: `e1_crypto/results/e1_crypto_benchmark.csv` (one row per measurement:
`experiment_id, run_id, timestamp_utc, algorithm, operation, iteration, duration_ns, status`).

## E1b: ML-DSA vs classical signatures

`experiments/e1b_signatures/benchmark_signatures.py` follows the E1 protocol
(same credential, 10 warm-up + 500 measured iterations per operation, a new
signer/verifier per call, same CSV columns) for five schemes:

- ML-DSA-44, ML-DSA-65, ML-DSA-87 (liboqs-python)
- ECDSA P-256 and Ed25519 (pyca/cryptography, OpenSSL backend); key objects are
  rebuilt from bytes on every call, matching the per-call object creation of ML-DSA

In addition to E1:

- the order of the five schemes is shuffled in every run
- the CPU governor, turbo state and core affinity are recorded per run
- runs use a fixed CPU frequency (governor `performance`, turbo off) and are pinned to core 2

Output: `e1b_signatures/results/e1b_signature_benchmark.csv` (raw measurements)
and `e1b_signatures/results/e1b_run_environment.csv` (one row per run: scheme
order, governor, turbo, affinity, library versions and key/signature sizes).
The paper data are 5 runs from 7 Nov 2026.

## Test machine

All data were collected on the machine described in
`experiments/common/environment.txt`:

- Intel Core i3-12100 (12th Gen), 4 cores / 8 threads, 7.0 GiB RAM
- Ubuntu 26.04.1 LTS, kernel 7.0.0-38-generic
- Python 3.14.4, liboqs-python 0.16.0.1, cryptography 50.0.2

## Setup

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

liboqs-python needs the liboqs C library; see the liboqs-python documentation
if it is not already installed.

## Rerunning and analysing

Run everything from the repository root with the venv active.

`experiments/common/cpu_setup.sh` writes to `/sys` and must be run with sudo
(run it yourself; the benchmark scripts do not call it). It sets the governor
to `performance` and disables turbo; `restore` sets `powersave` and turbo on.

The benchmark scripts append to the CSV files in `results/`. To collect a new
data set instead of adding to the paper data, move the existing CSV files away first.

```bash
# E1 (default CPU settings): run 5 times
python experiments/e1_crypto/benchmark_crypto.py

# E1b: fix the CPU frequency, run 5 times on core 2, then restore
sudo bash experiments/common/cpu_setup.sh
taskset -c 2 python experiments/e1b_signatures/benchmark_signatures.py
sudo bash experiments/common/cpu_setup.sh restore

# Analysis (E1b data by default, or the E1 file as argument)
python experiments/e1b_signatures/analyze_signatures.py
python experiments/e1b_signatures/analyze_signatures.py experiments/e1_crypto/results/e1_crypto_benchmark.csv
```

`analyze_signatures.py` reports, per scheme and operation, the median, mean,
p95 and p99 over all successful measurements, and the mean of the per-run means
with a 95% confidence interval (t distribution, runs - 1 degrees of freedom).
It flags runs whose mean is more than 10% away from the median run, and writes
`<name>_summary.csv` and `<name>_report.txt` next to the input file.

## License

Apache License 2.0, see `LICENSE`.
