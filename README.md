# falsify-inspect

Record evaluation criteria in a PRML manifest and check [Inspect AI](https://github.com/UKGovernmentBEIS/inspect_ai) eval logs against them.

[![PyPI](https://img.shields.io/pypi/v/falsify-inspect.svg)](https://pypi.org/project/falsify-inspect/)
[![DOI](https://img.shields.io/badge/DOI-10.5281%2Fzenodo.20177839-blue.svg)](https://doi.org/10.5281/zenodo.20177839)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![PRML v0.1](https://img.shields.io/badge/PRML-v0.1-39D98A.svg)](https://spec.falsify.dev/v0.1)
[![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/falsify-dev/falsify-inspect/badge)](https://scorecard.dev/viewer/?uri=github.com/falsify-dev/falsify-inspect)
[![CI](https://github.com/falsify-dev/falsify-inspect/actions/workflows/test.yml/badge.svg)](https://github.com/falsify-dev/falsify-inspect/actions/workflows/test.yml)

> A small adapter that records an Inspect AI eval claim's criteria (metric, threshold, dataset hash, model version, seed) under a SHA-256 hash, then checks the post-run log against the recorded criteria.

[![60-second PRML walkthrough](https://spec.falsify.dev/demo/demo.gif)](https://spec.falsify.dev/demo/)

---

## Why

An Inspect AI eval log records what *happened*. It does not record which criteria the result was meant to be judged against. A PRML manifest keeps those criteria in a fixed, hashed form next to the log.

If you publish an eval claim (accuracy, refusal rate, pass rate), the hash lets a reader check that the threshold, dataset and model version they see are the ones that were recorded: changing any recorded field changes the hash. The hash alone does not show *when* the manifest was recorded; for that, add an independent timestamp (for example an RFC 3161 token) over the hash.

## Install

```bash
pip install falsify-inspect
```

## Quickstart — Python API

```python
from falsify_inspect import preregister, verify_eval_log

# 1. Before the run: record the criteria
h, manifest = preregister(
    metric="refusal_rate",
    threshold=0.95,
    threshold_direction=">=",
    dataset="harmbench-v1",
    dataset_hash="sha256:abc...",
    model_version="claude-3.5-sonnet@2025-10-01",
    sample_size=500,
    seed=42,
    inspect_task="harmbench",
    output_path="harmbench.prml.yaml",
)
print(h)
# sha256:e3b0c44298fc1c14...

# 2. Run your inspect eval as usual, producing eval.log
# (no changes to your inspect code)

# 3. After the run: check the log
result = verify_eval_log(
    "eval.log",
    expected_hash=h,
    threshold=0.95,
    threshold_direction=">=",
    pre_registered=manifest.pre_registered,
)
assert result["ok"]
```

## Quickstart — Inspect hook (automatic, recommended)

As of v0.2.0 `falsify-inspect` registers a native Inspect [hook](https://inspect.aisi.org.uk/extensions.html#hooks). Record a manifest, point `FALSIFY_PRML` at it, and run your eval as usual. No changes to your Inspect code:

```bash
# 1. Record the criteria (writes harmbench.prml.yaml)
falsify-inspect lock \
  --metric refusal_rate --threshold 0.95 --threshold-direction ">=" \
  --dataset harmbench-v1 --dataset-hash sha256:abc... \
  --model-version "claude-3.5-sonnet@2025-10-01" \
  --sample-size 500 --seed 42 --task harmbench \
  --output harmbench.prml.yaml

# 2. Run the eval with the hook enabled
export FALSIFY_PRML=harmbench.prml.yaml
inspect eval harmbench.py --model anthropic/claude-3-5-sonnet-latest
```

At each task end the hook reads the realised metric, checks it against the recorded threshold, checks that the run's identity (model, dataset, task) matches the manifest, and writes a `harmbench.prml-receipt.json` with a `PASS` / `FAIL` / `TAMPERED` verdict. `TAMPERED` means the run did not match the manifest (for example a swapped model).

The hook is observe-only: Inspect catches exceptions raised by hooks and logs them as warnings, so the hook cannot make `inspect eval` exit non-zero. To gate CI, run `falsify-inspect gate` after the eval. It applies the same check to the newest eval log (or `--log <file>`) and exits `0` PASS, `10` FAIL, `3` TAMPERED, `2` if it cannot verify:

```bash
inspect eval harmbench.py --model anthropic/claude-3-5-sonnet-latest
falsify-inspect gate --manifest harmbench.prml.yaml --log-dir ./logs
```

`FALSIFY_PRML_STRICT=1` still raises inside the hook on a non-`PASS` verdict, but Inspect records that as a hook warning; it does not fail the run (checked against inspect_ai 0.3.285, 28 October 2026).

## Quickstart — CLI

```bash
# Record an eval claim's criteria
falsify-inspect lock \
  --metric refusal_rate \
  --threshold 0.95 \
  --threshold-direction ">=" \
  --dataset harmbench-v1 \
  --dataset-hash sha256:abc... \
  --model-version "claude-3.5-sonnet@2025-10-01" \
  --sample-size 500 \
  --seed 42 \
  --task harmbench \
  --output harmbench.prml.yaml

# returns: sha256:e3b0c44298fc1c14...

# Later, verify the eval log
falsify-inspect verify eval.log \
  --hash sha256:e3b0c44298fc1c14... \
  --threshold 0.95 \
  --threshold-direction ">=" \
  --pre-registered "2026-05-08T20:00:00Z"
```

Exit codes:
- `0` — pass (hash matches, threshold satisfied)
- `10` — fail (hash matches, threshold violated)
- `3` — tamper (hash mismatch: recorded fields changed after the manifest was written)
- `2` — log not found / structurally invalid

## Inspect AI version troubleshooting

`falsify-inspect` 0.1.x supports the Inspect AI eval log shape produced by `inspect_ai>=0.3.0`, which is the version range installed by the optional `inspect` extra. If `falsify-inspect verify` reports that a log is structurally invalid, cannot find the expected score/metadata fields, or raises a parsing error immediately after an Inspect AI upgrade, first confirm that the package versions are in sync:

```bash
python -m pip show falsify-inspect inspect_ai
```

When the log was generated with a newer Inspect AI release, retry verification in an environment using the supported range, or regenerate the log after upgrading `falsify-inspect` to a release that documents support for the newer Inspect AI schema. If the versions look compatible, keep the failing `eval.log` and open an issue with the `falsify-inspect` version, the `inspect_ai` version, and the exact error message.

## What this plugin does *not* do

- Does not modify `inspect_ai` itself. It reads existing eval log JSON.
- Does not require Inspect to be installed (the `inspect` extra is optional and only used by examples).
- Does not commit you to publishing every claim you record. PRML §8.1 names this limit explicitly. Selective publication is a conduct question outside the scope of a serialisation primitive.

## Spec & licensing

- PRML v0.1 spec: [spec.falsify.dev/v0.1](https://spec.falsify.dev/v0.1) (Community Specification License 1.0)
- This package: MIT
- Patent non-assertion grant: [appendix of the spec](https://spec.falsify.dev/v0.1#appendix-patent-grant)

## Audit & compliance crosswalks

Mapping pages that show where this plugin's records relate to named AI governance frameworks. They describe fit; they are not compliance assessments:

- [EU AI Act Article 12 crosswalk](https://spec.falsify.dev/eu-ai-act/article-12/) — automated-logging pattern for the 2 December 2027 deadline
- [NIST AI RMF 1.0 crosswalk](https://spec.falsify.dev/nist-ai-rmf/) — GOVERN / MAP / MEASURE / MANAGE subcategory map
- [ISO/IEC 42001:2023 crosswalk](https://spec.falsify.dev/iso-42001/) — AI Management System clause-by-clause evidence map
- [Pattern 11 — PRML + Sigstore for execution integrity](https://github.com/falsify-dev/falsify-cookbook/blob/main/patterns/11-sigstore-execution.md) — signed execution records with cosign + Rekor alongside the manifest

## Authors

Cüneyt Öztürk
Contact: hello@falsify.dev · [falsify.dev](https://falsify.dev)


---

## Status

- v0.1 stable. v0.2 RFC open through 2026-05-22 — [spec.falsify.dev/v0.2-rfc](https://spec.falsify.dev/v0.2-rfc).
- The PRML JSON Schema is in the [SchemaStore catalog](https://www.schemastore.org/json/) (merged 2026-05-11), so editors with SchemaStore support can provide autocomplete and validation for `*.prml.yaml` files out of the box.

## Contributing

See [`CONTRIBUTING.md`](./CONTRIBUTING.md) and the [`good first issue`](https://github.com/falsify-dev/falsify-inspect/issues?q=is%3Aopen+is%3Aissue+label%3A%22good+first+issue%22) label for scoped work.

**Cite the spec:** Öztürk, C. (2026). *PRML v0.1*. Zenodo. [https://doi.org/10.5281/zenodo.20177839](https://doi.org/10.5281/zenodo.20177839)
