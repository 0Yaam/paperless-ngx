# PBT-03 Cycle 4: Final Report, Clean-Machine Reproduction & Defense Demo Runbook

## Metadata & Ownership

- **Issue**: [#50 — [Cycle 4][Vương] Final report & demo PBT-03 Path security](https://github.com/0Yaam/paperless-ngx/issues/50)
- **Owner**: Phan Khánh Vương (`@vuong123s`, MSSV `2312802`)
- **Peer Reviewer**: Phùng Nguyễn Hoài Bo (`@HubertPhung`, MSSV `2312585`)
- **Suite**: `PBT-03` — Path Containment Security
- **Target**: `documents.file_handling.validate_path_in_root(path: Path, root: Path) -> None`
- **Exception**: `documents.file_handling.UnsafeFilePathError`
- **Pinned Fork Baseline**: `dev@b29fec117053637428a935d43af16d9e1d65c1fb` (and Cycle 4 opener `dev@0944d035255e9c6329ac74282661c9672240a5d5`)
- **Upstream Baseline**: `da3de299f`
- **Prior Cycle Artefacts**:
  - Cycle 1 Design: [`pbt-03-path-security-design.md`](pbt-03-path-security-design.md)
  - Cycle 2 Implementation PR: [#21](https://github.com/0Yaam/paperless-ngx/pull/21) (`20edae480`)
  - Cycle 3 Evidence & RCA: [`pbt-03-cycle-3-evidence.md`](pbt-03-cycle-3-evidence.md) (PR [#40](https://github.com/0Yaam/paperless-ngx/pull/40) `cb087a99b` + deterministic CI `#42` `2fa7b2dec`)

---

## 1. Final Report Section (Aligned with Course Structure A–H)

### Part A — System Role & Data Flow Context

Paperless-ngx ingests documents via Web UI, REST API (`POST /api/documents/post_document/`), mail rules, and the `consume/` directory (`UC-01`), indexes extracted text in Tantivy (`UC-02`), and serves archived or original files via `serve_file` (`UC-03`).

During document storage and consumption (`documents.consumer` and `documents.file_handling`), destination paths derived from filenames, storage path templates, and metadata must never escape the configured root directory (`ORIGINALS_DIR` or `ARCHIVE_DIR`). The target guard `validate_path_in_root` enforces filesystem containment prior to moving or writing files.

### Part B — Architecture & Target Module Boundary

- **Target Module**: `src/documents/file_handling.py` (lines `15–29`).
- **Mechanism**:
  1. Resolves both candidate `path` and `root` to canonical physical paths via `Path.resolve()`, eliminating `.` / `..` segments and resolving symlinks.
  2. Verifies structural path-component containment via `Path.is_relative_to()` instead of naive prefix string matching.
  3. Logs a warning on `paperless.filehandling` and raises `UnsafeFilePathError` whenever containment is violated.

### Part C — Environment & Verified Execution Baseline

- **Pinned Fork Baseline**: `dev@b29fec117053637428a935d43af16d9e1d65c1fb` (also verified on `dev@0944d0352`).
- **Locked Dependencies (`uv.lock`)**: `Python 3.12` (CI) / `3.13.5` (Local `.temp_test_venv`), `pytest 9.1.1`, `hypothesis 6.168.5`, `pytest-cov 7.1.0`, `pytest-django 4.14.0`, `coverage.py 7.17.2`.
- **Deterministic Seed & Profile**: `HYPOTHESIS_PROFILE=recorded` (`max_examples=250`, `deadline=None`, `suppress_health_check=[HealthCheck.function_scoped_fixture]`), `--hypothesis-seed=20261008`, `--cov=documents.file_handling --cov-branch`, `-n 0`.
- **Verified CI Run**: [Coursework CI Run #45791632304](https://github.com/0Yaam/paperless-ngx/actions/runs/45791632304) (`dev@0944d0352`, `2026-11-26T09:24:16Z -> 09:25:00Z`, job `PBT-03 path security` — **success** in `44s`).

### Part D — K01 PBT Rationale, Scope, Risks & Pass/Fail Criteria

- **Why PBT fits `validate_path_in_root`**: Path handling involves combinatorial depth, arbitrary nested prefixes, prefix-confusion siblings (`storage_root_rogue`), traversal chains (`..`), and symlink targets across OSes that manual examples easily miss.
- **Assumptions & Limitations**:
  - Bounded portable ASCII component generator (`_SAFE_COMPONENT_STRATEGY`, length `1..20`, prefixed with `p_`) guarantees OS portability across Windows and Linux.
  - Filesystem ACLs and TOCTOU race conditions outside `pathlib.Path.resolve()` are out of scope.
  - Symlink properties (`PBT03-INV3A/B`) use `pytest.skip` only when the host OS denies symlink creation (on Windows without Developer Mode/Administrator privileges); Ubuntu CI and Developer-Mode Windows execute all 500 symlink examples.
- **Pass/Fail Criteria**:
  - **PASS**: All 5 properties complete `max_examples=250` with `0` assertion failures, expected `UnsafeFilePathError` raised for all out-of-root paths, and `0` partial branches on `validate_path_in_root`.
  - **FAIL**: Any valid example triggers an unexpected exception or fails to raise `UnsafeFilePathError` on traversal/escape.

### Part E — Test Model, Invariants & Boundary `@example` Cases

Suite `src/documents/tests/test_path_security.py` implements **5 independent properties** plus **8 deterministic `@example` cases**:

| ID          | Test Method                                   | Invariant                                                             | Bounded Strategy                                                            | Oracle & Boundary `@example`                                                                             |
| :---------- | :-------------------------------------------- | :-------------------------------------------------------------------- | :-------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- |
| PBT03-INV1  | `test_descendants_inside_root_accepted`       | All valid descendants inside `root` are accepted.                     | `st.lists(_SAFE_COMPONENT_STRATEGY, min_size=1, max_size=5)`                | Returns normally; `target.resolve().is_relative_to(root.resolve())`. Ex: `["p_single_file.pdf"]`.        |
| PBT03-INV2A | `test_traversal_outside_root_rejected`        | Multi-level `..` traversal escaping `root` raises error.              | `prefix` (0–5) + `[".."] * (len(prefix) + extra_depth)` (1–8) + `filename`. | Raises `UnsafeFilePathError`; resolved target outside root. Ex: `([], 1, "p_passwd")`.                   |
| PBT03-INV2B | `test_prefix_confusion_outside_root_rejected` | Sibling dir `storage_root_<suffix>` sharing string prefix is outside. | `sibling_suffix` (`_SAFE_COMPONENT_STRATEGY`) + 1–4 parts.                  | Raises `UnsafeFilePathError`. Ex: `("rogue", ["p_secret.pdf"])`, `("_backup", ["p_sub", "p_file.png"])`. |
| PBT03-INV3A | `test_symlink_escaping_root_rejected`         | Symlink inside `root` pointing outside `root` is blocked.             | Secret file under `tmp_path/outside/`, symlink inside `root/`.              | Raises `UnsafeFilePathError`. Ex: `"p_sensitive_outside.txt"`.                                           |
| PBT03-INV3B | `test_symlink_internal_to_root_accepted`      | Symlink inside `root` pointing to internal file is accepted.          | Target file inside `root/`, symlink inside `root/`.                         | Returns normally; resolved link inside root. Ex: `"p_internal_target.txt"`.                              |

### Part F — Automation Harness & CI Integration

- **Test File**: `src/documents/tests/test_path_security.py` (centralized profile registration for `dev`, `ci`, `recorded`, `stress` controlled via `HYPOTHESIS_PROFILE`).
- **Workflow Job**: `.github/workflows/coursework-ci.yml` (`pbt-03`), running sequentially (`-n 0`) on `ubuntu-latest` with Python 3.12 (`uv sync --python 3.12 --group testing --frozen`) and uploading `.artifacts/pbt-03-run.log` as artifact `pbt-03-evidence`.

### Part G — Cycle 3 Measurements, Clean-Checkout Coverage & RCA Summary

1. **Clean-Checkout Verification Statistics (`dev@b29fec117`, detached worktree, `2026-11-26`)**:
   Executing the focused command inside a fresh detached worktree of `b29fec117053637428a935d43af16d9e1d65c1fb` produced **5/5 PASSED in 13.56s** (`1,250` valid examples + `8` explicit `@example`, `0` failures):

   | Property                                      | Passed | Failed | Invalid (Dedup) | Generate Phase | Call Duration |
   | :-------------------------------------------- | -----: | -----: | --------------: | -------------: | ------------: |
   | `test_descendants_inside_root_accepted`       |    250 |      0 |              43 |         1.43 s |        2.94 s |
   | `test_traversal_outside_root_rejected`        |    250 |      0 |              25 |         1.74 s |        1.76 s |
   | `test_prefix_confusion_outside_root_rejected` |    250 |      0 |              35 |         2.67 s |        2.70 s |
   | `test_symlink_escaping_root_rejected`         |    250 |      0 |               0 |         1.52 s |        1.53 s |
   | `test_symlink_internal_to_root_accepted`      |    250 |      0 |               0 |         1.18 s |        1.20 s |
   | **Total (`PBT-03`)**                          |  1,250 |      0 |             103 |         8.54 s |       13.56 s |

   _(The `103` invalid counts (< 7.6%) come from Hypothesis internal input deduplication across `st.lists`/`st.text`; the suite does not use `.filter()` or `assume()`.)_

2. **Deterministic Branch Coverage (`coverage.py --branch`)**:

   ```text
   Name                             Stmts   Miss Branch BrPart  Cover   Missing
   ----------------------------------------------------------------------------
   src\documents\file_handling.py      96     76     44      0    16%   33, 37-62, 77-122, 126-153, 164-216
   ----------------------------------------------------------------------------
   ```

   - **Target `validate_path_in_root` (`lines 21–29`) & `UnsafeFilePathError` (`lines 15–18`)**: **100% Statement Coverage (10/10 lines)** and **100% Branch Coverage** (`0` missing lines in range `15–30`, `BrPart = 0`). Both `True` (rejection + warning log + exception) and `False` (containment acceptance) decision outcomes are exercised. Uncovered lines (`33`, `37-62`, `77-216`) belong to sibling file-handling helpers outside the `validate_path_in_root` target scope.

3. **Counterexample Shrinking Verification & Defense-in-Depth RCA** (from Cycle 3 evidence [`pbt-03-cycle-3-evidence.md`](pbt-03-cycle-3-evidence.md)):
   - **Fault-Injection Shrinking Proof**: Removing `.resolve()` in a simulated defective implementation (`if not path.is_relative_to(root):`) immediately fails `test_symlink_escaping_root_rejected` and shrinks to the minimized counterexample `filename='p_0'`.
   - **Defense-in-Depth Finding (`delete_empty_directories`)**: Code inspection during RCA found that sibling cleanup helper `delete_empty_directories(directory, root)` (`src/documents/file_handling.py:36–62`) checks `if not directory.is_relative_to(root):` lexically **without** calling `.resolve()`. If a symlink inside `root` resolves outside `root`, recursive cleanup risks traversing outside the intended directory boundary. Recommendation: normalize with `directory.resolve().is_relative_to(root.resolve())` or invoke `validate_path_in_root(directory, root)`.

### Part H — Traceability & Artifact Inventory

| Artefact / Run                        | Reference / Commit / ID                             | Purpose                                                |
| :------------------------------------ | :-------------------------------------------------- | :----------------------------------------------------- |
| Pinned Fork Baseline                  | `dev@b29fec117053637428a935d43af16d9e1d65c1fb`      | Clean checkout verification baseline                   |
| Cycle 4 Opener Commit                 | `dev@0944d035255e9c6329ac74282661c9672240a5d5`      | Updated Cycle 4 status and reproduction commands       |
| Coursework CI Run (All 5 Suites Pass) | Run `#45791632304` (`PBT-03 path security` — `44s`) | Authoritative Ubuntu Python 3.12 run & artifact upload |
| CI Log Artifact                       | `pbt-03-evidence` (`.artifacts/pbt-03-run.log`)     | Full pytest stdout, branch coverage, Hypothesis stats  |
| Local Test Suite                      | `src/documents/tests/test_path_security.py`         | Executable Hypothesis PBT suite                        |

---

## 2. Clean-Machine Reproduction Steps

### Prerequisites

- Git (`>= 2.40`)
- Astral `uv` (`0.12.x`) + Python `3.12` (Linux / macOS / WSL2 / Docker; recommended for locked `uv.lock` compatibility)

### Step-by-Step Reproduction (Linux / macOS / WSL2)

```bash
# 1. Fresh clone pinned to the Cycle 4 fork baseline
git clone https://github.com/0Yaam/paperless-ngx.git
cd paperless-ngx
git checkout b29fec117053637428a935d43af16d9e1d65c1fb

# 2. Install locked dependencies cleanly from uv.lock
uv sync --python 3.12 --group testing --frozen

# 3. Execute PBT-03 deterministically with recorded profile & branch coverage
HYPOTHESIS_PROFILE=recorded uv run --python 3.12 --dev --frozen pytest \
  src/documents/tests/test_path_security.py -k TestPathSecurity -n 0 -v \
  --hypothesis-seed=20261008 --hypothesis-show-statistics \
  --cov=documents.file_handling --cov-branch --cov-report=term-missing
```

### Windows Reproduction (PowerShell with Pre-Synced Testing Venv)

Because `uv.lock` constrains certain Linux/macOS runtime wheels (`sys_platform`), Windows machines using an existing synced venv run:

```powershell
git worktree add --detach ../paperless-pbt03-clean b29fec117053637428a935d43af16d9e1d65c1fb
Push-Location ../paperless-pbt03-clean
$env:PYTHONPATH = (Resolve-Path "src").Path
$env:DJANGO_SETTINGS_MODULE = "paperless.settings"
$env:PAPERLESS_SECRET_KEY = "test-secret-key-for-pbt-tests"
$env:HYPOTHESIS_PROFILE = "recorded"

# Run with deterministic seed & branch coverage
python -m coverage run --branch --source=documents.file_handling `
  -m pytest --no-cov src/documents/tests/test_path_security.py -k TestPathSecurity `
  -n 0 -v --hypothesis-seed=20261008 --hypothesis-show-statistics
python -m coverage report -m
Pop-Location
```

### Expected Output & Statistics

- **Pytest summary**: `5 passed` (`100%`)
- **Hypothesis statistics**: `250 passing, 0 failing` for each of the 5 tests (`1,250` total passing generated examples + `8` explicit `@example` boundary cases); `43 / 25 / 35 / 0 / 0` deduplicated invalid cases.
- **Branch Coverage**: `BrPart = 0`, no lines in `15–29` (`UnsafeFilePathError` and `validate_path_in_root`) listed under `Missing`.
- **GitHub Actions CI Artifact**: Download `pbt-03-evidence` (`.artifacts/pbt-03-run.log`) from workflow **Coursework CI** $\rightarrow$ job **PBT-03 path security**.

---

## 3. Final Defense Demo Runbook (3–5 Minutes)

Use this exact sequence during the live defense without additional setup:

1. **Pre-Flight Check (30s)**:
   Verify clean baseline and tool availability:

   ```bash
   git rev-parse --short HEAD   # Expect b29fec117 or current dev merge commit
   ```

2. **Live Execution of All 4 Security Scenarios + Branch Coverage (90s)**:
   Run the single deterministic command:

   ```bash
   HYPOTHESIS_PROFILE=recorded uv run --python 3.12 --dev --frozen pytest \
     src/documents/tests/test_path_security.py -k TestPathSecurity -n 0 -v \
     --hypothesis-seed=20261008 --hypothesis-show-statistics \
     --cov=documents.file_handling --cov-branch --cov-report=term-missing
   ```

   Walk the committee through the 5 printed test cases:
   - **Descendant**: `test_descendants_inside_root_accepted` (valid files accepted)
   - **Traversal**: `test_traversal_outside_root_rejected` (`..` escaping root blocked)
   - **Prefix Confusion**: `test_prefix_confusion_outside_root_rejected` (sibling prefix `storage_root_<suffix>` blocked)
   - **Symlinks**: `test_symlink_escaping_root_rejected` (external escape blocked) & `test_symlink_internal_to_root_accepted` (internal link allowed)
   - **Coverage table**: Point out `BrPart = 0` and Missing starting at line `33` (`100%` statement/branch coverage on lines `21–29`).

3. **Optional Live Shrinking Demonstration (60s)**:
   Run an ephemeral in-memory Hypothesis check (without editing tracked repository files) demonstrating why `.resolve()` is mandatory:

   ```python
   # Inline proof (flawed lexical check fails symlink test and shrinks to 'p_0'):
   # Expected stdout: Failed: DID NOT RAISE UnsafeFilePathError
   # Falsifying example: filename='p_0'
   ```

   Reference [`pbt-03-cycle-3-evidence.md#63`](pbt-03-cycle-3-evidence.md) and explain the defense-in-depth finding in `delete_empty_directories` (`line 40`).

4. **Fallback Plan (If Network / OS Privileges Interrupt Live Demo)**:
   - **Fallback A (Windows symlink permission denied)**: `PBT03-INV3A/B` cleanly report `SKIPPED` via `pytest.skip` rather than false pass; switch to WSL2/Ubuntu or show the GitHub Actions run.
   - **Fallback B (Offline / CI outage)**: Present the pre-downloaded CI artifact `.artifacts/pbt-03-run.log` from [Run #45791632304](https://github.com/0Yaam/paperless-ngx/actions/runs/45791632304) alongside the local clean-worktree log captured above.

---

## 4. Peer-Review Evidence Trail

| Cycle      | Pull Request                                                  | Reviewer       | Review Findings & Revisions                                                                                                                                                                                                                   | Outcome & Commit                                          |
| :--------- | :------------------------------------------------------------ | :------------- | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :-------------------------------------------------------- |
| Cycle 1–2  | [PR #21](https://github.com/0Yaam/paperless-ngx/pull/21)      | `@HubertPhung` | Requested splitting traversal vs prefix confusion, converting symlink checks into Hypothesis properties, dropping unused `django_db`, adding design doc. Resolved across follow-up pushes.                                                    | Merged `20edae480`                                        |
| Cycle 3    | [PR #40](https://github.com/0Yaam/paperless-ngx/pull/40)      | `@HubertPhung` | Verified 5/5 pass, 100% coverage & RCA on `delete_empty_directories`; requested fixes for `codespell` words, `prettier`, broken anchor, deduplicated `settings.register_profile`, and shrinking example. Resolved in `7905814bc`–`10747a862`. | **APPROVED** (`2026-11-24T07:55:23Z`), Merged `cb087a99b` |
| Cycle 3 CI | [PR #42](https://github.com/0Yaam/paperless-ngx/pull/42)      | `@0Yaam`       | Added `-n 0` and `--cov-branch` to `coursework-ci.yml` (`pbt-03`) for deterministic ordering and branch metrics.                                                                                                                              | Merged `2fa7b2dec`                                        |
| Cycle 4    | Issue [#50](https://github.com/0Yaam/paperless-ngx/issues/50) | `@HubertPhung` | Final A–H report alignment, clean checkout verification on `b29fec117`, defense demo runbook, and traceability review.                                                                                                                        | Pending final PR approval                                 |
