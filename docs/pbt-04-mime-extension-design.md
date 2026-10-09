# PBT-04 — Thiết kế MIME-extension (Cycle 1)

## Phạm vi và baseline

- Issue: [#13](https://github.com/0Yaam/paperless-ngx/issues/13).
- Owner: Huỳnh (`@1convitt`); reviewer: Vương (`@vuong123s`).
- Baseline upstream: `da3de299f`; HEAD khảo sát trên `dev`: `b6f9328534eca17630e93a99406288058bc1d3a3`.
- Target: `is_mime_type_supported`, `get_default_file_extension`, `is_file_ext_supported`, `get_supported_file_extensions` trong `src/documents/parsers.py`.
- Cycle 1 chỉ thiết kế; chưa có kết quả chạy PBT-04 hoặc xác nhận defect. Không thay public API, dependency hay production code.

Registry chỉ là nguồn mapping và dependency cần cô lập. Scoring, tie-break, external parser thắng và `allow_remote` thuộc PBT-05; OCR, đọc tài liệu và gọi dịch vụ từ xa ngoài phạm vi.

## Hiện trạng và hợp đồng kiểm thử

`get_default_file_extension` ưu tiên mapping của parser được chọn, rồi fallback sang `mimetypes.guess_extension`. Vì vậy MIME có extension không đồng nghĩa MIME được hỗ trợ: `application/zip` có thể trả `.zip` dù không có parser phù hợp.

`get_supported_file_extensions` hợp các default extension và alias từ `mimetypes` của **mọi parser đăng ký**. Parser Tika vẫn khai báo mapping khi Tika tắt; tập extension không phải tập MIME đang chọn được parser. Không yêu cầu mỗi extension chỉ ánh xạ về một MIME hoặc default extension bằng extension thư viện chuẩn chọn.

`is_file_ext_supported` nhận hậu tố có dấu chấm, chuyển lowercase rồi kiểm tra membership; chuỗi rỗng trả `False`. Không đặt yêu cầu case-insensitive cho MIME, không tự thêm dấu chấm hoặc trim input.

## Ba invariant độc lập

### P1 — MIME được hỗ trợ có extension hợp lệ

- **Strategy:** lấy snapshot mapping built-in trong registry cô lập; dùng `sampled_from` danh sách MIME được sắp xếp. Kiểm tra hai cấu hình Tika bật/tắt riêng biệt. Bổ sung kiểm tra hữu hạn toàn bộ MIME để không phụ thuộc xác suất sampling.
- **Precondition:** chỉ áp dụng cho MIME mà `is_mime_type_supported(mime)` trả `True`; tính danh sách đủ điều kiện trước khi sinh example, không dùng `assume` lặp lại. Không khởi tạo parser hoặc gọi `parse`.
- **Oracle:** `get_default_file_extension(mime)` là chuỗi bắt đầu bằng `.`, dài ít nhất 2, không chứa khoảng trắng, `/` hoặc `\`; kết quả thuộc `get_supported_file_extensions()`.
- **Pass/fail:** tất cả MIME đủ điều kiện thỏa các điều kiện trên; một MIME vi phạm làm test fail. Assert danh sách đủ điều kiện không rỗng để tránh pass rỗng.
- **Counterexample minh họa:** `application/pdf` được hỗ trợ nhưng extension là `""` hoặc `"pdf"`.

P1 kiểm tra hợp đồng đầu ra của helper trên dữ liệu built-in; không xác minh parser nào phải thắng khi nhiều parser nhận cùng MIME.

### P2 — Kiểm tra extension không phụ thuộc hoa/thường

- **Strategy:** snapshot `S` là tập extension hỗ trợ, sắp xếp trước khi sampling. Sinh input từ extension trong `S`, alias và chuỗi ASCII tùy ý dài 0–32, dùng alphabet chữ cái, chữ số, dấu chấm, gạch dưới, gạch ngang và khoảng trắng. Sinh biến thể bằng lựa chọn uppercase/lowercase độc lập ở từng chữ cái.
- **Precondition:** input là chuỗi ASCII trong giới hạn; không cần extension phải được hỗ trợ. Không sinh `None` vì nằm ngoài kiểu đầu vào.
- **Oracle:** với input `e`, kết quả bằng `bool(e) and e.lower() in S`; kết quả cho biến thể case bằng kết quả cho `e`. Snapshot lấy trước khi gọi helper, không dùng helper để tính expected membership.
- **Pass/fail:** kiểm tra cả membership chính xác và bất biến case; phải bắt được cả false positive và false negative.
- **Counterexample minh họa:** `.pdf` trả `True` nhưng `.PDF` trả `False`; hoặc `""` trả `True`.

Kiểm tra membership phân biệt P2 với một assertion case-invariance đơn thuần vốn có thể bỏ sót implementation luôn trả `True` hoặc luôn trả `False`.

### P3 — Tập extension nhất quán với mapping và alias

- **Strategy:** sinh 0–5 parser giả, mỗi parser có 0–8 mapping. MIME có dạng `application/x-pbt-<token>` với token ASCII lowercase dài 1–12; key duy nhất trong từng mapping, cho phép trùng giữa parser. Default extension có dấu chấm và token chữ/số lowercase dài 1–15. Bảng alias độc lập theo MIME, mỗi MIME có 0–4 alias cùng giới hạn extension; cho phép trùng alias và default.
- **Precondition:** registry và bảng alias cố định trong một example. Thay `documents.parsers.get_parser_registry` bằng nguồn parser giả chỉ cung cấp `all_parsers`, và thay `documents.parsers.mimetypes.guess_all_extensions` bằng lookup bảng alias. Không dùng registry scoring thật.
- **Oracle:** expected set tính từ dữ liệu strategy gốc: hợp mọi default extension và alias tương ứng từng MIME trong registry. Không gọi `get_supported_file_extensions` hoặc helper khác để tạo expected set.
- **Pass/fail:** actual set bằng expected set, không thiếu hoặc thêm phần tử; registry rỗng phải cho tập rỗng. Mapping của parser vẫn đóng góp mà không cần gọi `score`.
- **Counterexample minh họa:** mapping `image/jpeg → .jpg`, bảng alias gồm `.jpeg`, nhưng kết quả thiếu `.jpg` hoặc `.jpeg`. Strategy dùng MIME giả; ví dụ JPEG này minh họa cùng dạng lỗi trên MIME quen thuộc.

P3 kiểm tra phép hợp tập hợp, độc lập với P1 (extension mặc định cho MIME đủ điều kiện) và P2 (membership/case của input).

Các counterexample trong tài liệu là giả định dùng để giải thích fail condition, **không phải lỗi đã tái hiện**.

## Giới hạn, cô lập và tái lập

- Mỗi property dùng `max_examples=100`, `deadline=None`, giữ shrinking và health check mặc định. Không dùng suppression để che trạng thái fixture bị rò rỉ.
- P1 dùng mapping built-in hữu hạn tại baseline; lưu số parser/MIME trong evidence. P2 giới hạn chuỗi 32 ký tự; P3 tối đa 40 mapping và 4 alias mỗi MIME.
- Cô lập discovery plugin bên ngoài, reset registry trước/sau test, khôi phục settings và chặn cấu hình remote OCR. Với P1/P2, không cho môi trường máy chạy tự bổ sung plugin.
- P3 thay dependency trong context riêng của từng example, khôi phục khi context thoát kể cả assertion fail; không tích lũy parser hay alias giữa các example. Không dùng function-scoped fixture để giữ mutable example state.
- Không hardcode tập alias của host. P1/P2 lấy snapshot cùng môi trường; P3 dùng bảng alias giả xác định để kiểm tra logic.
- Môi trường nghiệm thu: Ubuntu/Python 3.12 theo Coursework CI, dependency theo `uv.lock`. Local Windows dùng WSL2/Linux vì `uv` của repo chỉ khai báo Linux/macOS.
- Chạy tái lập với `--hypothesis-seed=13`, ghi seed vào log. Khi fail, lưu output `Falsifying example` đã shrink và reproduction blob nếu Hypothesis cung cấp; seed không thay thế việc ghi phiên bản Python/Hypothesis và SHA.

## Lộ trình Cycle 2

1. Sau khi thiết kế được duyệt, thêm `TestMimeExtensionProperties` vào `src/documents/tests/test_parsers.py`; giữ regression tests hiện có. Không thêm dependency Hypothesis vì nhóm `testing` đã có.
2. Thêm ba property theo đặc tả trên và kiểm tra hữu hạn toàn bộ mapping built-in. Với fixture Tika, bảo đảm settings và registry được reset khi test kết thúc.
3. Regression cases: `.PDF`, `.JpEg`, chuỗi rỗng, `pdf` không có dấu chấm, extension ngoài tập; fallback `.zip` cho `application/zip` và MIME không có trong registry hoặc bảng stdlib; registry rỗng, default không có alias, alias bổ sung và extension trùng giữa parser. Expected fallback lấy từ bảng stdlib cô lập khi cần tránh phụ thuộc host.
4. Thêm job PBT-04 vào `.github/workflows/coursework-ci.yml`, dùng Ubuntu/Python 3.12 và phiên bản action/uv theo workflow hiện có. Chạy một worker để evidence dễ đọc; đặt timeout job 15 phút và upload log/JUnit kể cả khi fail.
5. Chạy suite riêng, sau đó toàn bộ file parser để phát hiện rò rỉ state. Lưu SHA, môi trường, lệnh, seed, thống kê Hypothesis, JUnit và counterexample nếu có. Báo lỗi production thành defect riêng trước khi đề xuất sửa.

Lệnh chạy suite sau khi triển khai Cycle 2:

```bash
uv sync --python 3.12 --group testing --frozen
uv run --python 3.12 --group testing --frozen pytest src/documents/tests/test_parsers.py -k TestMimeExtensionProperties -n 0 --hypothesis-seed=13 --hypothesis-show-statistics
uv run --python 3.12 --group testing --frozen pytest src/documents/tests/test_parsers.py -n 0 --hypothesis-seed=13
```

Không coi các lệnh này là bằng chứng đã chạy hoặc suite đã tồn tại ở Cycle 1.

### Trạng thái triển khai Cycle 2

Đã bổ sung `TestMimeExtensionProperties` vào file test parser và job PBT-04 vào Coursework CI. P1 kiểm tra Tika bật/tắt và duyệt toàn bộ mapping; P2 kiểm tra membership và biến thể case; P3 dùng registry/bảng alias sinh độc lập, gồm example registry rỗng và default không có alias. Các regression cases kiểm tra fallback không đồng nghĩa MIME được hỗ trợ.

P1/P2 dùng một registry built-in riêng, thay accessor của helper trong context và chặn `RemoteDocumentParser.score` để không đọc cấu hình DB/remote. Không sửa singleton toàn cục nên không cần reset singleton; dependency và settings tự khôi phục khi thoát context. Đây là cách cô lập thay cho phương án reset registry ở phần thiết kế.

Job CI chạy property rồi toàn bộ file parser, seed 13, một worker; lưu môi trường, log và JUnit dưới artifact `pbt-04-evidence`. Chưa có bằng chứng chạy pytest đầy đủ tại thời điểm triển khai local: máy Windows thiếu dependency testing, không có distro Linux phát triển và Docker engine chưa chạy. Checklist duyệt thiết kế vẫn chờ reviewer xác nhận.

Kiểm chứng local ngày 2026-10-09: cú pháp Python và YAML hợp lệ, `git diff --check` không có lỗi whitespace. P3 chạy cô lập trên phần thân AST thực tế của helper/test, với Hypothesis 6.168.5 và Python 3.12: 100 generated examples và 2 explicit examples, seed 13, đều pass. Kiểm tra này không thay thế pytest/Django integration hoặc bằng chứng P1/P2 trên Linux theo lockfile.

Ruff 0.16.10: `check --no-fix` và `format --check` đều pass cho `src/documents/tests/test_parsers.py`.

## Checklist nghiệm thu Cycle 1

- [ ] Reviewer xác nhận ít nhất ba invariant độc lập, đủ strategy, precondition, oracle và counterexample.
- [ ] Reviewer xác nhận giới hạn input/max_examples, shrinking và phương án tái lập phù hợp CI.
- [ ] Reviewer xác nhận không trùng phạm vi PBT-05 hoặc các suite khác.
- [ ] Reviewer kiểm tra phân biệt fallback extension, MIME đủ điều kiện và mapping của mọi parser đăng ký.
- [ ] PR chỉ chứa thiết kế/liên kết, xuất phát từ `dev`, xử lý issue #13.
- [ ] `@vuong123s` duyệt trước khi merge; chỉ đóng issue khi các tiêu chí được xác nhận.
