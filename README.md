# TrustLadder

A "we measured it and enforcement works" result that's actually fabricated is worse than no result at all: it tells every team downstream to trust a guardrail that doesn't hold. I ran a study to test whether forcing an AI coding agent through a real enforcement gate cuts its shipped-defect rate — and I ran it against my *own* governance kit, which is the maximum-conflict-of-interest version of that question. So I built the measurement engine to refuse to produce a confirmatory number until a validity check has actually passed. It fails closed against its own author.

That mattered. Before I froze the study, the apparatus turned out to have validated every component in isolation but never run one real record through the full pipeline. The first end-to-end pass surfaced a defect (D3) where the runner and the grader hashed the final code tree with incompatible algorithms — the integrity guard refused 100% of real records. I had a confirmatory number within reach. I halted grading and escalated to an independent methodologist rather than hand-assemble a result around a broken pipeline. Then I encoded that discipline into the code so it can't be bypassed:

```python
def run_confirmatory(workspace, cfg, log=print):
    verdict = _require_valid_verdict(workspace)   # FIRST ACT — raises unless
                                                  # validity_verdict.json reads status=VALID
```

## What's in this repo

The offline **measurement-and-analysis engine** — stdlib-only Python, no network, no secrets. It runs end-to-end on a committed synthetic fixture:

```bash
pip install -e .                              # Python 3.10+, zero runtime deps
trustladder-mini-run --workspace /tmp/mini    # sign → grade behind calibration gate
                                              # → merge verdict → verify chain → aggregate
```

That drives the whole chain on a stub agent: signs a hash-chained run-record, grades it blind behind a calibration gate, merges the verdict back in (the signature still verifies, because grading-mutable fields are excluded from the record hash), and aggregates per-arm escape rates via stdlib `sqlite3`.

The load-bearing pieces, all hand-rolled:
- **Statistics with no numpy/scipy** (`analysis/stats.py`): BCa bootstrap (Efron 1987), Newcombe (1998) MOVER-Wilson paired CI, Wilson score, Acklam inverse-normal PPF, Cohen's kappa, and a three-outcome decision rule against a fixed floor.
- **Signed, append-only run-records** (`schema/signing/`): Ed25519 via `cryptography`, with an `openssl` CLI fallback so signing works with no third-party deps.
- **A blind, calibration-gated grader** (`grading/`): the instrument must score known-defective artifacts RED and known-clean GREEN *before* it's allowed to grade real runs; verdicts record `blind_to_arm=true`.
- **A regression test for the D3 defect** (`tests/test_grading_seam.py`): reproduces the exact hashing mismatch the original freeze never exercised.

Suite: 29 test functions across 5 files, ruff-linted, CI on Python 3.10 and 3.12.

## What's deliberately not here

The live four-arm agent-dispatch layer (L0 / L1 / SHAM / L3) and the seeded task battery are excluded — they shell out to a private kit and would spoil the benchmark's answer keys. The design is in [ARCHITECTURE.md](ARCHITECTURE.md) and [METHODOLOGY.md](METHODOLOGY.md).

The 72 real subject runs are **private and not reproducible from this repo** — they carry machine paths and session identifiers, and the independent-methodologist ruling is private. The full account of what broke and why I stopped is in [RESULTS.md](RESULTS.md).

One honest caveat: the demo's `outcome=CONFIRMED` line is **synthetic** — a planted dataset (`analysis/dummy.py`) that proves the estimator and decision rule are wired correctly, *not* a finding about real agents. Run `trustladder-analyze dummy --scenario refuted` to watch the same pipeline correctly decline ([RUNNING.md](RUNNING.md) has the full invocation).

## Deeper

[RUNNING.md](RUNNING.md) walks the demo step by step. [ARCHITECTURE.md](ARCHITECTURE.md) is the component map; [GLOSSARY.md](GLOSSARY.md) defines the vocabulary. Licensed [Apache-2.0](LICENSE).
