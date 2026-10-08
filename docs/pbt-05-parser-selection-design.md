# PBT-05: Parser Selection Design (Cycle 1)

## Overview

- **Target under test**: `paperless.parsers.registry.ParserRegistry.get_parser_for_file`
- **Source file**: [`src/paperless/parsers/registry.py`](../src/paperless/parsers/registry.py)
- **Suite**: PBT-05 Parser selection
- **Owner**: Phùng Nguyễn Hoài Bo ([@HubertPhung](https://github.com/HubertPhung))
- **Reviewer**: Vũ Thế Huỳnh ([@1convitt](https://github.com/1convitt))
- **Issue**: [#14 — [Cycle 1][Bo] Design PBT-05 Parser selection](https://github.com/0Yaam/paperless-ngx/issues/14)

---

## 1. Context and Architecture

In Paperless-ngx, `ParserRegistry` tracks document parsers partitioned into:

- `_builtins`: Standard parsers bundled with the application (Text, Mail, Rasterised, Remote, Tika).
- `_external`: Third-party parsers discovered from Python entry points (`paperless_ngx.parsers`).

When resolving a parser for a document, `get_parser_for_file` arbitrates candidate parsers using:

1. Supported MIME type matches (`supported_mime_types()`).
2. Remote service filtering (`allow_remote` vs `uses_remote_service`).
3. Priority scoring (`score(mime_type, filename, path)`).
4. Tie-breaking precedence (external parsers win ties over built-ins).

---

## 2. Independent Invariants

### Invariant 1: Highest-Scoring Eligible Parser Wins (`PBT05-INV1-MAX-SCORE`)

- **Objective**: Ensure arbitration selects an eligible parser that achieves the strictly maximum integer score.
- **Preconditions**:
  - The eligible candidate set $\mathcal{E}$ is non-empty.
  - A candidate parser $P$ is eligible iff:
    - $\text{mime\_type} \in P.\text{supported\_mime\_types}()$
    - If $\text{allow\_remote} = \text{False}$, then $\operatorname{getattr}(P, \text{"uses\_remote\_service"}, \text{False}) == \text{False}$.
    - $P.\text{score}(\text{mime\_type}, \text{filename}, \text{path}) \ne \text{None}$.
- **Hypothesis Strategy**:
  - Generate 1 to 8 mock parser specifications with:
    - Random scores in $[-100, 100]$ or `None`.
    - Supported MIME subsets from `{"text/plain", "application/pdf", "image/png", "application/xml"}`.
    - Boolean flag `uses_remote_service`.
    - Partitioned into `_external` or `_builtins`.
  - Bounded input arguments: `mime_type`, `filename`, and `allow_remote: st.booleans()`.
- **Reference Oracle**:
  $$\text{best\_score} = \max_{P \in \mathcal{E}} P.\text{score}(\text{mime\_type}, \text{filename}, \text{path})$$
  $$\text{Oracle: } P^* \in \mathcal{E} \quad \land \quad P^*.\text{score}(\dots) == \text{best\_score}$$
- **Sample Counterexample**:
  - _Hypothetical bug_: Comparing with `score < best_score` (selecting lowest score) or failing to update `best_score`.
  - _Minimal input_: `BuiltinA(score=10)` and `BuiltinB(score=20)` for a text file.
  - _Erroneous outcome_: Returns `BuiltinA` (score 10) instead of `BuiltinB` (score 20).

---

### Invariant 2: External Parser Wins Score Tie with Built-in (`PBT05-INV2-TIE-BREAK-EXTERNAL`)

- **Objective**: Validate third-party extension precedence: when an external parser and a built-in parser tie for top score, the external parser wins.
- **Preconditions**:
  - Both an external parser and a built-in parser are eligible with score $S_{tie} = \max_{P \in \mathcal{E}} P.\text{score}$.
- **Hypothesis Strategy**:
  - Generate a tie score $S_{tie} \in [-50, 50]$.
  - Generate 1..4 external parsers and 1..4 built-in parsers supporting `mime_type`.
  - Force at least one external parser and one built-in parser to have score $S_{tie}$.
  - Ensure all other candidates have score $\le S_{tie}$.
- **Reference Oracle**:
  $$\text{Oracle: } P^* \in \text{registry.\_external} \quad \land \quad P^* \notin \text{registry.\_builtins} \quad \land \quad P^*.\text{score}(\dots) == S_{tie}$$
- **Sample Counterexample**:
  - _Hypothetical bug_: Iteration order reversed `(*self._builtins, *self._external)` or using `>=` (last-seen wins).
  - _Minimal input_: `ExternalParser(score=10)` and `BuiltinParser(score=10)`.
  - _Erroneous outcome_: Returns `BuiltinParser` instead of `ExternalParser`.

---

### Invariant 3: Remote Parsers Excluded When `allow_remote=False` (`PBT05-INV3-REMOTE-EXCLUSION`)

- **Objective**: Guarantee privacy and offline execution: when `allow_remote=False`, parsers with `uses_remote_service = True` are strictly excluded regardless of score superiority.
- **Preconditions**:
  - `allow_remote = False`.
  - The registry contains candidate parsers declaring `uses_remote_service = True`.
- **Hypothesis Strategy**:
  - Generate 1..3 remote parsers with `uses_remote_service = True` and high scores (e.g. 100).
  - Generate 0..3 local parsers with lower scores ($1..40$) or declining (`None`).
  - Invoke `get_parser_for_file(..., allow_remote=False)`.
- **Reference Oracle**:
  $$\text{Oracle: } P^* \text{ is None} \quad \lor \quad \operatorname{getattr}(P^*, \text{"uses\_remote\_service"}, \text{False}) == \text{False}$$
- **Sample Counterexample**:
  - _Hypothetical bug_: Omitting the `not allow_remote and getattr(parser_class, "uses_remote_service", False)` guard.
  - _Minimal input_: `RemoteParser(score=100, remote=True)` and `LocalParser(score=10)` with `allow_remote=False`.
  - _Erroneous outcome_: Returns `RemoteParser` due to higher score.

---

### Invariant 4: Robust Fallback to None (`PBT05-INV4-NONE-FALLBACK`)

- **Objective**: Ensure graceful handling when no registered parser matches or all eligible parsers decline.
- **Preconditions**:
  - Eligible set $\mathcal{E} = \emptyset$ (no MIME match, all `score() == None`, or all remote when `allow_remote=False`).
- **Hypothesis Strategy**:
  - Generate parsers with disjoint MIME types or `score() == None`.
- **Reference Oracle**:
  $$\text{Oracle: } P^* \text{ is None}$$
- **Sample Counterexample**:
  - _Hypothetical bug_: Returning `self._builtins[0]` as default fallback or raising unhandled exception.
  - _Minimal input_: Single registered parser `DecliningParser(score=None)`.
  - _Erroneous outcome_: Returns `DecliningParser` instead of `None`.

---

## 3. Bounded Parameters and CI Stability

- **Execution bounds**: `@settings(max_examples=200, deadline=None)` per property.
- **Collection bounds**: 0 to 8 mock parser classes per execution.
- **Score domain**: Bounded integers $[-100, 100]$.
- **Isolation**: Fresh `ParserRegistry` instance per test iteration, leaving global singleton untouched.

---

## 4. Scope Demarcation (K01)

- **PBT-04 (@1convitt - Reviewer)**: Tests static MIME-to-extension dictionary mappings in `documents.parsers`.
- **PBT-05 (@HubertPhung - Owner)**: Tests dynamic parser arbitration, scoring, and remote exclusion in `ParserRegistry.get_parser_for_file`.
- **Conclusion**: Target functions and domains are disjoint with zero overlap.
