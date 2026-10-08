# Thiết kế PBT-05: Parser Selection (Cycle 1)

## Thông tin chung
- **Target under test**: `paperless.parsers.registry.ParserRegistry.get_parser_for_file`
- **Tập tin nguồn**: [`src/paperless/parsers/registry.py`](../src/paperless/parsers/registry.py)
- **Suite**: PBT-05 Parser selection
- **Owner**: Phùng Nguyễn Hoài Bo ([@HubertPhung](https://github.com/HubertPhung))
- **Reviewer**: Vũ Thế Huỳnh ([@1convitt](https://github.com/1convitt))
- **Issue**: [#14 — [Cycle 1][Bo] Design PBT-05 Parser selection](https://github.com/0Yaam/paperless-ngx/issues/14)

---

## 1. Giới thiệu và Bối cảnh kiến trúc
Trong Paperless-ngx, `ParserRegistry` quản lý hai nhóm parser:
- `_builtins`: Parser tích hợp sẵn (ví dụ: Text, Mail, Rasterised, Remote, Tika).
- `_external`: Parser từ bên thứ ba (third-party plugins nạp qua Python entrypoint).

Khi người dùng upload hoặc nạp tài liệu, phương thức `get_parser_for_file` chịu trách nhiệm chọn parser tối ưu nhất dựa trên:
1. Định dạng MIME (`supported_mime_types()`).
2. Cờ xử lý dịch vụ từ xa (`allow_remote` so với thuộc tính `uses_remote_service`).
3. Điểm số độ ưu tiên (`score(mime_type, filename, path)`).
4. Quy tắc hòa điểm (External parser được ưu tiên hơn Built-in parser).

---

## 2. Các Bất biến Độc lập (Independent Invariants)

### Bất biến 1: Điểm số hợp lệ cao nhất luôn chiến thắng (`PBT05-INV1-MAX-SCORE`)
- **Mục tiêu**: Đảm bảo thuật toán chọn parser luôn tìm ra parser có điểm số strictly lớn nhất trong tập hợp các candidate hợp lệ.
- **Tiền điều kiện (Precondition)**:
  - Tập candidate hợp lệ $\mathcal{E}$ không rỗng.
  - Một candidate $P$ là hợp lệ khi và chỉ khi:
    - $\text{mime\_type} \in P.\text{supported\_mime\_types}()$
    - Nếu $\text{allow\_remote} = \text{False}$ thì $\operatorname{getattr}(P, \text{"uses\_remote\_service"}, \text{False}) == \text{False}$.
    - $P.\text{score}(\text{mime\_type}, \text{filename}, \text{path}) \ne \text{None}$.
- **Strategy (Sinh dữ liệu Hypothesis)**:
  - Sinh danh sách từ 1 đến 8 parser giả lập với cấu hình:
    - Điểm số ngẫu nhiên nguyên trong khoảng $[-100, 100]$ hoặc `None`.
    - Danh sách MIME hỗ trợ chọn từ tập `{"text/plain", "application/pdf", "image/png", "text/csv"}`.
    - Cờ `uses_remote_service`: `st.booleans()`.
    - Phân bổ ngẫu nhiên vào `_external` hoặc `_builtins`.
  - Sinh tham số truy vấn: `mime_type`, `filename` (chuỗi bounded 1-30 ký tự), `allow_remote: st.booleans()`.
- **Reference Oracle (Hàm mẫu đối chiếu)**:
  $$\text{best\_score} = \max_{P \in \mathcal{E}} P.\text{score}(\text{mime\_type}, \text{filename}, \text{path})$$
  $$\text{Oracle: } P^* \in \mathcal{E} \quad \land \quad P^*.\text{score}(\dots) == \text{best\_score}$$
- **Sample Counterexample (Trường hợp vi phạm mẫu)**:
  - *Lỗi giả định trong code*: Dùng sai toán tử `score < best_score` (chọn điểm thấp nhất) hoặc không cập nhật `best_score` khi gặp điểm lớn hơn.
  - *Input phản ví dụ tối thiểu*: Registry gồm `BuiltinA(score=10)` và `BuiltinB(score=20)` cho file `.txt`.
  - *Hành vi sai*: Trả về `BuiltinA` (score 10) thay vì `BuiltinB` (score 20).

---

### Bất biến 2: External Parser thắng khi hòa điểm với Built-in (`PBT05-INV2-TIE-BREAK-EXTERNAL`)
- **Mục tiêu**: Xác nhận chính sách ưu tiên plugin mở rộng: khi một plugin bên ngoài và một parser tích hợp cùng đạt điểm số cao nhất, plugin bên ngoài luôn được chọn.
- **Tiền điều kiện (Precondition)**:
  - Cả external parser và built-in parser đều hợp lệ và cùng đạt điểm số lớn nhất $S_{tie} = \max_{P \in \mathcal{E}} P.\text{score}$.
- **Strategy (Sinh dữ liệu Hypothesis)**:
  - Sinh một điểm số hòa $S_{tie} \in [-50, 50]$.
  - Sinh 1..4 external parsers và 1..4 built-in parsers cùng hỗ trợ `mime_type`.
  - Gán ít nhất một external parser và một built-in parser có điểm số đúng bằng $S_{tie}$.
  - Đảm bảo tất cả các parser còn lại có điểm số $\le S_{tie}$.
- **Reference Oracle (Hàm mẫu đối chiếu)**:
  $$\text{Oracle: } P^* \in \text{registry.\_external} \quad \land \quad P^*.\text{score}(\dots) == S_{tie}$$
  (Parser chiến thắng bắt buộc phải nằm trong `registry._external`, không bao giờ thuộc `registry._builtins`).
- **Sample Counterexample (Trường hợp vi phạm mẫu)**:
  - *Lỗi giả định trong code*: Thứ tự duyệt bị đảo thành `(*self._builtins, *self._external)` hoặc phép so sánh dùng `score >= best_score` (last-seen wins khiến built-in ghi đè external).
  - *Input phản ví dụ tối thiểu*: `ExternalParser(score=10)` và `BuiltinParser(score=10)`.
  - *Hành vi sai*: Trả về `BuiltinParser` thay vì `ExternalParser`.

---

### Bất biến 3: Loại trừ Remote Parser khi `allow_remote=False` (`PBT05-INV3-REMOTE-EXCLUSION`)
- **Mục tiêu**: Đảm bảo tính bảo mật và quyền riêng tư dữ liệu: khi người dùng hoặc hệ thống tắt remote processing (`allow_remote=False`), tài liệu không bao giờ được gửi tới parser từ xa.
- **Tiền điều kiện (Precondition)**:
  - `allow_remote = False`.
  - Registry có chứa parser có `uses_remote_service = True`.
- **Strategy (Sinh dữ liệu Hypothesis)**:
  - Sinh 1..3 remote parsers với `uses_remote_service = True` và điểm số vượt trội (ví dụ 100).
  - Sinh 0..3 local parsers (`uses_remote_service = False` hoặc không có thuộc tính này) với điểm thấp hơn ($1..50$) hoặc từ chối (`None`).
  - Gọi `get_parser_for_file(..., allow_remote=False)`.
- **Reference Oracle (Hàm mẫu đối chiếu)**:
  $$\text{Oracle: } P^* \text{ is None} \quad \lor \quad \operatorname{getattr}(P^*, \text{"uses\_remote\_service"}, \text{False}) == \text{False}$$
  Ngoài ra, nếu không có local parser nào hợp lệ thì $P^*$ phải là `None`. Nếu có ít nhất một local parser hợp lệ thì $P^*$ phải là local parser có điểm cao nhất.
- **Sample Counterexample (Trường hợp vi phạm mẫu)**:
  - *Lỗi giả định trong code*: Quên kiểm tra điều kiện `not allow_remote and getattr(parser_class, "uses_remote_service", False)`.
  - *Input phản ví dụ tối thiểu*: `RemoteParser(score=100, uses_remote_service=True)`, `LocalParser(score=5)`, `allow_remote=False`.
  - *Hành vi sai*: Trả về `RemoteParser` vì điểm cao hơn.

---

### Bất biến 4: Trả về None an toàn khi không có parser hợp lệ (`PBT05-INV4-NONE-FALLBACK`)
- **Mục tiêu**: Đảm bảo tính toàn vẹn (Graceful Fallback) khi không có bất kỳ parser nào nhận xử lý tài liệu.
- **Tiền điều kiện (Precondition)**:
  - Tập candidate hợp lệ $\mathcal{E} = \emptyset$ (do không khớp MIME, hoặc tất cả đều trả về `score = None`, hoặc tất cả là remote khi `allow_remote=False`).
- **Strategy (Sinh dữ liệu Hypothesis)**:
  - Sinh các parser có MIME không trùng khớp với file đầu vào, hoặc hàm `score()` luôn trả về `None`, hoặc chỉ gồm remote parser với `allow_remote=False`.
- **Reference Oracle (Hàm mẫu đối chiếu)**:
  $$\text{Oracle: } P^* \text{ is None}$$
- **Sample Counterexample (Trường hợp vi phạm mẫu)**:
  - *Lỗi giả định trong code*: Trả về parser mặc định đầu tiên hoặc throw exception `IndexError` khi không tìm thấy.
  - *Input phản ví dụ tối thiểu*: Registry chỉ có `DecliningParser(score=None)`.
  - *Hành vi sai*: Trả về `DecliningParser` thay vì `None`.

---

## 3. Bounded Input Parameters & Ổn định CI
- **Số lượng ví dụ**: `@settings(max_examples=200, deadline=None)` cho mỗi property.
- **Giới hạn số lượng parser**: Danh sách từ 0 đến 8 class (đảm bảo thời gian chạy mỗi test case $< 1$ ms).
- **Giới hạn điểm số**: Số nguyên trong khoảng $[-100, 100]$.
- **Môi trường cách ly**: Mỗi lần chạy khởi tạo một `ParserRegistry()` độc lập, không dùng biến singleton toàn cục để tránh race condition giữa các test case.

---

## 4. Phân định ranh giới K01 (Scope Demarcation)
- **PBT-04 (@1convitt - Reviewer)**: Tập trung vào `documents.parsers` kiểm tra tính nhất quán giữa danh sách MIME type và phần mở rộng file (extension mapping tĩnh).
- **PBT-05 (@HubertPhung - Owner)**: Tập trung vào `paperless.parsers.registry.ParserRegistry.get_parser_for_file` kiểm tra logic trọng tài, so khớp điểm số, ưu tiên external và bảo vệ remote (arbitration động).
- **Kết luận**: Hai suite có target function, phạm vi và mục đích kiểm thử hoàn toàn tách biệt, không xảy ra chồng lấn kiểm thử.
