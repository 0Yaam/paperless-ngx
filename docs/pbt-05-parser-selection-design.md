# PBT-05: Thiết kế kiểm thử Lựa chọn Parser (Parser Selection Design)

## Thông tin chung (Overview)

- **Target under test (Đối tượng kiểm thử)**: `paperless.parsers.registry.ParserRegistry.get_parser_for_file`
- **Source file (Tệp mã nguồn)**: [`src/paperless/parsers/registry.py`](../src/paperless/parsers/registry.py)
- **Suite**: PBT-05 Parser selection (Lựa chọn bộ phân tích tài liệu)
- **Owner**: Phùng Nguyễn Hoài Bo ([@HubertPhung](https://github.com/HubertPhung))
- **Reviewer**: Vũ Thế Huỳnh ([@1convitt](https://github.com/1convitt))
- **Issue**: [#14 — [Cycle 1][Bo] Design PBT-05 Parser selection](https://github.com/0Yaam/paperless-ngx/issues/14)
- **Implementation PR**: [#18 — test: design and cover parser selection properties](https://github.com/0Yaam/paperless-ngx/pull/18)

---

## 1. Bối cảnh hệ thống và Mục tiêu kiểm thử

### 1.1 Bài toán trong Paperless-ngx

Paperless-ngx tiếp nhận nhiều định dạng tài liệu khác nhau (PDF, TIFF, ảnh scan, văn bản thô, email...). Để trích xuất nội dung và metadata, hệ thống sử dụng nhiều lớp parser chuyên biệt.

Trong `ParserRegistry`, các parser được chia thành hai danh sách:

- `_builtins`: Các parser mặc định tích hợp sẵn (Tesseract OCR, Text, Mail, Rasterised, Remote Gotenberg/Tika).
- `_external`: Các parser mở rộng do bên thứ ba phát triển qua entry points (`paperless_ngx.parsers`).

Hàm `get_parser_for_file(mime_type, filename, path=None, allow_remote=True)` đảm nhận nhiệm vụ trọng tài, lựa chọn parser thích hợp nhất theo thứ tự ưu tiên:

1. **Khớp định dạng MIME**: Chỉ parser hỗ trợ `mime_type` mới được xét.
2. **Bảo vệ môi trường cô lập**: Khi `allow_remote=False`, cấm hoàn toàn parser có `uses_remote_service=True`.
3. **Chấm điểm phù hợp**: Gọi `score()`. Nếu trả về `None`, parser từ chối xử lý.
4. **Điểm cao nhất chiến thắng**: Parser có điểm số nguyên lớn nhất sẽ được chọn.
5. **Ưu tiên External Parser khi hòa điểm**: Nếu parser mở rộng và parser mặc định cùng điểm cao nhất, parser mở rộng được chọn để cho phép người dùng ghi đè cấu hình mặc định.
6. **Suy thoái êm dịu (Graceful Fallback)**: Trả về `None` an toàn nếu không tìm thấy parser phù hợp.

---

## 2. Chi tiết 7 Test Case (Property-Based Tests)

### Test Case 1: Điểm số hợp lệ cao nhất luôn chiến thắng (`PBT05-INV1-MAX-SCORE`)

- **Tên hàm test**: `test_highest_scoring_eligible_parser_wins`
- **Mục tiêu**: Đảm bảo giải thuật lựa chọn luôn chọn đúng parser đạt điểm số nguyên lớn nhất trong tập các parser hợp lệ.
- **Tiền điều kiện**: Tập ứng viên hợp lệ không rỗng. Ứng viên hợp lệ phải khớp MIME, không vi phạm cấm remote, và có điểm khác `None`.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Sinh từ 1 đến 8 parser ngẫu nhiên với điểm trong đoạn $[-100, 100]$ hoặc `None`.
  - Phân bổ ngẫu nhiên vào `_external` hoặc `_builtins`.
  - Cờ `uses_remote_service` ngẫu nhiên (True/False).
  - Cờ `allow_remote` ngẫu nhiên (True/False).
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - Parser được chọn phải thuộc tập hợp lệ và có điểm số bằng đúng điểm lớn nhất trong tập hợp lệ:
    $$\text{score}(P^*) = \max_{P \in \mathcal{E}} \text{score}(P)$$
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: So sánh nhầm dấu `<` (chọn parser điểm thấp nhất) hoặc không cập nhật biến lưu điểm cao nhất.
  - _Đầu vào mẫu_: `ParserA(score=10)` và `ParserB(score=90)`.
  - _Kết quả sai_: Hàm trả về `ParserA`.

---

### Test Case 2: Parser mở rộng thắng khi hòa điểm với parser mặc định (`PBT05-INV2-TIE-BREAK-EXTERNAL`)

- **Tên hàm test**: `test_external_parser_wins_tie_with_builtin`
- **Mục tiêu**: Đảm bảo cơ chế mở rộng hệ thống — khi external parser và built-in parser có cùng điểm số cao nhất, external parser luôn được ưu tiên chọn.
- **Tiền điều kiện**: Cả external parser và built-in parser cùng đạt điểm số cao nhất.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Sinh điểm hòa ngẫu nhiên $S_{tie} \in [-50, 50]$.
  - Sinh từ 1 đến 4 external parser và từ 1 đến 4 built-in parser cùng có điểm $S_{tie}$ và cùng hỗ trợ `target_mime`.
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - Parser chiến thắng bắt buộc phải thuộc `registry._external` và không thuộc `registry._builtins`.
  - Parser chiến thắng là phần tử external đầu tiên trong thứ tự duyệt.
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: Duyệt danh sách built-in trước external, hoặc dùng toán tử `>=` khiến phần tử sau ghi đè phần tử trước.
  - _Đầu vào mẫu_: `ExtParser(score=50)` và `BuiltinParser(score=50)`.
  - _Kết quả sai_: Hàm trả về `BuiltinParser`.

---

### Test Case 3: Loại bỏ hoàn toàn Remote Parser khi cấm kết nối ngoài (`PBT05-INV3-REMOTE-EXCLUSION`)

- **Tên hàm test**: `test_remote_parsers_excluded_when_not_allowed`
- **Mục tiêu**: Bảo mật dữ liệu trong môi trường nội bộ — khi `allow_remote=False`, mọi parser khai báo `uses_remote_service=True` phải bị loại bỏ tuyệt đối, không được chọn dù điểm số của nó cao đến đâu.
- **Tiền điều kiện**: `allow_remote=False` và registry có chứa parser khai báo `uses_remote_service=True`.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Sinh từ 1 đến 3 remote parser có điểm số rất cao ($[50, 100]$).
  - Sinh từ 0 đến 3 local parser có điểm số thấp hơn ($[-20, 40]$) hoặc trả về `None`.
  - Gọi hàm với `allow_remote=False`.
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - Kết quả trả về không bao giờ có cờ `uses_remote_service=True`.
  - Nếu không có local parser nào có điểm hợp lệ, kết quả phải là `None`.
  - Nếu có local parser hợp lệ, local parser có điểm cao nhất sẽ được chọn.
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: Bỏ sót điều kiện kiểm tra `allow_remote`, chọn nhầm remote parser do điểm số cao hơn.
  - _Đầu vào mẫu_: `CloudOCR(score=100, remote=True)` và `LocalParser(score=20, remote=False)` với `allow_remote=False`.
  - _Kết quả sai_: Hàm trả về `CloudOCR`.

---

### Test Case 4: Trả về None khi Registry rỗng (`PBT05-INV4A-EMPTY-REGISTRY`)

- **Tên hàm test**: `test_none_returned_when_registry_is_empty`
- **Mục tiêu**: Kiểm tra tính bền bỉ khi hệ thống chưa nạp parser nào hoặc registry bị xóa rỗng.
- **Tiền điều kiện**: Registry không chứa bất kỳ parser nào.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Registry rỗng (0 parser).
  - MIME type ngẫu nhiên, tên file ngẫu nhiên, cờ `allow_remote` ngẫu nhiên.
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - `result is None`.
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: Hàm bị văng ngoại lệ `IndexError` hoặc `StopIteration` khi truy cập danh sách rỗng.

---

### Test Case 5: Trả về None khi không hỗ trợ định dạng MIME (`PBT05-INV4B-UNSUPPORTED-MIME`)

- **Tên hàm test**: `test_none_returned_when_mime_unsupported_despite_valid_scores`
- **Mục tiêu**: Kiểm chứng việc từ chối tài liệu khi không có parser nào hỗ trợ định dạng MIME của file, dù các parser đều có điểm số hợp lệ và chạy cục bộ.
- **Tiền điều kiện**: Tồn tại parser cục bộ có điểm số dương ($[1, 100]$), nhưng không parser nào khai báo hỗ trợ `target_mime`.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Sinh từ 1 đến 5 parser hỗ trợ MIME không liên quan (`other/unsupported-mime`).
  - Truy vấn với `target_mime` ngẫu nhiên.
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - `result is None`.
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: Tự ý suy đoán theo phần mở rộng của tên tệp (`.bin`) hoặc mặc định chọn parser đầu tiên.

---

### Test Case 6: Trả về None khi các parser đều từ chối tài liệu (`PBT05-INV4C-DECLINING-SCORE`)

- **Tên hàm test**: `test_none_returned_when_matching_parsers_decline_with_none_score`
- **Mục tiêu**: Đảm bảo xử lý chính xác khi parser đúng MIME nhưng chủ động từ chối tài liệu bằng cách trả về điểm `None`.
- **Tiền điều kiện**: Các parser đều khớp `target_mime`, nhưng tất cả đều trả về `score() == None`.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Sinh từ 1 đến 5 parser khớp định dạng `application/pdf`, nhưng điểm số đều là `None`.
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - `result is None`.
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: Coi `score is None` tương đương điểm 0 và chọn nhầm parser đầu tiên.

---

### Test Case 7: Trả về None khi chỉ có Remote Parser nhưng bị cấm (`PBT05-INV4D-BLOCKED-REMOTE-ONLY`)

- **Tên hàm test**: `test_none_returned_when_only_remote_parsers_available_and_not_allowed`
- **Mục tiêu**: Đảm bảo hệ thống không linh động chọn remote parser khi chạy offline nếu không có local parser nào khác.
- **Tiền điều kiện**: Toàn bộ parser ứng viên đều là remote parser (`uses_remote_service=True`), có điểm số dương ($[10, 100]$), nhưng gọi với `allow_remote=False`.
- **Dữ liệu sinh tự động (Hypothesis Strategy)**:
  - Sinh từ 1 đến 5 remote parser khớp MIME, điểm từ 10 đến 100, gọi với `allow_remote=False`.
- **Oracle (Tiêu chuẩn kiểm chứng)**:
  - `result is None`.
- **Phản ví dụ giả định (Counterexample)**:
  - _Lỗi code_: Hệ thống tự ý nới lỏng và chọn remote parser vì không còn giải pháp nào khác.

---

## 3. Cấu hình thực hiện và Tính tái lập trong CI

- **Số lượng mẫu thử**: `@settings(max_examples=200, deadline=None)` cho mỗi property.
- **Seed cố định tái lập**: `--hypothesis-seed=20261008`.
- **Thống kê chi tiết**: `--hypothesis-show-statistics`.
- **Lưu trữ bằng chứng**: Log thực hiện được ghi vào `.artifacts/pbt-05-run.log` và tải lên CI dưới tên artifact `pbt-05-evidence`.
- **Môi trường cách ly**: Mỗi lần lặp kiểm thử tạo một đối tượng `ParserRegistry` mới hoàn toàn độc lập, không làm ảnh hưởng đối tượng singleton toàn cục.

---

## 4. Bảng ma trận đối chiếu (Traceability Matrix)

| Mã Invariant  | Tên hàm kiểm thử                                                        | Tính chất nghiệp vụ kiểm chứng                        |  Seed CI   |  Trạng thái   |
| :------------ | :---------------------------------------------------------------------- | :---------------------------------------------------- | :--------: | :-----------: |
| `PBT05-INV1`  | `test_highest_scoring_eligible_parser_wins`                             | Parser hợp lệ có điểm số cao nhất luôn thắng          | `20261008` | Đạt (200 mẫu) |
| `PBT05-INV2`  | `test_external_parser_wins_tie_with_builtin`                            | Parser mở rộng thắng khi hòa điểm với parser mặc định | `20261008` | Đạt (200 mẫu) |
| `PBT05-INV3`  | `test_remote_parsers_excluded_when_not_allowed`                         | Cấm triệt để parser từ xa khi chạy offline            | `20261008` | Đạt (200 mẫu) |
| `PBT05-INV4A` | `test_none_returned_when_registry_is_empty`                             | Xử lý an toàn khi registry rỗng                       | `20261008` | Đạt (200 mẫu) |
| `PBT05-INV4B` | `test_none_returned_when_mime_unsupported_despite_valid_scores`         | Xử lý an toàn khi không khớp định dạng MIME           | `20261008` | Đạt (200 mẫu) |
| `PBT05-INV4C` | `test_none_returned_when_matching_parsers_decline_with_none_score`      | Xử lý an toàn khi parser từ chối file (điểm None)     | `20261008` | Đạt (200 mẫu) |
| `PBT05-INV4D` | `test_none_returned_when_only_remote_parsers_available_and_not_allowed` | Xử lý an toàn khi chỉ có remote parser nhưng bị cấm   | `20261008` | Đạt (200 mẫu) |

---

## 5. Phân định phạm vi trong K01 (Scope Demarcation)

- **PBT-04 (@1convitt - Huỳnh)**: Kiểm thử từ điển ánh xạ tĩnh giữa định dạng MIME và phần mở rộng tệp trong `documents.parsers`.
- **PBT-05 (@HubertPhung - Bo)**: Kiểm thử động cơ chế lựa chọn parser, chấm điểm và cách ly remote trong `ParserRegistry.get_parser_for_file`.
- **Kết luận**: Hai bộ kiểm thử hoàn toàn độc lập về đối tượng, logic và phạm vi mã nguồn.
