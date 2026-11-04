# Hồ sơ Gate 0 · Meter Assist

**Đọc 5 trường từ ảnh công tơ, FAQ/chatbot và ticket VGP** · bản v0.2

| | |
|---|---|
| Thực hiện | Đàm Quang Trung |
| Mentor | PhuNN4 |
| Pair | Đoàn Tuấn Long (mentor TungNX15) |
| Ngày | 04/11/2026 |
| Loại tài liệu | Hồ sơ trình Gate 0 |

> Mọi ngưỡng, mốc và phân công dưới đây là **đề xuất**, chưa phải kết quả đã đạt. Quy tắc đã thống nhất được ghi **[Đã chốt]**.

**Mục lục:** 1. Kế hoạch · 2. Rubric · 3. Pair contract · 4. Dữ liệu mockup · 5. Rủi ro và phụ thuộc

---

## Tình trạng tại 04/11

| Hạng mục | Trạng thái | Chi tiết |
|---|---|---|
| Code | Chạy được, test qua | Bản C# .NET 10: 55 check C# và 9 test giao diện qua; build Release 0 cảnh báo |
| Model đọc ảnh | Còn yếu | Crop chỉ số thật, tập test 4 ảnh: OCR gốc đúng 1/4; bản fine-tune đúng 0/4 nên không đưa vào dùng |
| Dữ liệu thật | 20 ảnh, 5 công tơ | Validation chỉ 1 công tơ, test 2 công tơ; tập test đã dùng để chẩn đoán, cần tập test mới |
| Training | Tạm dừng | Chờ Gate 0 chốt dữ liệu, kiểu công tơ và metric |

---

## 1. Kế hoạch

MVP gồm hai luồng chạy hoàn toàn cục bộ cho VGP/CPO:

- **Luồng chính:** đọc 5 trường từ ảnh công tơ điện tử, đối chiếu hai bộ đọc, người duyệt xác nhận rồi xuất JSON.
- **Luồng phụ:** FAQ/chatbot và tạo ticket hỗ trợ.

Mọi văn bản tự do (câu hỏi, ticket, văn bản OCR, log) đi qua bộ che PII của Đoàn Tuấn Long trước khi lưu hoặc gửi.

### 1.1 Vấn đề

- Chỉ số công tơ tại trạm sạc đang được đọc và nhập tay từ ảnh *(giả định, xác nhận tại Gate 0)*. Lỗi hay gặp: nhầm biểu khi màn hình LCD tự lật, mất số 0 đầu, sai dấu thập phân, số serial bị cắt hoặc lóa.
- Ảnh và chỉ số là dữ liệu nội bộ, nên không được dùng OCR cloud hay LLM qua API.
- Câu hỏi của người vận hành (chụp lại thế nào, vì sao sai biểu, chỉ số bất thường) chưa có kênh trả lời chuẩn. Yêu cầu hỗ trợ thường kèm tên, số điện thoại, địa chỉ trạm nên không chuyển nguyên văn được.

### 1.2 Người dùng và giá trị

| Người dùng | Việc họ làm | Giá trị nhận được |
|---|---|---|
| Nhân viên vận hành CPO | Chọn công tơ và thanh ghi, tải ảnh; hỏi FAQ | Năm trường đề xuất sẵn kèm vùng ảnh; câu trả lời có nguồn đã duyệt |
| Người duyệt | Đối chiếu ảnh, xác nhận đủ 5 trường hoặc yêu cầu chụp lại | Thấy kết quả OCR và AI đọc ảnh cạnh nhau, chỗ lệch và lý do |
| Bộ phận đối soát VGP | Nhận chỉ số đã xác nhận | JSON đúng schema, truy vết được tới ảnh, model và người duyệt |
| Business/CSKH VGP | Nhận ticket hỗ trợ | Ticket đã che PII, kèm dữ liệu công tơ có cấu trúc, không bị trùng |

### 1.3 Phạm vi

| Trong MVP | Ngoài MVP |
|---|---|
| Ảnh PNG/JPEG công tơ điện tử LCD; một ảnh ứng với một màn hình và biểu đang hiển thị | Hóa đơn, PDF, chứng từ khác; OCR hoặc LLM qua cloud |
| 5 trường `meterSerialNo`, `meterModel`, `bieu`, `displayReading`, `readingUnit`, mỗi trường có raw, value, confidence, page, bbox | Tự chèn dấu thập phân theo kiểu công tơ; tự bù chữ số serial bị mất |
| Hai bộ đọc local PP-OCRv6 tiny và PaddleOCR-VL-1.6; đối chiếu từng trường; sai OBIS thì yêu cầu chụp lại | Tự chấp nhận kết quả khi chưa hiệu chỉnh confidence trên dữ liệu độc lập |
| Người duyệt xác nhận đủ 5 trường; xuất JSON và file bằng chứng | Công tơ cơ (bánh xe): luôn chuyển người duyệt |
| Benchmark trên ảnh thật chia theo công tơ; fine-tune chỉ khi có baseline và validation đủ công tơ | Đối soát kWh với phiên sạc |
| FAQ từ danh sách đã duyệt; câu ngoài FAQ thành nháp ticket, người dùng xác nhận rồi mới lưu | LLM tự sinh câu trả lời ngoài FAQ; gửi ticket tới API thật khi chưa có hợp đồng API |
| Che PII bằng masker của Long; dữ liệu công tơ giữ nguyên qua allowlist | Train detector (cần gán nhãn toàn bộ vùng chữ); nhiều người dùng, production |

### 1.4 Quy tắc đã chốt

- **[Đã chốt] Dấu thập phân** lấy đúng như OCR hoặc AI đọc trên ảnh. Không chèn theo số chữ số hay kiểu công tơ, không xử lý dấu chấm thừa. Chuỗi không đúng dạng `số.số` (ví dụ `001241256` hoặc `000731.93.`) thì `value = null`, `confidence = 0` và chuyển người duyệt.
- **[Đã chốt] Chuẩn hóa:** giữ số 0 đầu. `bieu` nhận 1.8.0–1.8.3 và 2.8.0–2.8.3; Tổng/Total về 1.8.0; Biểu hoặc Rate 1/2/3 về 1.8.1/1.8.2/1.8.3. `readingUnit` nhận kWh, kVArh, hoặc OTHER khi thấy rõ đơn vị khác.
- **[Đã chốt] Confidence** là điểm của bộ đọc, chưa hiệu chỉnh, không gọi là độ chính xác. Khi chưa hiệu chỉnh, mọi hồ sơ đều qua người duyệt.

### 1.5 Success metric

Đo trên tập test ảnh thật đã che PII, chia theo công tơ, khóa trước khi chạy và chỉ chạy một lần. Báo tử số, mẫu số, phiên bản dữ liệu và fingerprint model. Trả null ở trường có đáp án tính là sai.

| Chỉ số | Cách đo | Mục tiêu CP3 (đề xuất) | Hiện tại 04/11 |
|---|---|---|---|
| Đúng từng trường | Value khớp đáp án; ≥ 100 ảnh từ ≥ 20 công tơ | reading, serial, bieu ≥ 95%; model, unit ≥ 90% | Chưa đo trên ảnh nguyên tấm; crop chỉ số: OCR gốc 1/4 |
| Đúng cả hồ sơ | Cả 5 value khớp | ≥ 85% | Chưa đo |
| Tự điền sai | Trường có đáp án null mà hệ thống điền giá trị; ≥ 30 cơ hội | ≤ 1% | Chưa đo |
| Tỷ lệ chuyển người duyệt | Sau khi hiệu chỉnh confidence trên validation | ≤ 40% | 100% (chưa hiệu chỉnh) |
| Tự chấp nhận | Cận dưới Wilson 95% của nhóm tự chấp nhận; 0 lỗi cần ≥ 381 ca | ≥ 99%; không đủ mẫu thì giữ tắt | Tắt |
| Thời gian mỗi ảnh | p95 đủ 5 trường, laptop CPU, gồm nạp model | ≤ 60 s | 37–40 s, ảnh giả lập |
| RAM đỉnh | Tiến trình suy luận | ≤ 4 GiB | ≈ 2,1 GB, ảnh giả lập |
| Không gửi dữ liệu ra ngoài | Suy luận với Docker `--network none` | Đạt | Đã kiểm chứng trên ảnh giả lập |
| FAQ trả đúng | ≥ 100 câu trong phạm vi, diễn đạt khác nhau | ≥ 90% đúng mã FAQ | Chưa có trong bản C# (bản Java cũ có 3 câu demo) |
| Câu ngoài phạm vi bị trả nhầm | ≥ 50 câu ngoài FAQ | ≤ 2% | Chưa đo |
| PII lộ | ≥ 200 ticket gài PII giả, quét bằng bộ của Long | 0 PII lộ; 0 giá trị công tơ bị che nhầm | Chưa đo |
| Ticket gửi đúng một lần | Gửi lặp, mất mạng giả lập | 100% không trùng | Prototype cũ: gửi 2 lần, nhận 1 |
| Thời gian thao tác của người dùng | So với nhập tay; ≥ 2 người, ≥ 20 tác vụ | Giảm trung vị ≥ 20% | Chưa đo |

### 1.6 Kiến trúc sơ bộ

Ứng dụng ASP.NET Core (.NET 10) chỉ nghe trên `127.0.0.1`. Toàn bộ suy luận viết bằng C#: ONNX Runtime cho PP-OCRv6, LLamaSharp cho PaddleOCR-VL-1.6. Lúc chạy không cần Python, không gọi dịch vụ ngoài; model được kiểm tra checksum trước khi nạp. Huấn luyện là bộ công cụ Python riêng, chạy offline trong Docker `--network none`.

| Luồng | Các bước |
|---|---|
| Đọc ảnh | Ảnh PNG/JPEG (≤ 8 MiB, xoay theo EXIF) → **PP-OCRv6 tiny** tìm vùng chữ, đọc chữ → tìm 5 trường theo nhãn Số sx, Kiểu, OBIS → **PaddleOCR-VL-1.6** đọc crop nới 20% (không nhận chữ OCR) → đối chiếu từng trường, sai OBIS thì chụp lại → người duyệt xác nhận đủ 5 trường → JSON `meter-fields.v1` |
| FAQ và ticket | Câu hỏi → **masker PII của Long** (keep giữ giá trị công tơ) → tra FAQ đã duyệt → trả lời có nguồn, hoặc nháp ticket → người dùng xác nhận → hộp thư local, gửi idempotent |
| Huấn luyện (tách khỏi ứng dụng) | Label Studio local → manifest chia theo công tơ, test khóa → train trong Docker không mạng → chọn trên validation → xuất ONNX/GGUF có checksum → bật qua `runtime.json` |

### 1.7 Mốc chính

T0 là ngày Gate 0 được duyệt. Dữ liệu masked của mentor dự kiến có từ 19/11.

| Mốc | Việc chính | Bằng chứng |
|---|---|---|
| Gate 0 (04/11) | Chốt phạm vi, 5 trường, metric, rubric, quyền dữ liệu, contract với Long | Hồ sơ này được mentor duyệt |
| Gate 1 (T0 + 3 ngày làm việc) | Sửa code trước khi train: bỏ giới hạn ME-41/ME-42, danh mục công tơ thật (mã đã che), tắt adapter VLM cũ. Đưa FAQ/ticket sang bản C#. Sinh bộ mockup | Test qua; fixture hợp đồng dùng chung với Long |
| CP1 (T0 + 10 ngày) | Baseline 5 trường trên ảnh nguyên tấm của validation, tách lỗi tìm vùng và lỗi đọc chữ. Masker một chiều trong luồng FAQ/ticket | Báo cáo baseline có fingerprint, hash dữ liệu |
| CP2 (T0 + 20 ngày) | Nhận dữ liệu masked, lập tập test mới. Fine-tune có kiểm soát nếu baseline chưa đạt (learning rate thấp, kiểm tra leave-one-meter-out trên train/validation). Hiệu chỉnh confidence. Tích hợp hai chiều với Long | So sánh với baseline trên cùng validation |
| CP3 (T0 + 30 ngày) | Chạy test một lần, UAT với người dùng nghiệp vụ, bàn giao tái lập | Báo cáo test, biên bản UAT, README chạy lại được |

---

## 2. Rubric đã cụ thể hóa

Mỗi mức được xác định bằng số đo trên tập test ảnh thật đã che PII, cùng harness và cùng fingerprint model. Mức cao gồm điều kiện của mức thấp hơn. Gate 0 và Gate 1 chỉ đạt hoặc chưa đạt, được ghi vào tiêu chí 3.

### Tiêu chí 1 · Giá trị và kết quả

| Mức | Điều kiện đo được |
|---|---|
| **2 · Đạt** | Luồng ảnh → hai bộ đọc → 5 trường → duyệt → JSON chạy trên ≥ 50 ảnh thật từ ≥ 10 công tơ. reading và bieu ≥ 85%, ba trường còn lại ≥ 80%. Không có ca tự chấp nhận sai. FAQ ≥ 80% đúng mã; luồng nháp → xác nhận → hộp thư chạy qua masker của Long. |
| **3 · Tốt** | Đạt mọi metric CP3 ở mục 1.5: ≥ 100 ảnh từ ≥ 20 công tơ; reading, serial, bieu ≥ 95%; model, unit ≥ 90%; cả hồ sơ ≥ 85%; p95 ≤ 60 s. FAQ ≥ 90%, trả nhầm ≤ 2%; 0 PII lộ trên 200 ticket. Ít nhất 1 người vận hành dùng thử cả hai luồng (≥ 20 ảnh, ≥ 20 câu hỏi), phản hồi được ghi lại. |
| **4 · Xuất sắc** | reading và bieu ≥ 97%, serial ≥ 98%, model và unit ≥ 95%, cả hồ sơ ≥ 90%. Thời gian thao tác giảm ≥ 30% với 2 người dùng. Người dùng nghiệp vụ đề xuất chạy thử thật, có biên bản hoặc backlog được duyệt. |

### Tiêu chí 2 · Chất lượng kỹ thuật

| Mức | Điều kiện kiểm chứng được |
|---|---|
| **2 · Đạt** | README chạy được từ đầu; toàn bộ check C# và test giao diện qua (hiện 55 + 9), build 0 cảnh báo; smoke offline với `--network none` thoát 0. JSON 100% đúng schema. |
| **3 · Tốt** | Benchmark ghi fingerprint model, code, runtime, hash dữ liệu, seed. Chọn model và ngưỡng chỉ trên validation; test chia theo công tơ, khóa, chạy một lần. Báo lỗi theo trường và theo bước (tìm vùng hay đọc chữ). Có test cho "masker lỗi thì không lưu/gửi" và "gửi lặp không tạo ticket trùng". Không có ảnh thật hoặc PII trong repo, log, ZIP bàn giao. |
| **4 · Xuất sắc** | Người khác dựng lại trên máy sạch và ra cùng số liệu trong ≤ 2 giờ chỉ theo README. Thêm kiểu công tơ mới chỉ bằng danh mục và nhãn, thêm câu FAQ chỉ bằng file FAQ, không sửa code. Runbook train, hiệu chỉnh, rollback đã chạy thử một lần. |

### Tiêu chí 3 · Thực thi và chủ động

| Mức | Bằng chứng |
|---|---|
| **2 · Đạt** | ≥ 80% đầu việc của mốc xong đúng checkpoint; Gate 0 và Gate 1 đạt; cập nhật tiến độ ít nhất 1 lần/tuần. |
| **3 · Tốt** | 100% mốc đúng hạn hoặc được mentor duyệt đổi trước hạn. Rủi ro mới ghi vào bảng rủi ro trong ≤ 1 ngày làm việc; góp ý của mentor xử lý trong ≤ 2 ngày làm việc. |
| **4 · Xuất sắc** | Điều chỉnh kế hoạch dựa trên số liệu ít nhất 2 lần, có ghi lý do (ví dụ: dừng dùng bản fine-tune khi test giảm từ 1/4 xuống 0/4). Pair contract được cả hai bên dùng và tích hợp hai chiều chạy được. |

---

## 3. Pair contract · Cặp A (Long – Trung)

| Bên | Phụ trách | Mentor |
|---|---|---|
| Đàm Quang Trung | OCR công tơ 5 trường, FAQ/chatbot, ticket | PhuNN4 |
| Đoàn Tuấn Long | Phát hiện và che PII trong văn bản, ticket, log | TungNX15 |

> **Phiên bản:** v0.1 là bản trong Google Doc "Contract", tab 2; mục 3.6 liệt kê các thay đổi so với v0.1. Chỉ dùng mockup đến CP1; dữ liệu masked của mentor từ 19/11.

### 3.1 Hai bài toán giao nhau ở đâu, giá trị cộng hưởng là gì

- **Nơi giao nhau:** câu hỏi FAQ, ticket và văn bản OCR (các dòng chữ trên ảnh ngoài chỉ số và serial, ví dụ nhãn dán) là văn bản tự do, hay chứa tên, SĐT, email, địa chỉ trạm, biển số xe, số CCCD. Đây là đầu vào thật cho bộ phát hiện PII của Long.
- **Bài toán khó chung:** ticket phải giữ nguyên dữ liệu nghiệp vụ (serial, thanh ghi, chỉ số) trong khi che PII. Serial 10 chữ số dễ bị nhận nhầm là SĐT hoặc số định danh, nên cần che có allowlist.
- **Long nhận được:** pipeline thật và bộ đo chất lượng che trên ticket, log có nhãn.
- **Trung nhận được:** ticket, log và bằng chứng OCR không chứa PII, đủ điều kiện cho người dùng thử.

### 3.2 Trách nhiệm và ranh giới

| Phần việc | Trung | Long |
|---|---|---|
| FAQ/chatbot, nháp ticket, xác nhận, hộp thư, gửi idempotent | Chịu trách nhiệm | — |
| Gọi masker tại 3 điểm: nhận câu hỏi, trước khi lưu ticket, sau OCR; chặn lưu/gửi khi masker lỗi | Chịu trách nhiệm | Cung cấp thư viện `piimask` hoặc HTTP localhost |
| Phát hiện và che PII trong văn bản tiếng Việt; danh sách loại PII; phiên bản masker | Đóng góp mẫu lỗi, đề xuất loại PII mới | Chịu trách nhiệm |
| Allowlist nghiệp vụ: serial, thanh ghi, chỉ số | Định nghĩa; truyền giá trị cụ thể trong `keep` | Không che đúng các giá trị trong `keep` |
| OCR ảnh → 5 trường và các dòng chữ kèm bbox, chạy local | Chịu trách nhiệm | — |
| Che vùng ảnh có PII theo bbox của dòng mà masker báo có entity | Chịu trách nhiệm | — (Long không xử lý ảnh) |
| Ticket và log mockup dạng object kèm nhãn PII | Cung cấp | Đọc, quét, duyệt, che, xuất file |
| Bộ kiểm thử chung: câu hỏi, ticket, log có PII giả | Sinh dữ liệu | Gán nhãn PII, đo recall và precision |

**Ranh giới**

- Trung không tự viết quy tắc nhận diện PII.
- Long không sửa logic FAQ, ticket, model OCR hay quy tắc đọc chỉ số.
- Masker lỗi hoặc quá thời gian thì **không lưu, không gửi** (fail closed).

### 3.3 Trao đổi gì, định dạng nào

| Quy ước | Giá trị |
|---|---|
| Định dạng | JSON UTF-8, có `schemaVersion` (hiện là 1) |
| Cách gọi | `import piimask` hoặc HTTP `127.0.0.1`; không dữ liệu nào rời máy |
| Văn bản | Chuẩn hóa Unicode NFC trước khi gửi |
| Vị trí ký tự | `start`, `end` là chỉ số code point trong văn bản gửi đi; `end` không tính (giống cắt chuỗi Python) |
| Timeout | 1 s mỗi lần gọi |
| Khi lỗi | `status` khác `OK` thì Trung chặn lưu và gửi |

**Trung → Long: yêu cầu che văn bản.** `context` nhận `faq`, `ocr`, `ticket`, `log`. `keep` ghi giá trị cụ thể lấy từ danh mục hoặc kết quả OCR, không chỉ tên loại.

```json
{
  "schemaVersion": 1,
  "requestId": "req-0001",
  "context": "ticket",
  "locale": "vi",
  "text": "Trạm của anh Nguyễn Văn A, gọi 0912345678, công tơ 2915161251 B2 đọc 00420.1 bất thường",
  "keep": [
    {"type": "METER_SERIAL", "value": "2915161251"},
    {"type": "REGISTER", "value": "B2"},
    {"type": "METER_READING", "value": "00420.1"}
  ]
}
```

**Long → Trung: kết quả che.** Vị trí tính trên văn bản gửi đi: tên ở 13–25, số điện thoại ở 31–41.

```json
{
  "schemaVersion": 1,
  "requestId": "req-0001",
  "maskerVersion": "…",
  "status": "OK",
  "maskedText": "Trạm của anh [NAME], gọi [PHONE], công tơ 2915161251 B2 đọc 00420.1 bất thường",
  "entities": [
    {"type": "NAME", "start": 13, "end": 25},
    {"type": "PHONE", "start": 31, "end": 41}
  ]
}
```

**Kết quả OCR của Trung** (cùng hồ sơ với ticket ở trên; số liệu minh họa, không phải ảnh thật):

```json
[
  {"meterSerialNo":  {"raw": "2915161251", "value": "2915161251", "confidence": 0.97, "page": 1, "bbox": [0.10, 0.80, 0.45, 0.86]}},
  {"meterModel":     {"raw": "ME-42",      "value": "ME-42",      "confidence": 0.95, "page": 1, "bbox": [0.10, 0.72, 0.30, 0.78]}},
  {"bieu":           {"raw": "1.8.2",      "value": "1.8.2",      "confidence": 0.93, "page": 1, "bbox": [0.20, 0.30, 0.31, 0.36]}},
  {"displayReading": {"raw": "00420.1",    "value": "00420.1",    "confidence": 0.90, "page": 1, "bbox": [0.32, 0.41, 0.71, 0.52]}},
  {"readingUnit":    {"raw": "kWh",        "value": "kWh",        "confidence": 0.92, "page": 1, "bbox": [0.72, 0.44, 0.80, 0.50]}}
]
```

- `raw` chép nguyên văn; `value` đã chuẩn hóa; `confidence` trong 0..1; `page` ≥ 1; `bbox` là `[x1, y1, x2, y2]` chuẩn hóa 0..1 theo ảnh đã xoay EXIF, hoặc `null`.
- Giá trị `value` của serial, biểu và chỉ số được đưa vào `keep` khi gọi masker, nên không bị che.
- **[Đã chốt]** Dấu thập phân lấy đúng như trên ảnh, không chèn; sai dạng thì `value = null` và chuyển người duyệt.

**Ticket lưu và gửi.** Câu hỏi đã che; `meterData` lấy từ kết quả OCR đã được người dùng xác nhận và không đi qua masker.

```json
{
  "id": "tk-0001",
  "question": "Trạm của anh [NAME], gọi [PHONE], công tơ 2915161251 B2 đọc 00420.1 bất thường",
  "created": "2026-11-05T08:00:00Z",
  "maskerVersion": "…",
  "meterData": {"meterSerialNo": "2915161251", "register": "B2", "reading": "00420.1"}
}
```

**Chiều ngược lại: log mockup cho ứng dụng của Long.** Mỗi bản ghi có nhãn PII theo cùng quy ước vị trí.

```json
{"id": "log-0001", "text": "Người duyệt Trần Thị B xác nhận công tơ 2915161251", "labels": [{"type": "NAME", "start": 12, "end": 22}]}
```

### 3.4 Khi nào tích hợp, đo giá trị cộng hưởng bằng gì

| Mốc | Tích hợp | Số đo và mục tiêu (đề xuất) | Điểm pair |
|---|---|---|---|
| Gate 0 | Hai bên xác nhận contract | Contract, fixture và issue được cả hai bên dùng | Mức 1 |
| Sprint 1, trước CP1 | **Một chiều:** Trung gọi `piimask` trên câu hỏi, ticket và văn bản OCR | 0 giá trị `keep` bị che; recall PII ≥ 95% trên ticket mockup có nhãn; p95 ≤ 200 ms mỗi lần gọi; demo được | Mức 2 |
| Sprint 2, trước CP3 | **Hai chiều:** ticket và log mockup của Trung được ứng dụng của Long đọc, quét, che, xuất file | Recall ≥ 95%, precision ≥ 90% trên ticket và log có nhãn; báo % câu hỏi được che và độ trễ | Mức 3 |

### 3.5 Phiên bản, lỗi và nhịp phối hợp

- **Phiên bản:** đổi schema theo kiểu phá vỡ (breaking) thì tăng `schemaVersion` và hai bên cùng sửa fixture; giữ được bản cũ ít nhất một sprint.
- **Gọi lại:** `requestId` duy nhất; gọi lại cùng `requestId` trả cùng kết quả, không tạo bản ghi trùng.
- **Mã lỗi:** `status` là `OK` hoặc `ERROR` kèm `error` (ví dụ `TIMEOUT`, `INVALID_INPUT`, `MODEL_UNAVAILABLE`). Trung không tự sửa văn bản khi nhận lỗi.
- **Sự cố dữ liệu:** phát hiện PII lộ hoặc giá trị `keep` bị che thì dừng xuất ngay, báo cả hai mentor trong 1 ngày làm việc.
- **Nhịp:** 2 buổi ngắn mỗi tuần; issue và fixture để ở một chỗ chung hai bên cùng xem.

### 3.6 Thay đổi so với v0.1

| Thay đổi | Lý do |
|---|---|
| Ví dụ JSON viết lại thành JSON hợp lệ | Bản v0.1 xuống dòng giữa chuỗi, không parse được |
| Ví dụ OCR dùng cùng hồ sơ với ticket và đủ 5 trường | Bản v0.1 dùng chỉ số khác (`0012412.56`) và chỉ có 2 trường |
| `meterData` đổi tên theo trường OCR: `{meterSerialNo, bieu, displayReading}`; `REGISTER` trong `keep` nhận cả `B2` và `1.8.2` | Bản v0.1 dùng `register`, `reading`, khác tên 5 trường OCR |
| Thêm mục 3.5 (phiên bản, gọi lại, mã lỗi, sự cố) | Bản v0.1 chưa quy định cách xử lý khi đổi schema hoặc khi lỗi |
| Mốc Sprint 2 dùng cùng ngưỡng độ trễ p95 ≤ 200 ms | Bản v0.1 chỉ ghi "độ trễ", chưa có ngưỡng |

---

## 4. Đặc tả bộ dữ liệu mockup

Mockup dùng để thử hợp đồng 5 trường, quy tắc, giao diện, trường hợp phải trả null, FAQ và masker cùng Long.

> Mockup **không bao giờ vào validation hoặc test** và không dùng để báo độ chính xác hay chọn ngưỡng tự chấp nhận. Code hiện tại đã từ chối dòng `source=synthetic` ở validation/test.

Hiện có 4 ảnh giả lập của công tơ DEMO-001 và 2 ảnh mặt công tơ đủ 5 trường; phần dưới là đề xuất mở rộng.

### 4.1 Thành phần

| Tập | Số lượng | Cách tạo | Dùng cho |
|---|---|---|---|
| Danh mục giả lập | 8 công tơ: 6 LCD thuộc 4 kiểu, 2 cơ | `catalog.json`, `synthetic=true`; thanh ghi TOTAL, B1–B3 và 2.8.x có mã OBIS | Quy tắc biểu; kiểu công tơ không bị giới hạn cứng |
| Ảnh mặt công tơ đủ 5 trường | 200 ảnh từ 40 công tơ giả, 5 ảnh mỗi công tơ | Phần mềm dựng mặt công tơ, font giấy phép mở (DSEG7, OFL) | Hợp đồng 5 trường, giao diện, smoke |
| Crop chỉ số LCD | 1.200 crop | Cùng bộ dựng, nhiều chiều dài số và vị trí dấu thập phân | Smoke và hỗ trợ train rec, không đo độ chính xác |
| Ảnh có PII giả | 50 ảnh | Nhãn dán tên, SĐT, biển số giả; EXIF GPS giả | Che vùng ảnh theo bbox |
| FAQ mẫu | 20–30 mục | Soạn từ quy trình chụp và đọc chỉ số; business duyệt trước khi dùng thật | Nguồn trả lời |
| Câu hỏi test FAQ | 150 câu: 100 trong, 50 ngoài phạm vi | Diễn đạt khác, không dấu, viết tắt, lỗi chính tả | Đo đúng mã FAQ và trả nhầm |
| Ticket và log gài PII giả | 200 ticket, 100 dòng log | Tên, SĐT, email, CCCD, địa chỉ, biển số giả trộn với serial và chỉ số | Đo masker: PII lộ, che nhầm |

### 4.2 Phân bố 200 ảnh mặt công tơ

| Nhóm | Số ảnh | Tình huống cần có |
|---|---:|---|
| LCD rõ | 40 | 4 kiểu cân bằng; nhãn Số sx, No., S/N; nhãn Kiểu, Type, Model |
| Biểu và OBIS | 40 | Đủ 8 mã 1.8.x, 2.8.x; Tổng/Total; Biểu và Rate 1–3; một số ca không có mã |
| Dấu thập phân | 30 | Có dấu ở các vị trí khác nhau; thiếu dấu; dấu chấm thừa (phải ra `value = null`) |
| Lóa, mờ, nghiêng, tối | 40 | Mỗi lỗi ảnh tác động riêng từng trường; mất một chữ số serial phải ra null |
| Cắt hoặc che khuất | 20 | Cắt đuôi serial, thiếu vùng chỉ số; không tự bù số |
| Nhãn dễ nhầm và đơn vị | 20 | Tên hãng cạnh kiểu công tơ; hằng số xung gần chỉ số; kWh, kVArh, đơn vị khác, không có đơn vị |
| Không hỗ trợ | 10 | Công tơ cơ, màn hình lạ; phải chuyển người duyệt |
| **Tổng** | **200** | |

### 4.3 Mỗi dòng manifest (JSONL)

| Trường | Quy ước |
|---|---|
| `id`, `image`, `imageSha256` | id duy nhất; đường dẫn tương đối từ thư mục manifest; hash từng ảnh |
| `meterId`, `meterGroup`, `registerId` | Có trong danh mục; mọi ảnh cùng công tơ thuộc cùng một tập |
| `truth` | Chỉ số đúng như trên ảnh, giữ số 0 đầu và dấu thập phân, ví dụ `00512.4` |
| `fields` | Đáp án 5 trường, mỗi trường raw/value hoặc `nullReason` khi không đọc được |
| `source`, `labelSource`, `split` | `synthetic`, `RENDERED_TRUTH`; chỉ `train` hoặc `smoke`; seed 20261104 |
| `cropBox`, `conditions`, `piiPlanted` | Pixel theo ảnh đã xoay; glare, blur, tilt, lowlight, cut; loại và vị trí PII giả |

```json
{"id": "syn-0001", "image": "images/syn-0001.png", "imageSha256": "…", "meterId": "MOCK-03", "meterGroup": "MOCK-03", "registerId": "B1", "truth": "00210.6", "source": "synthetic", "labelSource": "RENDERED_TRUTH", "split": "smoke", "cropBox": [412, 300, 690, 372], "conditions": ["glare"], "piiPlanted": []}
```

### 4.4 Quy tắc

- Mockup chỉ ở train hoặc smoke. Ảnh tổng hợp của công tơ thuộc tập giữ lại bị loại khỏi train.
- Ảnh thật chỉ vào dự án khi đã qua che PII và được owner dữ liệu cho phép; lưu trong `private/`, không đưa vào repo, ZIP hay cuộc trò chuyện.
- Mỗi phiên bản dữ liệu có hash manifest và hash từng ảnh; báo cáo benchmark ghi lại hai hash này.
- Câu hỏi test FAQ không được dùng làm bí danh trong file FAQ.

---

## 5. Rủi ro và phụ thuộc

> Rủi ro lớn nhất là thiếu dữ liệu thật. Thử nghiệm ngày 04/11 với 20 ảnh từ 5 công tơ cho thấy fine-tune làm model tốt hơn trên validation nhưng kém đi trên test.

### 5.1 Rủi ro

| Rủi ro | Khả năng | Ảnh hưởng | Giảm thiểu | Dấu hiệu báo sớm |
|---|---|---|---|---|
| Dữ liệu thật quá ít, ít công tơ | Đã xảy ra | Không đánh giá được; fine-tune học thuộc (validation 4/4, test 0/4) | Dữ liệu masked từ 19/11; không train lại tới khi validation có ≥ 4 công tơ | Tới CP1 dưới 50 ảnh |
| Tập test hiện tại đã bị dùng | Đã xảy ra | Không còn ước lượng khách quan | Lập tập test mới từ công tơ chưa từng thấy, khóa trước khi train | Chưa có tập test mới ở CP2 |
| Kiểu công tơ thực tế ngoài ME-41/ME-42 | Đã xảy ra | Code hiện chỉ nhận 2 kiểu, không duyệt được hồ sơ kiểu khác | Sửa trước khi train; danh sách kiểu lấy từ danh mục | Hồ sơ bị từ chối khi duyệt |
| Adapter VLM cũ có thể đã thấy ảnh validation/test mới | Trung bình | Số đo bị lạc quan | Tắt adapter cũ; chỉ train mới từ trọng số gốc | Adapter còn bật trong `runtime.json` |
| Confidence chưa hiệu chỉnh | Cao | 100% hồ sơ chuyển người duyệt | Hiệu chỉnh trên validation khi đủ công tơ; giữ tự chấp nhận tắt | Chưa đủ mẫu ở CP2 |
| CPU chậm khi OCR hụt trường | Trung bình | VLM có thể phải đọc thêm tới 20 vùng mỗi ảnh, vượt 60 s | Đo p95 trên ảnh thật; giới hạn số vùng dự phòng | p95 > 60 s trên 20 ảnh |
| Ảnh nghiêng, chữ dính nhau | Trung bình | Vùng chữ lấy theo trục ảnh nên cắt sai | Báo lỗi riêng theo bước tìm vùng; yêu cầu chụp lại | Lỗi tìm vùng chiếm phần lớn |
| Màn hình tự lật, nhầm biểu | Trung bình | Đúng số nhưng sai thanh ghi | Kiểm tra OBIS trên ảnh; sai thì chụp lại | Tỷ lệ chụp lại > 15% |
| Masker bỏ sót PII tiếng Việt hoặc che nhầm serial | Cao | Lộ dữ liệu hoặc ticket mất thông tin | `keep` theo giá trị; `meterData` tách khỏi văn bản; người dùng xem bản đã che | Có PII lộ trên bộ 200 ticket |
| FAQ/ticket chưa có trong bản C# | Cao | Trễ CP1 | Chuyển từ prototype Java cũ ở Gate 1; ưu tiên luồng che một chiều | Gate 1 chưa xong |
| Nội dung FAQ chưa được business duyệt | Trung bình | Không cho người dùng thử được | Soạn nháp từ quy trình, xin duyệt sớm | Chưa có người duyệt sau Gate 0 |
| Giới hạn máy: 16 GB RAM, không GPU | Cao | Train VLM chậm, dễ hết bộ nhớ | LoRA nhỏ, batch 1; train trên máy nội bộ khi được duyệt | Container train bị dừng vì RAM |

### 5.2 Phụ thuộc

| Phụ thuộc | Bên cung cấp | Cần trước |
|---|---|---|
| Danh sách kiểu công tơ, thanh ghi mục tiêu, danh mục với mã đã che | Mentor PhuNN4, vận hành VGP/CPO | Gate 1 |
| Quyền dữ liệu: nơi lưu, thời gian lưu, được train ở đâu | Mentor, an toàn thông tin | Gate 0 |
| Dữ liệu masked và người xác nhận đáp án | Mentor, vận hành VGP/CPO | Từ 19/11 |
| Masker PII, schema, danh sách loại PII | Đoàn Tuấn Long | Sprint 1 |
| Nội dung FAQ và người duyệt | Business/CSKH VGP | Đo FAQ, dùng thử |
| Bên nhận ticket: địa chỉ nội bộ, quyền gửi, hợp đồng API | Business/IT VGP | Gửi thật, CP2 |
| Người dùng thử | Mentor giới thiệu | CP1 |
| Máy nội bộ để train | Quản trị IT | Fine-tune |

---

*Thay đổi so với bản Gate 0 trước:* kiến trúc chuyển sang C# (.NET 10, ONNX Runtime, LLamaSharp), không còn worker Python lúc chạy; mở rộng từ chỉ số sang 5 trường; chốt quy tắc dấu thập phân; thêm kết quả thử train 04/11 và rủi ro mới; pair contract kèm các thay đổi so với v0.1. Số liệu hiện tại lấy từ báo cáo kiểm thử và báo cáo train trên máy phát triển ngày 04/11/2026.
