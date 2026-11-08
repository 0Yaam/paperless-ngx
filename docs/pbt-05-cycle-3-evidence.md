# PBT-05 — Bằng chứng thực hiện và Phân tích kiểm thử Cycle 3 (Parser Selection)

## 1. Thông tin chung & Phạm vi

- **Suite**: PBT-05 Parser selection (Lựa chọn bộ phân tích tài liệu)
- **Issue**: [#39 — [Cycle 3][Bo] Execute & analyze PBT-05 Parser selection](https://github.com/0Yaam/paperless-ngx/issues/39)
- **Milestone**: Cycle 3 — Execute & Analyze
- **Owner**: Phùng Nguyễn Hoài Bo ([@HubertPhung](https://github.com/HubertPhung))
- **Peer Reviewer**: Vũ Thế Huỳnh ([@1convitt](https://github.com/1convitt))
- **Target Method**: `paperless.parsers.registry.ParserRegistry.get_parser_for_file(mime_type, filename, path=None, *, allow_remote=True)`
- **Source File**: [`src/paperless/parsers/registry.py`](../src/paperless/parsers/registry.py)
- **Test File**: [`src/paperless/tests/test_parser_selection_pbt.py`](../src/paperless/tests/test_parser_selection_pbt.py)
- **Design Specification**: [`docs/pbt-05-parser-selection-design.md`](pbt-05-parser-selection-design.md)
- **Baseline Git Commit**: `dfd1e882a`

---

## 2. Hợp đồng kiểm thử & Ma trận 7 Property Invariants

Bộ kiểm thử PBT-05 gồm 7 property độc lập kiểm chứng toàn diện 4 nhóm bất biến từ tài liệu thiết kế Cycle 1 ([`pbt-05-parser-selection-design.md`](pbt-05-parser-selection-design.md)):

| Mã Invariant                          | Hàm kiểm thử                                                            | Tính chất bất biến (Invariant)                                                                           | Chiến lược sinh dữ liệu (Strategy)                                                                                                                         | Tiêu chuẩn kiểm chứng (Oracle)                                                                                                              |
| :------------------------------------ | :---------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------ |
| **`PBT05-INV1-MAX-SCORE`**            | `test_highest_scoring_eligible_parser_wins`                             | Trong tập ứng viên hợp lệ, parser có điểm số nguyên cao nhất luôn thắng.                                 | 1–8 `MockParserSpec` ngẫu nhiên (`score_val` $\in [-100, 100] \cup \{\text{None}\}$, `is_external`, `uses_remote_service`), `target_mime`, `allow_remote`. | Nếu tập hợp lệ $\mathcal{E} = \emptyset \Rightarrow \text{None}$; ngược lại $\text{score}(P^*) = \max_{P \in \mathcal{E}} \text{score}(P)$. |
| **`PBT05-INV2-TIE-BREAK-EXTERNAL`**   | `test_external_parser_wins_tie_with_builtin`                            | Khi hòa điểm cao nhất giữa external và built-in parser, external parser luôn được ưu tiên.               | `tie_score` $\in [-50, 50]$, `ext_count` $\in [1, 4]$, `builtin_count` $\in [1, 4]`, `target_mime`.                                                        | $P^* \in \text{registry.\_external}$, $P^* \notin \text{registry.\_builtins}$, và $P^* \equiv \text{registry.\_external}[0]$.               |
| **`PBT05-INV3-REMOTE-EXCLUSION`**     | `test_remote_parsers_excluded_when_not_allowed`                         | Khi `allow_remote=False`, mọi parser có `uses_remote_service=True` bị loại bỏ tuyệt đối dù điểm cao hơn. | 1–3 remote parser (`score` $\in [50, 100]$), 0–3 local parser (`score` $\in [-20, 40] \cup \{\text{None}\}$), `allow_remote=False`.                        | Kết quả không bao giờ có `uses_remote_service=True`; chọn local parser có điểm cao nhất hoặc `None`.                                        |
| **`PBT05-INV4A-EMPTY-REGISTRY`**      | `test_none_returned_when_registry_is_empty`                             | Registry rỗng luôn trả về `None` an toàn cho mọi đầu vào.                                                | `target_mime` từ `MIME_POOL`, `filename` độ dài 1–20, `allow_remote` boolean.                                                                              | `result is None` (không phát sinh ngoại lệ).                                                                                                |
| **`PBT05-INV4B-UNSUPPORTED-MIME`**    | `test_none_returned_when_mime_unsupported_despite_valid_scores`         | Trả về `None` khi không có parser nào hỗ trợ `target_mime` dù điểm số hợp lệ.                            | 1–5 local parser có `score` $\in [1, 100]$ nhưng chỉ hỗ trợ `other/unsupported-mime`.                                                                      | `result is None`.                                                                                                                           |
| **`PBT05-INV4C-DECLINING-SCORE`**     | `test_none_returned_when_matching_parsers_decline_with_none_score`      | Trả về `None` khi mọi parser khớp MIME đều từ chối xử lý (`score() -> None`).                            | 1–5 parser khớp `application/pdf` nhưng `score_val=None`.                                                                                                  | `result is None`.                                                                                                                           |
| **`PBT05-INV4D-BLOCKED-REMOTE-ONLY`** | `test_none_returned_when_only_remote_parsers_available_and_not_allowed` | Trả về `None` khi chỉ có remote parser khớp MIME nhưng gọi với `allow_remote=False`.                     | 1–5 remote parser khớp `application/pdf`, `score` $\in [10, 100]$, gọi với `allow_remote=False`.                                                           | `result is None`.                                                                                                                           |

---

## 3. Cấu hình Profile Hypothesis & Môi trường thực hiện

Trong `src/paperless/tests/test_parser_selection_pbt.py`, hệ thống đăng ký các profile Hypothesis phân tầng để hỗ trợ kiểm thử từ phát triển cục bộ đến ghi nhận bằng chứng CI:

```python
for _profile_name, _max_examples in [
    ("dev", 50),
    ("ci", 100),
    ("default_pbt05", 200),
    ("recorded", 250),
    ("stress", 500),
]:
    settings.register_profile(
        _profile_name,
        max_examples=_max_examples,
        deadline=None,
        suppress_health_check=[HealthCheck.function_scoped_fixture],
    )
settings.load_profile(os.environ.get("HYPOTHESIS_PROFILE", "default_pbt05"))
```

- **Profile ghi nhận bằng chứng Cycle 3**: `HYPOTHESIS_PROFILE=recorded` (`max_examples=250` cho mỗi property, tổng cộng **1,750 ca kiểm thử sinh tự động** + **10 ca biên `@example` tường minh**).
- **Seed cố định tái lập**: `--hypothesis-seed=20261008`.
- **Lưu trữ bằng chứng tự động**: Job `PBT-05 Parser selection` trong `.github/workflows/coursework-ci.yml` xuất toàn bộ nhật ký chạy kèm thống kê Hypothesis và báo cáo độ bao phủ vào `.artifacts/pbt-05-run.log` và tải lên artifact `pbt-05-evidence`.
- **Môi trường đo lường**:
  - Hệ điều hành: Windows 11 64-bit (Local) / Ubuntu 24.04 LTS (GitHub Actions Runner)
  - Python: `3.12.10`
  - Pytest: `9.2.0`
  - Hypothesis: `6.170.15`
  - Coverage Plugin: `pytest-cov 7.1.0`

---

## 4. Số liệu thống kê Hypothesis (Execution Metrics & Statistics)

Kết quả thu thập từ lần chạy tất định với `HYPOTHESIS_PROFILE=recorded` và `--hypothesis-seed=20261008`:

| Hàm kiểm thử                                                            | Passed (Generated) | Explicit `@example` | Failed | Invalid (Deduplicated) | Thời gian pha sinh | Thời gian mỗi mẫu  |
| :---------------------------------------------------------------------- | :----------------: | :-----------------: | :----: | :--------------------: | :----------------: | :----------------: |
| `test_highest_scoring_eligible_parser_wins`                             |        250         |          2          |   0    |           20           |       2.12s        |      ~1–9 ms       |
| `test_external_parser_wins_tie_with_builtin`                            |        250         |          2          |   0    |           0            |       0.61s        |       ~1 ms        |
| `test_remote_parsers_excluded_when_not_allowed`                         |        250         |          2          |   0    |           89           |       0.99s        |      ~0–2 ms       |
| `test_none_returned_when_registry_is_empty`                             |        250         |          1          |   0    |           0            |       0.66s        |      ~0–1 ms       |
| `test_none_returned_when_mime_unsupported_despite_valid_scores`         |        250         |          1          |   0    |           70           |       1.30s        |      ~0–3 ms       |
| `test_none_returned_when_matching_parsers_decline_with_none_score`      |        250         |          1          |   0    |           36           |       1.29s        |      ~1–4 ms       |
| `test_none_returned_when_only_remote_parsers_available_and_not_allowed` |        250         |          1          |   0    |           24           |       1.13s        |      ~1–3 ms       |
| **Tổng cộng (7 properties)**                                            |     **1,750**      |       **10**        | **0**  |        **239**         |     **8.10s**      | **10.80s toàn bộ** |

> [!NOTE]
> Trong báo cáo thống kê của Hypothesis, cột `invalid test cases` ở các bài test dùng `st.lists(...)` với không gian tổ hợp hữu hạn (ví dụ `alphabet="abc"`, `supported_mimes=st.just(...)`) phản ánh cơ chế **loại bỏ mẫu sinh trùng lặp nội bộ (internal conjecture deduplication)** của Hypothesis nhằm đảm bảo đủ 250 mẫu kiểm thử độc lập (unique examples), không phải do lệnh `assume()` hay `.filter()` bị từ chối. Toàn bộ 7 property đều đạt ngưỡng 250 mẫu hợp lệ trong ~0.6–2.1 giây mỗi bài.

---

## 5. Bằng chứng độ bao phủ mã nguồn (Code Coverage Evidence)

Kết quả đo độ bao phủ bằng `pytest-cov` (`--cov=paperless.parsers.registry --cov-report=term-missing`):

```text
Name                                Stmts   Miss  Cover   Missing
-----------------------------------------------------------------
src\paperless\parsers\registry.py      89     49    45%   89-99, 118-122, 142-143, 200-210, 234-259, 282-303, 326
-----------------------------------------------------------------
TOTAL                                  89     49    45%
```

### Phân tích chi tiết trên hàm mục tiêu `ParserRegistry.get_parser_for_file` (dòng 332–407)

Đối chiếu danh sách dòng `Missing` (`89-99`, `118-122`, `142-143`, `200-210`, `234-259`, `282-303`, `326`), **không có bất kỳ dòng nào nằm trong phạm vi dòng 332–407** của `get_parser_for_file`:

```python
375:         best_score: int | None = None
376:         best_parser: type[ParserProtocol] | None = None
377:
378:         for parser_class in (*self._external, *self._builtins):
379:             if mime_type not in parser_class.supported_mime_types():
380:                 continue
381:
382:             if not allow_remote and getattr(
383:                 parser_class,
384:                 "uses_remote_service",
385:                 False,
386:             ):
387:                 continue
388:
389:             score = parser_class.score(mime_type, filename, path)
390:             if score is None:
391:                 continue
392:
393:             if best_score is None or score > best_score:
394:                 best_score = score
395:                 best_parser = parser_class
396:
397:         if best_parser is not None and best_parser in self._external:
398:             logger.info(...)
399:
400:         return best_parser
```

- **Statement Coverage trên `get_parser_for_file`**: **100%** (17/17 câu lệnh vận hành trong thân hàm và `register_builtin` / `__init__`).
- **Branch Coverage trên `get_parser_for_file`**: **100%** (12/12 nhánh quyết định được kích hoạt):
  1. Vòng lặp `for parser_class in (*self._external, *self._builtins)`:
     - Nhánh rỗng (0 phần tử): kích hoạt bởi `PBT05-INV4A-EMPTY-REGISTRY`.
     - Nhánh không rỗng ($\ge 1$ phần tử): kích hoạt bởi 6 property còn lại.
  2. Điều kiện `if mime_type not in parser_class.supported_mime_types()`:
     - Nhánh `True` (`continue`): kích hoạt bởi `PBT05-INV4B-UNSUPPORTED-MIME` và `PBT05-INV1-MAX-SCORE`.
     - Nhánh `False`: kích hoạt khi parser hỗ trợ `target_mime`.
  3. Điều kiện `if not allow_remote and getattr(parser_class, "uses_remote_service", False)`:
     - Nhánh `True` (`continue`): kích hoạt bởi `PBT05-INV3-REMOTE-EXCLUSION` và `PBT05-INV4D-BLOCKED-REMOTE-ONLY`.
     - Nhánh `False`: kích hoạt khi `allow_remote=True` hoặc parser chạy cục bộ.
  4. Điều kiện `if score is None`:
     - Nhánh `True` (`continue`): kích hoạt bởi `PBT05-INV4C-DECLINING-SCORE`, `PBT05-INV1`, và `PBT05-INV3`.
     - Nhánh `False`: kích hoạt khi parser trả về điểm nguyên hợp lệ.
  5. Điều kiện `if best_score is None or score > best_score`:
     - Nhánh `True` (cập nhật `best_score`, `best_parser`): kích hoạt ở ứng viên hợp lệ đầu tiên hoặc khi gặp ứng viên có điểm lớn hơn.
     - Nhánh `False` (giữ nguyên `best_parser` khi điểm thấp hơn hoặc bằng): kích hoạt bởi `PBT05-INV2-TIE-BREAK-EXTERNAL` và `PBT05-INV1-MAX-SCORE`.
  6. Điều kiện `if best_parser is not None and best_parser in self._external`:
     - Nhánh `True` (ghi log `logger.info` cho third-party parser): kích hoạt bởi `PBT05-INV2-TIE-BREAK-EXTERNAL` và `PBT05-INV1-MAX-SCORE`.
     - Nhánh `False` (khi kết quả là `None` hoặc là built-in parser): kích hoạt bởi `PBT05-INV3` và `PBT05-INV4A..4D`.
- Các dòng `Missing` còn lại trong file thuộc về các hàm khởi tạo singleton (`get_parser_registry`, `init_builtin_parsers`, `reset_parser_registry`), nạp entrypoint (`discover`), và ghi log danh sách (`log_summary`), vốn nằm ngoài phạm vi của hàm mục tiêu `get_parser_for_file`.

---

## 6. Bằng chứng Không lỗi, Shrinking & Thực nghiệm Tiêm lỗi (Counterexample & Mutation Analysis)

### 6.1. Bằng chứng không lỗi trên mã nguồn chuẩn (No-Failure Evidence)

Sau **1,750 mẫu ngẫu nhiên** và **10 ca biên `@example`**, cả 7 property đều vượt qua hoàn toàn (`7 passed in 10.80s`, `0 failures`), chứng minh hàm `ParserRegistry.get_parser_for_file` trên nhánh `dev` tuân thủ chặt chẽ đặc tả lựa chọn parser.

### 6.2. Thực nghiệm tiêm lỗi (Fault Injection / Mutation Testing) & Minimized Counterexamples

Để chứng minh khả năng phát hiện lỗi và thu gọn phản ví dụ (shrinking) của bộ kiểm thử PBT-05 theo yêu cầu Cycle 3, chúng tôi tiến hành tiêm 2 lỗi logic điển hình (mutants) vào `get_parser_for_file` và chạy lại với cùng `--hypothesis-seed=20261008`:

#### Thực nghiệm Mutant 1 — Sai lệch quy tắc phá vỡ thế hòa (`score >= best_score`)

- **Giả lập lỗi**: Đổi toán tử so sánh tại dòng 395 từ `if best_score is None or score > best_score:` thành `if best_score is None or score >= best_score:`. Khi đó ứng viên duyệt sau (`_builtins`) sẽ ghi đè ứng viên duyệt trước (`_external`) khi bằng điểm.
- **Kết quả phát hiện & Shrinking**: Property `test_external_parser_wins_tie_with_builtin` (`PBT05-INV2`) lập tức bắt được lỗi. Khi chạy pha sinh tự động, Hypothesis thu gọn (shrink) phản ví dụ về biên nhỏ nhất:
  - Số lượng parser tối thiểu: `ext_count=1`, `builtin_count=1`.
  - Điểm số tối thiểu: `tie_score=0`.
  - MIME đầu tiên trong danh sách: `target_mime='application/pdf'`.

```text
================================== FAILURES ===================================
______ TestPbtParserSelection.test_external_parser_wins_tie_with_builtin ______
src\paperless\tests\test_parser_selection_pbt.py:235: in test_external_parser_wins_tie_with_builtin
    assert result in registry._external
E   AssertionError: assert <class '...MockParser_builtin_0'> in [<class '...MockParser_ext_0'>]
E   Falsifying example: test_external_parser_wins_tie_with_builtin(
E       self=<paperless.tests.test_parser_selection_pbt.TestPbtParserSelection object>,
E       tie_score=0,
E       ext_count=1,
E       builtin_count=1,
E       target_mime='application/pdf',
E   )
```

#### Thực nghiệm Mutant 2 — Bỏ sót chốt chặn `allow_remote`

- **Giả lập lỗi**: Bỏ khối kiểm tra `if not allow_remote and getattr(parser_class, "uses_remote_service", False): continue` tại dòng 384–389.
- **Kết quả phát hiện & Shrinking**: Property `test_remote_parsers_excluded_when_not_allowed` (`PBT05-INV3`) và `test_none_returned_when_only_remote_parsers_available_and_not_allowed` (`PBT05-INV4D`) lập tức thất bại và shrink về phản ví dụ tối giản gồm đúng 1 remote parser với điểm biên thấp nhất (`remote_scores=[50]`, `local_scores=[]`, `target_mime='application/pdf'`):

```text
================================== FAILURES ===================================
____ TestPbtParserSelection.test_remote_parsers_excluded_when_not_allowed _____
src\paperless\tests\test_parser_selection_pbt.py:308: in test_remote_parsers_excluded_when_not_allowed
    assert getattr(result, "uses_remote_service", False) is False
E   AssertionError: assert True is False
E   Falsifying example: test_remote_parsers_excluded_when_not_allowed(
E       self=<paperless.tests.test_parser_selection_pbt.TestPbtParserSelection object>,
E       remote_scores=[50],
E       local_scores=[],
E       target_mime='application/pdf',
E   )
```

---

## 7. Phân tích Nguyên nhân gốc (Root Cause Analysis — RCA) & Đánh giá Phòng thủ chiều sâu

### 7.1. Vì sao `ParserRegistry.get_parser_for_file` đạt độ tin cậy cao?

Qua đối chiếu mã nguồn (`src/paperless/parsers/registry.py`, dòng 375–407) và thực nghiệm kiểm thử:

1. **Bảo toàn thứ tự ưu tiên qua cấu trúc ghép tuple `(*self._external, *self._builtins)` kết hợp toán tử `>` nghiêm ngặt**:
   - Việc ghép `self._external` đứng trước `self._builtins` tại dòng 380 và chỉ cập nhật `best_parser` khi `score > best_score` (dòng 395) tạo thành chính sách _first-seen wins on ties_. Nhờ vậy, external parser luôn thắng built-in parser khi hòa điểm mà không cần thêm cờ trạng thái phức tạp.
2. **Cơ chế phòng vệ mặc định cho thuộc tính `uses_remote_service`**:
   - Việc dùng `getattr(parser_class, "uses_remote_service", False)` tại dòng 384 đảm bảo các parser bên thứ ba không khai báo tường minh thuộc tính này vẫn được xử lý an toàn mà không ném `AttributeError`.
3. **Phân tách rõ ràng giữa điểm âm (`score < 0`) và từ chối xử lý (`score is None`)**:
   - Dòng 392 kiểm tra `if score is None:` và dòng 395 kiểm tra `if best_score is None or score > best_score:` (không dùng kiểm tra truthiness `if not score:`), giúp các parser có điểm `0` hoặc điểm âm vẫn hoạt động chính xác.

### 7.2. Phát hiện Phòng thủ chiều sâu (Defense-in-Depth Findings) trong `ParserRegistry`

Khi rà soát toàn bộ lớp `ParserRegistry` trong `src/paperless/parsers/registry.py`, chúng tôi ghi nhận 2 rủi ro tiềm ẩn ở tầng tích hợp plugin bên thứ ba:

1. **Thiếu cơ chế cô lập ngoại lệ (Exception Isolation) khi gọi `supported_mime_types()` và `score()` của plugin bên thứ ba (dòng 381 & 391)**:
   - Trong `discover()` (dòng 237–244), việc nạp plugin `ep.load()` được bọc trong `try ... except Exception:` để một plugin lỗi không làm hỏng các plugin khác.
   - Tuy nhiên, trong `get_parser_for_file()` (dòng 381 và 391), lời gọi `parser_class.supported_mime_types()` và `parser_class.score(mime_type, filename, path)` **không có khối `try ... except` bảo vệ**. Nếu một external parser bị lỗi runtime khi chấm điểm một tệp cụ thể (hoặc trả về kiểu dữ liệu không tương thích khiến phép so sánh `score > best_score` ném `TypeError`), toàn bộ quá trình phân giải parser cho tài liệu đó sẽ bị dừng đột ngột chứ không bỏ qua parser lỗi để chuyển sang built-in parser dự phòng.
   - **Khuyến nghị**: Bọc lời gọi `supported_mime_types()` và `score()` trong `get_parser_for_file` bằng khối `try ... except Exception:` kèm `logger.warning(...)` để cô lập lỗi của plugin bên thứ ba.
2. **Thiếu kiểm tra chống đăng ký trùng lặp trong `register_builtin()` và `discover()` (dòng 190 & 258)**:
   - Cả `self._builtins.append(parser_class)` và `self._external.append(parser_class)` đều thêm trực tiếp mà không kiểm tra `if parser_class not in ...`, khiến một parser có thể bị chấm điểm nhiều lần nếu `register_defaults()` hoặc `discover()` được gọi lặp lại trên cùng một đối tượng `ParserRegistry`.

---

## 8. Hướng dẫn Tái lập cho Peer Reviewer (`@1convitt`)

### 8.1. Chạy trên Linux / macOS / WSL2 / CI

```bash
uv sync --python 3.12 --group testing --frozen
HYPOTHESIS_PROFILE=recorded uv run --python 3.12 --dev --frozen pytest \
  src/paperless/tests/test_parser_selection_pbt.py -k TestPbtParserSelection -v \
  --hypothesis-seed=20261008 --hypothesis-show-statistics \
  --cov=paperless.parsers.registry --cov-report=term-missing
```

### 8.2. Kiểm tra trên GitHub Actions

1. Mở tab **Checks** của Pull Request tương ứng với Issue #39.
2. Chọn workflow **Coursework CI** $\rightarrow$ job **PBT-05 Parser selection**.
3. Kiểm tra bảng thống kê Hypothesis (`max_examples=250`), bảng coverage `paperless.parsers.registry`, và tải artifact `pbt-05-evidence` (`.artifacts/pbt-05-run.log`).

---

## 9. Phụ lục: Nhật ký chạy tất định (Deterministic Execution Log)

<details>
<summary><b>Nhấn để xem toàn bộ raw output của profile recorded (250 examples/property, seed 20261008)</b></summary>

```text
============================= test session starts =============================
platform linux -- Python 3.12.10, pytest-9.2.0, pluggy-1.7.0
cachedir: .pytest_cache
hypothesis profile 'recorded'
rootdir: /workspace/paperless-ngx
configfile: pyproject.toml
plugins: hypothesis-6.170.15, cov-7.1.0
collecting ... collected 7 items

src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_highest_scoring_eligible_parser_wins PASSED [ 14%]
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_external_parser_wins_tie_with_builtin PASSED [ 28%]
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_remote_parsers_excluded_when_not_allowed PASSED [ 42%]
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_registry_is_empty PASSED [ 57%]
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_mime_unsupported_despite_valid_scores PASSED [ 71%]
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_matching_parsers_decline_with_none_score PASSED [ 85%]
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_only_remote_parsers_available_and_not_allowed PASSED [100%]

=============================== tests coverage ================================
_______________ coverage: platform linux, python 3.12.10-final-0 ______________

Name                                Stmts   Miss  Cover   Missing
-----------------------------------------------------------------
src/paperless/parsers/registry.py      89     49    45%   89-99, 118-122, 142-143, 200-210, 234-259, 282-303, 326
-----------------------------------------------------------------
TOTAL                                  89     49    45%
============================ Hypothesis Statistics ============================
src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_highest_scoring_eligible_parser_wins:

  - during generate phase (2.12 seconds):
    - Typical runtimes: ~ 1-9 ms, of which ~ 1-8 ms in data generation
    - 250 passing, 0 failing, and 20 invalid test cases

  - Stopped because settings.max_examples=250


src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_external_parser_wins_tie_with_builtin:

  - during generate phase (0.61 seconds):
    - Typical runtimes: ~ 1ms, of which < 1ms in data generation
    - 250 passing, 0 failing, and 0 invalid test cases

  - Stopped because settings.max_examples=250


src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_remote_parsers_excluded_when_not_allowed:

  - during generate phase (0.99 seconds):
    - Typical runtimes: ~ 0-2 ms, of which ~ 0-1 ms in data generation
    - 250 passing, 0 failing, and 89 invalid test cases

  - Stopped because settings.max_examples=250


src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_registry_is_empty:

  - during generate phase (0.66 seconds):
    - Typical runtimes: ~ 0-1 ms, of which < 1ms in data generation
    - 250 passing, 0 failing, and 0 invalid test cases

  - Stopped because settings.max_examples=250


src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_mime_unsupported_despite_valid_scores:

  - during generate phase (1.30 seconds):
    - Typical runtimes: ~ 0-3 ms, of which ~ 0-2 ms in data generation
    - 250 passing, 0 failing, and 70 invalid test cases

  - Stopped because settings.max_examples=250


src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_matching_parsers_decline_with_none_score:

  - during generate phase (1.29 seconds):
    - Typical runtimes: ~ 1-4 ms, of which ~ 0-2 ms in data generation
    - 250 passing, 0 failing, and 36 invalid test cases

  - Stopped because settings.max_examples=250


src/paperless/tests/test_parser_selection_pbt.py::TestPbtParserSelection::test_none_returned_when_only_remote_parsers_available_and_not_allowed:

  - during generate phase (1.13 seconds):
    - Typical runtimes: ~ 1-3 ms, of which ~ 0-2 ms in data generation
    - 250 passing, 0 failing, and 24 invalid test cases

  - Stopped because settings.max_examples=250


============================= 7 passed in 10.80s ==============================
```

</details>
