# PBT-05: Parser Selection Design (Cycle 1)

## Overview

- **Target under test**: `paperless.parsers.registry.ParserRegistry.get_parser_for_file`
- **Source file**: [`src/paperless/parsers/registry.py`](../src/paperless/parsers/registry.py)
- **Suite**: PBT-05 Parser selection
- **Owner**: Phùng Nguyễn Hoài Bo ([@HubertPhung](https://github.com/HubertPhung))
- **Reviewer**: Vũ Thế Huỳnh ([@1convitt](https://github.com/1convitt))
- **Issue**: [#14 — [Cycle 1][Bo] Design PBT-05 Parser selection](https://github.com/0Yaam/paperless-ngx/issues/14)
- **Implementation PR**: [#18 — test: design and cover parser selection properties](https://github.com/0Yaam/paperless-ngx/pull/18)

---

## 1. Context and Problem Statement

### 1.1 The Document Parsing Challenge in Paperless-ngx

Paperless-ngx ingests heterogeneous file formats (PDFs, TIFF images, plain text, emails, Office documents). To parse these documents into searchable text and metadata, the system maintains a collection of specialized document parser classes.

In `src/paperless/parsers/registry.py`, the `ParserRegistry` class manages two separate lists of parsers:

1. `_builtins`: Standard parsers bundled with the core system (e.g., Tesseract OCR parser, text parser, email parser, remote Gotenberg/Tika parsers).
2. `_external`: Third-party parser extensions discovered dynamically via Python entry points (`paperless_ngx.parsers`).

### 1.2 Arbitration Algorithm (`get_parser_for_file`)

When a document is consumed, the engine invokes `get_parser_for_file(mime_type, filename, path=None, allow_remote=True)`. The method must resolve exactly one optimal parser class from all registered candidates by executing the following arbitration rules in strict order:

1. **MIME Compatibility**: Only parsers declaring `mime_type in parser_class.supported_mime_types()` are eligible.
2. **Air-gap / Remote Protection**: If `allow_remote=False`, any parser declaring `getattr(parser_class, "uses_remote_service", False) == True` must be strictly excluded.
3. **Dynamic Suitability Scoring**: Each surviving parser computes `score = parser_instance.score(mime_type, filename, path)`. If `score is None`, the parser declines the file and is discarded.
4. **Maximum Score Principle**: Parsers with strictly higher numerical scores win over parsers with lower scores.
5. **Tie-breaking Precedence**: If an external plugin parser and a built-in parser achieve the identical highest score, the **external parser takes precedence**, allowing users to override default behavior with custom plugins.
6. **Graceful Degradation**: If no eligible parser exists or all parsers decline, the method safely returns `None` without raising uncaught exceptions.

---

## 2. Mathematical Formalization of Parser Arbitration

Let $\mathcal{P} = \mathcal{P}_{\text{ext}} \cup \mathcal{P}_{\text{built}}$ be the finite set of registered parser classes, where:

- $\mathcal{P}_{\text{ext}} = \text{registry.\_external}$
- $\mathcal{P}_{\text{built}} = \text{registry.\_builtins}$
- $\mathcal{P}_{\text{ext}} \cap \mathcal{P}_{\text{built}} = \emptyset$

Given input query parameters $q = (\mu, f, p, \alpha)$ where:

- $\mu \in \text{MIME}$: MIME type string
- $f \in \text{String}$: filename
- $p \in \text{Path} \cup \{\text{None}\}$: file path
- $\alpha \in \{\text{True}, \text{False}\}$: `allow_remote` flag

### Step 1: Filtering Eligible Candidates

A parser $P \in \mathcal{P}$ is eligible under query $q$, denoted $P \in \mathcal{E}(q)$, if and only if:
$$\mu \in P.\text{supported\_mime\_types}()$$
$$\neg \alpha \implies \neg \operatorname{getattr}(P, \text{"uses\_remote\_service"}, \text{False})$$
$$P.\text{score}(\mu, f, p) \ne \text{None}$$

### Step 2: Scoring Function

Define score function $S(P) = P.\text{score}(\mu, f, p) \in \mathbb{Z}$ for all $P \in \mathcal{E}(q)$.

### Step 3: Candidate Ranking and Tie-breaking

Let $S^* = \max_{P \in \mathcal{E}(q)} S(P)$ be the optimal score.
The set of winning candidates is:
$$\mathcal{W}(q) = \{ P \in \mathcal{E}(q) \mid S(P) = S^* \}$$

The arbitration decision $P^*(q)$ satisfies:

$$ P^*(q) = \begin{cases}
\text{None} & \text{if } \mathcal{E}(q) = \emptyset \\
\text{first } P \in \mathcal{W}(q) \cap \mathcal{P}_{\text{ext}} & \text{if } \mathcal{W}(q) \cap \mathcal{P}_{\text{ext}} \ne \emptyset \\
\text{first } P \in \mathcal{W}(q) \cap \mathcal{P}_{\text{built}} & \text{otherwise}
\end{cases}$$

---

## 3. Independent Invariants Specification

### Invariant 1: Highest-Scoring Eligible Parser Wins (`PBT05-INV1-MAX-SCORE`)

- **Objective**: Verify that the selected parser always achieves the maximum integer score among all eligible candidates.
- **Preconditions**:
  - $\mathcal{E}(q) \ne \emptyset$.
- **Hypothesis Strategy**:
  - Generate 1 to 8 mock parser specifications:
    - Scores in $[-100, 100]$ or `None`.
    - MIME subsets sampled from `{"text/plain", "application/pdf", "image/png", "application/xml"}`.
    - Boolean `uses_remote_service`.
    - Flag `is_external` placing the parser into either `_external` or `_builtins`.
  - Input parameters: `target_mime`, `filename: st.text(min_size=1, max_size=20)`, `allow_remote: st.booleans()`.
- **Reference Oracle**:
  $$P^* \in \mathcal{E}(q) \quad \land \quad S(P^*) = \max_{P \in \mathcal{E}(q)} S(P)$$
- **Sample Counterexample**:
  - *Hypothetical bug*: Comparing scores using `<` instead of `>` (returning the minimum score parser), or failing to update the current maximum.
  - *Input*: `ParserA(score=10)` and `ParserB(score=95)` for `application/pdf`.
  - *Erroneous outcome*: Returns `ParserA` (score 10).

---

### Invariant 2: External Parser Wins Score Tie with Built-in (`PBT05-INV2-TIE-BREAK-EXTERNAL`)

- **Objective**: Validate extension precedence: when an external parser and a built-in parser tie for the top score, the external parser wins.
- **Preconditions**:
  - Both an external parser and a built-in parser belong to $\mathcal{W}(q)$.
- **Hypothesis Strategy**:
  - Generate a shared tie score $S_{tie} \in [-50, 50]$.
  - Generate 1..4 external parsers and 1..4 built-in parsers supporting `target_mime`.
  - Ensure at least one external and one built-in parser have score $S_{tie}$, with all others scoring $\le S_{tie}$.
- **Reference Oracle**:
  $$P^* \in \mathcal{P}_{\text{ext}} \quad \land \quad P^* \notin \mathcal{P}_{\text{built}} \quad \land \quad S(P^*) = S_{tie}$$
- **Sample Counterexample**:
  - *Hypothetical bug*: Order of iteration inverted `(*self._builtins, *self._external)` or using `>=` (last-seen wins).
  - *Input*: `ExternalPDFParser(score=50)` and `BuiltinPDFParser(score=50)`.
  - *Erroneous outcome*: Returns `BuiltinPDFParser`.

---

### Invariant 3: Remote Parsers Excluded When `allow_remote=False` (`PBT05-INV3-REMOTE-EXCLUSION`)

- **Objective**: Guarantee air-gapped data security: when `allow_remote=False`, parsers with `uses_remote_service = True` are strictly discarded regardless of score superiority.
- **Preconditions**:
  - `allow_remote = False`.
  - The registry contains candidate parsers declaring `uses_remote_service = True`.
- **Hypothesis Strategy**:
  - Generate 1..3 remote parsers declaring `uses_remote_service = True` with high scores (e.g. 100).
  - Generate 0..3 local parsers with lower scores ($1..40$) or declining (`None`).
  - Query with `allow_remote=False`.
- **Reference Oracle**:
  $$P^* \text{ is None} \quad \lor \quad \operatorname{getattr}(P^*, \text{"uses\_remote\_service"}, \text{False}) == \text{False}$$
- **Sample Counterexample**:
  - *Hypothetical bug*: Omitting the `allow_remote` check in the parser loop.
  - *Input*: `CloudOCRParser(score=100, remote=True)` and `LocalTesseract(score=40, remote=False)` with `allow_remote=False`.
  - *Erroneous outcome*: Returns `CloudOCRParser`.

---

### Invariant 4: Robust None Fallbacks (`PBT05-INV4-NONE-FALLBACK`)

To cleanly verify that `None` is returned for specific, distinct failure causes without confounding multiple conditions, Invariant 4 is partitioned into four independent sub-properties:

#### Invariant 4A: Empty Registry (`PBT05-INV4A-EMPTY-REGISTRY`)
- **Preconditions**: `len(registry.all_parsers()) == 0`.
- **Strategy**: Invoke `get_parser_for_file` on arbitrary MIME types and filenames with an empty registry.
- **Reference Oracle**: `get_parser_for_file(...) is None`.
- **Counterexample**: Implementation raising `IndexError` or `StopIteration` when searching an empty candidate list.

#### Invariant 4B: Unsupported MIME with Valid Scores (`PBT05-INV4B-UNSUPPORTED-MIME`)
- **Preconditions**: Parsers exist with positive scores ($[1, 100]$) and local execution, but none advertise `target_mime`.
- **Strategy**: Generate 1..5 local parsers supporting only unrelated MIME types (e.g., `other/unsupported-mime`).
- **Reference Oracle**: `get_parser_for_file(target_mime, ...) is None`.
- **Counterexample**: Implementation falling back to filename extension matching or defaulting to the first parser.

#### Invariant 4C: Matching MIME with Declining Scores (`PBT05-INV4C-DECLINING-SCORE`)
- **Preconditions**: Candidate parsers match `target_mime`, but all return `score() == None`.
- **Strategy**: Generate 1..5 parsers matching `target_mime` whose `score()` returns `None`.
- **Reference Oracle**: `get_parser_for_file(...) is None`.
- **Counterexample**: Implementation coercing `score is None` to score `0` and returning the candidate.

#### Invariant 4D: Blocked Remote Parsers Only (`PBT05-INV4D-BLOCKED-REMOTE-ONLY`)
- **Preconditions**: Candidate parsers match `target_mime` and return positive scores, but all declare `uses_remote_service = True` while `allow_remote = False`.
- **Strategy**: Generate 1..5 remote parsers with scores in $[10, 100]$ while invoking `allow_remote=False`.
- **Reference Oracle**: `get_parser_for_file(..., allow_remote=False) is None`.
- **Counterexample**: Implementation falling back to a remote parser when no local parser is available.

---

## 4. Bounded Parameters and CI Reproducibility

- **Execution bounds**: `@settings(max_examples=200, deadline=None)` per property.
- **Reproducibility seed**: Fixed CLI seed `--hypothesis-seed=20261008`.
- **Statistics reporting**: `--hypothesis-show-statistics` enabled to capture coverage and shrinking metrics.
- **Artifact retention**: Execution logs captured to `.artifacts/pbt-05-run.log` and uploaded as workflow artifact `pbt-05-evidence`.
- **Collection bounds**: 0 to 8 mock parser classes per execution iteration.
- **Score domain**: Bounded integers $[-100, 100]$.
- **Isolation**: Fresh `ParserRegistry` instance per test iteration, leaving global singleton untouched.

---

## 5. Traceability Matrix and Test Results

| Invariant ID | Test Method | Covered Rule | CI Seed | Status |
| :--- | :--- | :--- | :---: | :---: |
| `PBT05-INV1` | `test_highest_scoring_eligible_parser_wins` | Maximum integer score wins | `20261008` | :white_check_mark: Pass (200 ex) |
| `PBT05-INV2` | `test_external_parser_wins_tie_with_builtin` | External beats built-in on tie | `20261008` | :white_check_mark: Pass (200 ex) |
| `PBT05-INV3` | `test_remote_parsers_excluded_when_not_allowed` | Air-gap remote parser exclusion | `20261008` | :white_check_mark: Pass (200 ex) |
| `PBT05-INV4A` | `test_none_returned_when_registry_is_empty` | Empty registry fallback | `20261008` | :white_check_mark: Pass (200 ex) |
| `PBT05-INV4B` | `test_none_returned_when_mime_unsupported_despite_valid_scores` | Unsupported MIME fallback | `20261008` | :white_check_mark: Pass (200 ex) |
| `PBT05-INV4C` | `test_none_returned_when_matching_parsers_decline_with_none_score` | Declining parser fallback | `20261008` | :white_check_mark: Pass (200 ex) |
| `PBT05-INV4D` | `test_none_returned_when_only_remote_parsers_available_and_not_allowed` | Remote-only blocked fallback | `20261008` | :white_check_mark: Pass (200 ex) |

---

## 6. Scope Demarcation (K01)

- **PBT-04 (@1convitt - Reviewer)**: Tests static MIME-to-extension dictionary mappings in `documents.parsers`.
- **PBT-05 (@HubertPhung - Owner)**: Tests dynamic parser arbitration, scoring, and remote exclusion in `ParserRegistry.get_parser_for_file`.
- **Boundary**: Disjoint domain models with zero overlapping code paths or assertions.
$$
