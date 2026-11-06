# Hồ sơ Gate 0 · Meter Assist

**Đọc 5 trường từ ảnh công tơ, FAQ/chatbot và ticket VGP** · bản v0.2

| | |
|---|---|
| Thực hiện | Đàm Quang Trung |
| Mentor | PhuNN4 |
| Pair | Đoàn Tuấn Long (mentor TungNX15) |
| Ngày | 04/11/2026 |
| Loại tài liệu | Hồ sơ trình Gate 0 |

> Mọi ngưỡng, mốc và phân công dưới đây là **đề xuất**, chưa phải kết quả đã đạt. Quy tắc đã thống nhất được ghi **[Đã chốt]**; quy tắc đã đổi và đang chờ mentor xác nhận được ghi **[Đổi, chờ xác nhận]**.

**Mục lục:** 1. Kế hoạch · 2. Rubric · 3. Pair contract · 4. Dữ liệu mockup · 5. Rủi ro và phụ thuộc

---

## Tình trạng hiện tại

| Hạng mục | Trạng thái | Chi tiết |
|---|---|---|
| Code | Chạy được, test qua | Bản C# .NET 10, **chỉ OCR** (đã bỏ VLM). 84 check C# và 27 check HTTP qua; build Release 0 cảnh báo; có CI. Code chia theo module, có tài liệu kiến trúc, luật từng trường, cách đánh giá |
| Model đọc ảnh | Đọc được, tìm vùng còn yếu | Model đọc chỉ số fine-tune **T3** đang dùng: 38/50 crop chỉ số thật đúng nguyên chuỗi. Trên ảnh thật nguyên tấm, lỗi chính là **tìm vùng** của từng trường, không phải đọc chữ |
| Dữ liệu thật | 57 ảnh, khoảng 17 công tơ | Export Label Studio VGreen (58 file, 1 cặp trùng); 50 crop chỉ số đã xác minh; chia 5 fold theo công tơ. Chưa có tập test chưa từng dùng |
| Training | Đang đo | Kiểm chéo theo công tơ: model train thêm ảnh thật đúng 36/40 so với T3 29/40 (fold 0–3). Fold 4 và model cuối đang chạy; chỉ thay T3 khi đạt luật đã viết trước |
| FAQ/ticket | Chưa có trong bản C# | Chỉ có prototype Java cũ |

---

## 1. Kế hoạch

MVP gồm hai luồng chạy hoàn toàn cục bộ cho VGP/CPO:

- **Luồng chính:** đọc 5 trường từ ảnh công tơ điện tử bằng OCR local, người duyệt xác nhận rồi xuất JSON.
- **Luồng phụ:** FAQ/chatbot và tạo ticket hỗ trợ.

Mọi văn bản tự do (câu hỏi, ticket, văn bản OCR, log) đi qua bộ che PII của Đoàn Tuấn Long trước khi lưu hoặc gửi.

### 1.1 Vấn đề

- Chỉ số công tơ tại trạm sạc đang được đọc và nhập tay từ ảnh *(giả định, xác nhận tại Gate 0)*. Lỗi hay gặp: nhầm biểu khi màn hình LCD tự lật, mất số 0 đầu, sai dấu thập phân, số serial bị cắt hoặc lóa.
- Ảnh và chỉ số là dữ liệu nội bộ, nên không được dùng OCR cloud hay LLM qua API.
- Câu hỏi của người vận hành (chụp lại thế nào, vì sao sai biểu, chỉ số bất thường) chưa có kênh trả lời chuẩn. Yêu cầu hỗ trợ thường kèm tên, số điện thoại, địa chỉ trạm nên không chuyển nguyên văn được.

### 1.2 Người dùng và giá trị

| Người dùng | Việc họ làm | Giá trị nhận được |
|---|---|---|
| Nhân viên vận hành CPO | Tải ảnh (chọn công tơ và thanh ghi nếu có); hỏi FAQ | Năm trường đề xuất sẵn kèm vùng trên ảnh; gợi ý chụp lại khi ảnh tối, lóa, mờ; câu trả lời có nguồn đã duyệt |
| Người duyệt | Đối chiếu ảnh, xác nhận đủ 5 trường hoặc yêu cầu chụp lại | Thấy từng trường, vị trí trên ảnh và lý do một trường bị để trống (lóa, các lần đọc không khớp, sai định dạng danh mục…) |
| Bộ phận đối soát VGP | Nhận chỉ số đã xác nhận | JSON đúng schema, truy vết được tới ảnh, model và người duyệt |
| Business/CSKH VGP | Nhận ticket hỗ trợ | Ticket đã che PII, kèm dữ liệu công tơ có cấu trúc, không bị trùng |

### 1.3 Phạm vi

| Trong MVP | Ngoài MVP |
|---|---|
| Ảnh PNG/JPEG công tơ điện tử LCD; một ảnh ứng với một màn hình và biểu đang hiển thị | Hóa đơn, PDF, chứng từ khác; OCR hoặc LLM qua cloud |
| 5 trường `meterSerialNo`, `meterModel`, `bieu`, `displayReading`, `readingUnit`, mỗi trường có raw, value, confidence, page, bbox | Tự chèn dấu thập phân theo kiểu công tơ; tự bù chữ số serial bị mất; đọc mã vạch |
| OCR local PP-OCRv6 tiny và một model đọc chỉ số fine-tune cho LCD; đọc chỉ số 4 lần rồi bỏ phiếu; sai OBIS thì yêu cầu chụp lại | Bộ đọc ảnh thứ hai bằng VLM (PaddleOCR-VL): **đã bỏ** vì đúng 1/10 crop thật và khoảng 35 s/ảnh |
| Người duyệt xác nhận đủ 5 trường; xuất JSON và file bằng chứng | Tự chấp nhận kết quả khi chưa hiệu chỉnh confidence trên dữ liệu độc lập; công tơ cơ (bánh xe): chỉ số luôn để trống |
| Benchmark 5 trường trên ảnh nguyên tấm (tách lỗi tìm vùng và lỗi đọc chữ); fine-tune có kế hoạch và luật chọn viết trước | Đối soát kWh với phiên sạc |
| FAQ từ danh sách đã duyệt; câu ngoài FAQ thành nháp ticket, người dùng xác nhận rồi mới lưu | LLM tự sinh câu trả lời ngoài FAQ; gửi ticket tới API thật khi chưa có hợp đồng API |
| Che PII bằng masker của Long; dữ liệu công tơ giữ nguyên qua allowlist | Train detector (cần gán nhãn toàn bộ vùng chữ); nhiều người dùng, production |

### 1.4 Quy tắc

- **[Đổi, chờ xác nhận] Dấu thập phân** lấy đúng như OCR đọc trên ảnh, không bao giờ chèn hay dời theo số chữ số hay kiểu công tơ. Màn hình **không có dấu** (nhiều công tơ hiển thị số nguyên) được điền nguyên dãy số như nhìn thấy; nếu danh mục của thanh ghi có phần lẻ thì chỉ **gắn cờ** `CATALOG_FORMAT_MISMATCH` và giảm một nửa confidence, không sửa giá trị. Dấu chấm thừa (ví dụ `000731.93.`) và màn hình giờ vẫn `value = null` và chuyển người duyệt. *(Trước đây để trống mọi chuỗi không có dạng `số.số`.)*
- **[Đã chốt] Chuẩn hóa:** giữ số 0 đầu. `bieu` nhận 1.8.0–1.8.3 và 2.8.0–2.8.3; Tổng/Total về 1.8.0; Biểu hoặc Rate 1/2/3 về 1.8.1/1.8.2/1.8.3. `readingUnit` nhận kWh, kVArh, hoặc OTHER khi thấy rõ đơn vị khác.
- **Số công tơ không giới hạn độ dài** (không phải 8 hay 10 chữ số; người dùng đã xác nhận). Serial bị lóa, mờ hoặc chạm mép ảnh thì để trống.
- **[Đã chốt] Confidence** là điểm của bộ đọc, chưa hiệu chỉnh, không gọi là độ chính xác. Khi chưa hiệu chỉnh, mọi hồ sơ đều qua người duyệt. Đã đo: kể cả lần đọc có điểm ≥ 0,99 vẫn sai 5–8% trên crop thật.
- **Đổi model hay luật đọc** chỉ khi đạt luật quyết định viết trước khi đo: không trường nào thêm ca điền sai, số chỉ đọc sai không tăng, tổng số trường đúng tăng. Không tự đổi cấu hình đang chạy.

### 1.5 Success metric

Đo trên tập test ảnh thật đã che PII, chia theo công tơ, khóa trước khi chạy và chỉ chạy một lần. Báo tử số, mẫu số, phiên bản dữ liệu và fingerprint model. Trả null ở trường có đáp án tính là sai. Cột "Hiện tại" chưa phải số đo trên tập test như vậy: mockup do phần mềm dựng, ảnh thật mới có 10 ảnh nguyên tấm của 3 công tơ và đã được dùng nhiều lần.

| Chỉ số | Cách đo | Mục tiêu CP3 (đề xuất) | Hiện tại |
|---|---|---|---|
| Đúng từng trường | Value khớp đáp án; ≥ 100 ảnh từ ≥ 20 công tơ | reading, serial, bieu ≥ 95%; model, unit ≥ 90% | Mockup 200 ảnh: reading 164/180, serial 139/180, bieu 130/180, model 166/190, unit 182/196. Ảnh thật 10 ảnh: model 9/10, unit 8/10, bieu 2/9, reading 2/10, serial 0/8 |
| Đúng cả hồ sơ | Cả 5 value khớp | ≥ 85% | Mockup 96/200; ảnh thật chưa đủ mẫu |
| Tự điền sai | Trường có đáp án null mà hệ thống điền giá trị; ≥ 30 cơ hội | ≤ 1% | Mockup: reading 0/20; serial 8/20 (serial bị dán che mất đuôi); unit 4/4 |
| Tỷ lệ chuyển người duyệt | Sau khi hiệu chỉnh confidence trên validation | ≤ 40% | 100% (chưa hiệu chỉnh) |
| Tự chấp nhận | Cận dưới Wilson 95% của nhóm tự chấp nhận; 0 lỗi cần ≥ 381 ca | ≥ 99%; không đủ mẫu thì giữ tắt | Tắt (code chặn) |
| Thời gian mỗi ảnh | p95 đủ 5 trường, laptop CPU | ≤ 60 s | p50 0,42 s, p95 0,88 s (mockup, 2 luồng CPU; trước đây 37–40 s khi còn VLM) |
| RAM đỉnh | Tiến trình suy luận | ≤ 4 GiB | ≈ 0,7 GB (trước đây ≈ 2,1 GB khi còn VLM) |
| Không gửi dữ liệu ra ngoài | Suy luận với Docker `--network none` | Đạt | Đã kiểm chứng 04/11; cần chạy lại sau khi bỏ VLM |
| FAQ trả đúng | ≥ 100 câu trong phạm vi, diễn đạt khác nhau | ≥ 90% đúng mã FAQ | Chưa có trong bản C# (bản Java cũ có 3 câu demo) |
| Câu ngoài phạm vi bị trả nhầm | ≥ 50 câu ngoài FAQ | ≤ 2% | Chưa đo |
| PII lộ | ≥ 200 ticket gài PII giả, quét bằng bộ của Long | 0 PII lộ; 0 giá trị công tơ bị che nhầm | Chưa đo |
| Ticket gửi đúng một lần | Gửi lặp, mất mạng giả lập | 100% không trùng | Prototype cũ: gửi 2 lần, nhận 1 |
| Thời gian thao tác của người dùng | So với nhập tay; ≥ 2 người, ≥ 20 tác vụ | Giảm trung vị ≥ 20% | Chưa đo |

### 1.6 Kiến trúc

Ứng dụng ASP.NET Core (.NET 10) chỉ nghe trên `127.0.0.1`. Toàn bộ suy luận viết bằng C# trên ONNX Runtime (CPU). Lúc chạy không cần Python, không gọi dịch vụ ngoài; model được kiểm tra checksum trước khi nạp. Huấn luyện là bộ công cụ Python riêng, chạy offline trong Docker `--network none`. Log chỉ ghi route, mã trạng thái, thời gian và loại lỗi, không ghi ảnh hay chữ đọc được.

| Luồng | Các bước |
|---|---|
| Đọc ảnh | Ảnh PNG/JPEG (≤ 8 MiB, xoay theo EXIF) → **PP-OCRv6 tiny** tìm vùng chữ, đọc chữ → tìm 5 trường theo nhãn Số sx, Kiểu, OBIS và vị trí → vùng chỉ số: đọc lại bằng **model fine-tune T3** và khung nới rộng; chưa thấy biểu thì tìm lại quanh chỉ số, phóng to → **bỏ phiếu 4 lần đọc** chỉ số (2 model × ảnh gốc/xám), không thống nhất thì để trống → chuẩn hóa, kiểm tra theo danh mục (thanh ghi, số chữ số, lịch sử) → người duyệt xác nhận đủ 5 trường → JSON `meter-fields.v1` |
| Tìm vùng bằng YOLOX (thử nghiệm, đang tắt) | YOLOX-nano tìm khung 5 trường khi luật chữ không tìm ra. Tăng số trường đúng (R0 + mockup: 802 → 917) nhưng còn thêm ca điền sai nên chưa bật |
| FAQ và ticket | Câu hỏi → **masker PII của Long** (keep giữ giá trị công tơ) → tra FAQ đã duyệt → trả lời có nguồn, hoặc nháp ticket → người dùng xác nhận → hộp thư local, gửi idempotent |
| Huấn luyện (tách khỏi ứng dụng) | Label Studio local → manifest chia theo công tơ, test khóa → train trong Docker không mạng → chọn theo kế hoạch viết trước → xuất ONNX có checksum → bật qua `runtime.json` |

### 1.7 Mốc chính

T0 là ngày Gate 0 được duyệt. Dữ liệu masked của mentor dự kiến có từ 19/11.

| Mốc | Việc chính | Bằng chứng | Tình trạng |
|---|---|---|---|
| Gate 0 (04/11) | Chốt phạm vi, 5 trường, metric, rubric, quyền dữ liệu, contract với Long | Hồ sơ này được mentor duyệt | Chờ duyệt |
| Gate 1 (T0 + 3 ngày làm việc) | Sửa code trước khi train: bỏ giới hạn ME-41/ME-42, danh mục công tơ thật (mã đã che), tắt adapter VLM cũ. Đưa FAQ/ticket sang bản C#. Sinh bộ mockup | Test qua; fixture hợp đồng dùng chung với Long | Đã bỏ giới hạn kiểu; VLM bỏ hẳn; mockup đã sinh. **Còn:** danh mục thật, FAQ/ticket C# |
| CP1 (T0 + 10 ngày) | Baseline 5 trường trên ảnh nguyên tấm của validation, tách lỗi tìm vùng và lỗi đọc chữ. Masker một chiều trong luồng FAQ/ticket | Báo cáo baseline có fingerprint, hash dữ liệu | Benchmark 5 trường đã có (mockup + 10 ảnh thật), có fingerprint và hash. **Còn:** masker |
| CP2 (T0 + 20 ngày) | Nhận dữ liệu masked, lập tập test mới. Fine-tune có kiểm soát nếu baseline chưa đạt (learning rate thấp, kiểm tra chéo theo công tơ). Hiệu chỉnh confidence. Tích hợp hai chiều với Long | So sánh với baseline trên cùng validation | Đã fine-tune T3 và kiểm chéo theo công tơ trên ảnh VGreen; chờ dữ liệu masked |
| CP3 (T0 + 30 ngày) | Chạy test một lần, UAT với người dùng nghiệp vụ, bàn giao tái lập | Báo cáo test, biên bản UAT, README chạy lại được | README, tài liệu kiến trúc và CI đã có |

---

## 2. Rubric đã cụ thể hóa

Mỗi mức được xác định bằng số đo trên tập test ảnh thật đã che PII, cùng harness và cùng fingerprint model. Mức cao gồm điều kiện của mức thấp hơn. Gate 0 và Gate 1 chỉ đạt hoặc chưa đạt, được ghi vào tiêu chí 3.

### Tiêu chí 1 · Giá trị và kết quả

| Mức | Điều kiện đo được |
|---|---|
| **2 · Đạt** | Luồng ảnh → OCR → 5 trường → duyệt → JSON chạy trên ≥ 50 ảnh thật từ ≥ 10 công tơ. reading và bieu ≥ 85%, ba trường còn lại ≥ 80%. Không có ca tự chấp nhận sai. FAQ ≥ 80% đúng mã; luồng nháp → xác nhận → hộp thư chạy qua masker của Long. |
| **3 · Tốt** | Đạt mọi metric CP3 ở mục 1.5: ≥ 100 ảnh từ ≥ 20 công tơ; reading, serial, bieu ≥ 95%; model, unit ≥ 90%; cả hồ sơ ≥ 85%; p95 ≤ 60 s. FAQ ≥ 90%, trả nhầm ≤ 2%; 0 PII lộ trên 200 ticket. Ít nhất 1 người vận hành dùng thử cả hai luồng (≥ 20 ảnh, ≥ 20 câu hỏi), phản hồi được ghi lại. |
| **4 · Xuất sắc** | reading và bieu ≥ 97%, serial ≥ 98%, model và unit ≥ 95%, cả hồ sơ ≥ 90%. Thời gian thao tác giảm ≥ 30% với 2 người dùng. Người dùng nghiệp vụ đề xuất chạy thử thật, có biên bản hoặc backlog được duyệt. |

### Tiêu chí 2 · Chất lượng kỹ thuật

| Mức | Điều kiện kiểm chứng được |
|---|---|
| **2 · Đạt** | README chạy được từ đầu; toàn bộ check C# và HTTP qua (hiện 84 + 27), build 0 cảnh báo; smoke offline với `--network none` thoát 0. JSON 100% đúng schema. |
| **3 · Tốt** | Benchmark ghi fingerprint model, code, runtime, hash dữ liệu, seed. Chọn model và ngưỡng chỉ trên validation, theo kế hoạch viết trước; test chia theo công tơ, khóa, chạy một lần. Báo lỗi theo trường và theo bước (tìm vùng hay đọc chữ). Refactor được kiểm chứng không đổi kết quả (0 khác biệt trên 1.050 trường). Có test cho "masker lỗi thì không lưu/gửi" và "gửi lặp không tạo ticket trùng". Không có ảnh thật hoặc PII trong repo, log, ZIP bàn giao. |
| **4 · Xuất sắc** | Người khác dựng lại trên máy sạch và ra cùng số liệu trong ≤ 2 giờ chỉ theo README. Thêm kiểu công tơ mới chỉ bằng danh mục và nhãn, thêm câu FAQ chỉ bằng file FAQ, không sửa code. Runbook train, hiệu chỉnh, rollback đã chạy thử một lần. CI chạy check mỗi lần push. |

### Tiêu chí 3 · Thực thi và chủ động

| Mức | Bằng chứng |
|---|---|
| **2 · Đạt** | ≥ 80% đầu việc của mốc xong đúng checkpoint; Gate 0 và Gate 1 đạt; cập nhật tiến độ ít nhất 1 lần/tuần. |
| **3 · Tốt** | 100% mốc đúng hạn hoặc được mentor duyệt đổi trước hạn. Rủi ro mới ghi vào bảng rủi ro trong ≤ 1 ngày làm việc; góp ý của mentor xử lý trong ≤ 2 ngày làm việc. |
| **4 · Xuất sắc** | Điều chỉnh kế hoạch dựa trên số liệu ít nhất 2 lần, có ghi lý do. Ví dụ đã có: dừng dùng bản fine-tune khi test giảm từ 1/4 xuống 0/4; chọn T3 và loại T4 theo luật viết trước vì T4 thêm một ca điền sai; không bật YOLOX dù tăng số trường đúng, vì thêm ca điền sai; bỏ VLM vì đúng 1/10 và chậm 35 s/ảnh. Pair contract được cả hai bên dùng và tích hợp hai chiều chạy được. |

---

## 3. Pair contract · Cặp A (Long – Trung)

| Bên | Phụ trách | Mentor |
|---|---|---|
| Đàm Quang Trung | OCR công tơ 5 trường, FAQ/chatbot, ticket | PhuNN4 |
| Đoàn Tuấn Long | Phát hiện và che PII trong văn bản, ticket, log | TungNX15 |

> **Phiên bản:** v0.1 là bản trong Google Doc "Contract", tab 2; mục 3.6 liệt kê các thay đổi so với v0.1. Chỉ dùng mockup đến CP1; dữ liệu masked của mentor từ 19/11. Hiện chưa tích hợp masker.

### 3.1 Hai bài toán giao nhau ở đâu, giá trị cộng hưởng là gì

- **Nơi giao nhau:** câu hỏi FAQ, ticket và văn bản OCR (các dòng chữ trên ảnh ngoài chỉ số và serial, ví dụ nhãn dán) là văn bản tự do, hay chứa tên, SĐT, email, địa chỉ trạm, biển số xe, số CCCD. Đây là đầu vào thật cho bộ phát hiện PII của Long.
- **Bài toán khó chung:** ticket phải giữ nguyên dữ liệu nghiệp vụ (serial, thanh ghi, chỉ số) trong khi che PII. Serial dài nhiều chữ số dễ bị nhận nhầm là SĐT hoặc số định danh, nên cần che có allowlist.
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
- Dấu thập phân lấy đúng như trên ảnh, không chèn (mục 1.4); dấu thừa hoặc chuỗi không phải số thì `value = null` và chuyển người duyệt.

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

Log vận hành của ứng dụng OCR không chứa nội dung: chỉ giờ, route, mã trạng thái, thời gian và loại lỗi. Log có PII để Long thử là bộ log mockup ở trên, không phải log thật.

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

> Mockup **không bao giờ vào validation hoặc test ảnh thật** và không dùng để báo độ chính xác hay chọn ngưỡng tự chấp nhận. Code đã từ chối dòng `source=synthetic` ở validation/test. Bộ mockup giữ riêng chỉ dùng để so sánh giữa các phiên bản app; bộ nào đã dùng để xem lỗi thì không còn là bộ giữ riêng.

### 4.1 Thành phần

| Tập | Số lượng | Cách tạo | Tình trạng |
|---|---|---|---|
| Danh mục giả lập | 20 công tơ mỗi bộ (`MOCK-01`…`MOCK-20`), 5 kiểu LCD và công tơ cơ | `catalog.mock.json`, `synthetic=true`; thanh ghi có mã OBIS và số chữ số | Đã có |
| Ảnh mặt công tơ đủ 5 trường | 1.200 ảnh train (v1, v2, 10 bộ seed 20261201–20261210) và 600 ảnh giữ riêng (6 bộ seed 20261301–20261306), 100 ảnh mỗi bộ | `mockup_faceplates.py` dựng mặt công tơ từ thống kê bố cục, font DejaVu; `--hard 0.5` mô phỏng ảnh chụp điện thoại (xiên, lóa, rung, nén) | Đã có; cùng lệnh sinh lại đúng từng byte |
| Crop chỉ số LCD | 1.200 crop | Cắt từ ảnh train, nhãn đúng chữ nhìn thấy (gồm số nguyên và dấu thừa) | Đã dùng train model T3 |
| Ảnh có PII giả | 50 ảnh | Nhãn dán tên, SĐT, biển số giả; EXIF GPS giả | Chưa làm |
| FAQ mẫu | 20–30 mục | Soạn từ quy trình chụp và đọc chỉ số; business duyệt trước khi dùng thật | Chưa làm |
| Câu hỏi test FAQ | 150 câu: 100 trong, 50 ngoài phạm vi | Diễn đạt khác, không dấu, viết tắt, lỗi chính tả | Chưa làm |
| Ticket và log gài PII giả | 200 ticket, 100 dòng log | Tên, SĐT, email, CCCD, địa chỉ, biển số giả trộn với serial và chỉ số | Chưa làm |

### 4.2 Phân bố mỗi bộ 100 ảnh mặt công tơ

| Nhóm | Số ảnh | Tình huống |
|---|---:|---|
| Rõ | 20 | Các kiểu cân bằng; nhãn Số sx, No., S/N; nhãn Kiểu, Type, Model |
| Biểu và OBIS | 20 | Mã 1.8.x, 2.8.x; Tổng/Total; Biểu và Rate 1–3; một số ca không có mã |
| Dấu thập phân | 15 | Có dấu ở các vị trí khác nhau; thiếu dấu (màn hình số nguyên); dấu chấm thừa (phải ra `value = null`) |
| Chất lượng ảnh | 20 | Lóa, mờ, tối, độ phân giải thấp |
| Cắt hoặc che khuất | 10 | Ảnh cắt ngang serial (serial và kiểu phải ra null); nhãn dán che đuôi serial |
| Nhãn dễ nhầm và đơn vị | 10 | Tên hãng cạnh kiểu công tơ; hằng số xung gần chỉ số; kWh, kVArh, đơn vị khác |
| Không hỗ trợ | 5 | Công tơ cơ; phải chuyển người duyệt |
| **Tổng** | **100** | |

### 4.3 Mỗi dòng manifest (JSONL)

| Trường | Quy ước |
|---|---|
| `id`, `image` | id duy nhất; đường dẫn tương đối từ thư mục manifest; báo cáo benchmark ghi hash từng ảnh và hash manifest |
| `meterId`, `registerId` | Có trong danh mục; mọi ảnh cùng công tơ thuộc cùng một tập |
| `fields` | Đáp án 5 trường: `raw`, `value` (đúng như trên ảnh, giữ số 0 đầu và dấu thập phân) hoặc `nullReason` khi không đọc được (ví dụ `SERIAL_CUT`, `SERIAL_OCCLUDED`, `EXTRA_SEPARATOR`), và `bbox` |
| `scenario`, `conditions` | Nhóm ở mục 4.2; hiệu ứng thực tế của từng ảnh (`perspective`, `glare`, `motion-blur`, `cut`, `occluded`, `decimal-missing`…) |
| `split` | `train` hoặc `smoke`; seed ghi trong `summary.json` của bộ |

```json
{"id": "syn-0001", "image": "images/syn-0001.jpg", "meterId": "MOCK-03", "registerId": "B1", "scenario": "decimal", "conditions": ["decimal", "perspective"], "split": "smoke", "fields": [{"displayReading": {"raw": "00210.6", "value": "00210.6", "nullReason": null, "bbox": [0.31, 0.40, 0.70, 0.51]}}]}
```

### 4.4 Quy tắc

- Mockup chỉ ở train, smoke hoặc bộ so sánh giữa các phiên bản app; không bao giờ là validation/test ảnh thật.
- Ảnh thật chỉ vào dự án khi đã qua che PII và được owner dữ liệu cho phép; lưu ngoài repo, không đưa vào repo, ZIP hay cuộc trò chuyện. Script đánh giá trên ảnh thật chỉ in số đếm tổng hợp.
- Mỗi phiên bản dữ liệu có hash manifest và hash từng ảnh; báo cáo benchmark ghi lại hai hash này.
- Câu hỏi test FAQ không được dùng làm bí danh trong file FAQ.

---

## 5. Rủi ro và phụ thuộc

> Rủi ro lớn nhất vẫn là thiếu dữ liệu thật độc lập: 57 ảnh của khoảng 17 công tơ, đã dùng nhiều lần để chẩn đoán. Trên ảnh thật, bước yếu nhất là **tìm vùng** từng trường.

### 5.1 Rủi ro

| Rủi ro | Khả năng | Ảnh hưởng | Giảm thiểu | Dấu hiệu báo sớm |
|---|---|---|---|---|
| Dữ liệu thật quá ít, ít công tơ | Đã xảy ra | Không đánh giá được; fine-tune dễ học thuộc | Kiểm chéo theo công tơ (5 fold); dữ liệu masked từ 19/11 | Tới CP1 dưới 50 ảnh |
| Ảnh thật hiện có đã bị dùng nhiều lần | Đã xảy ra | Số đo trên đó lạc quan | Lập tập test mới từ công tơ chưa từng thấy, khóa trước khi train | Chưa có tập test mới ở CP2 |
| Tìm sai vùng trên ảnh thật | Đã xảy ra | Phần lớn lỗi ảnh thật nằm ở bước tìm vùng (10 ảnh: chỉ số đúng 2/10) | Thử YOLOX tìm vùng (tìm đúng 50/50 trường trên 10 ảnh); chỉ bật khi không thêm ca điền sai | Lỗi tìm vùng chiếm phần lớn trên dữ liệu masked |
| Điền sai số công tơ bị che hoặc cắt | Đã xảy ra (mockup) | Số serial thiếu chữ số được điền | Luật lóa/mờ/chạm mép; đối chiếu danh mục; người duyệt | Có ca điền sai serial trên dữ liệu masked |
| Mockup không giống ảnh thật | Cao | Cải thiện trên mockup không chuyển sang ảnh thật | Luôn đo thêm trên ảnh thật; quyết định cuối trên dữ liệu masked | Mockup tăng nhưng ảnh thật không tăng |
| Confidence chưa hiệu chỉnh | Cao | 100% hồ sơ chuyển người duyệt; điểm ≥ 0,99 vẫn sai 5–8% | Bỏ phiếu nhiều lần đọc thay vì tin điểm; hiệu chỉnh khi đủ công tơ; giữ tự chấp nhận tắt | Chưa đủ mẫu ở CP2 |
| Nhãn ảnh thật có thể sai hoặc thiếu | Trung bình | Chấm sai một trường (ví dụ 1 ảnh có mã biểu trên màn hình nhưng nhãn ghi không có) | Người phụ trách dữ liệu xem lại các ca lệch; không sửa nhãn tự động | Ca lệch lặp lại ở cùng loại trường |
| Ảnh nghiêng, chữ dính nhau | Trung bình | Vùng chữ lấy theo trục ảnh nên cắt sai | Báo lỗi riêng theo bước tìm vùng; yêu cầu chụp lại | Lỗi tìm vùng chiếm phần lớn |
| Màn hình tự lật, nhầm biểu | Trung bình | Đúng số nhưng sai thanh ghi | Kiểm tra OBIS trên ảnh; sai thì chụp lại | Tỷ lệ chụp lại > 15% |
| Masker bỏ sót PII tiếng Việt hoặc che nhầm serial | Cao | Lộ dữ liệu hoặc ticket mất thông tin | `keep` theo giá trị; `meterData` tách khỏi văn bản; người dùng xem bản đã che | Có PII lộ trên bộ 200 ticket |
| FAQ/ticket chưa có trong bản C# | Cao | Trễ CP1 | Chuyển từ prototype Java cũ; ưu tiên luồng che một chiều | Gate 1 chưa xong |
| Nội dung FAQ chưa được business duyệt | Trung bình | Không cho người dùng thử được | Soạn nháp từ quy trình, xin duyệt sớm | Chưa có người duyệt sau Gate 0 |
| Giới hạn máy: 16 GB RAM, không GPU | Trung bình | Train song song có thể hết RAM (Docker đã bị dừng một lần) | Giới hạn RAM từng container; bộ canh dừng job phụ khi RAM thấp | Container bị dừng vì RAM |

*Đã xử lý:* kiểu công tơ ngoài ME-41/ME-42 (đã nhận mọi mã kiểu hợp lệ); adapter VLM cũ có thể đã thấy ảnh test và VLM làm chậm khi OCR hụt trường (đã bỏ VLM).

### 5.2 Phụ thuộc

| Phụ thuộc | Bên cung cấp | Cần trước |
|---|---|---|
| Danh sách kiểu công tơ, thanh ghi mục tiêu, danh mục với mã đã che | Mentor PhuNN4, vận hành VGP/CPO | Gate 1 |
| Ý nghĩa BT/CĐ/TĐ và mã OBIS; quy ước đọc công tơ cơ; luật màn hình số nguyên (mục 1.4) | Mentor PhuNN4 | CP1 |
| Quyền dữ liệu: nơi lưu, thời gian lưu, được train ở đâu | Mentor, an toàn thông tin | Gate 0 |
| Dữ liệu masked và người xác nhận đáp án | Mentor, vận hành VGP/CPO | Từ 19/11 |
| Masker PII, schema, danh sách loại PII | Đoàn Tuấn Long | Sprint 1 |
| Nội dung FAQ và người duyệt | Business/CSKH VGP | Đo FAQ, dùng thử |
| Bên nhận ticket: địa chỉ nội bộ, quyền gửi, hợp đồng API | Business/IT VGP | Gửi thật, CP2 |
| Người dùng thử | Mentor giới thiệu | CP1 |
| Máy nội bộ để train | Quản trị IT | Fine-tune lớn hơn |

---

*Thay đổi so với bản Gate 0 trước:* kiến trúc chuyển sang C# (.NET 10, ONNX Runtime), không còn worker Python lúc chạy; mở rộng từ chỉ số sang 5 trường; chốt quy tắc dấu thập phân; thêm kết quả thử train và rủi ro mới; pair contract kèm các thay đổi so với v0.1. Bỏ bộ đọc VLM (PaddleOCR-VL-1.6, LLamaSharp) vì đúng 1/10 crop thật và khoảng 35 s/ảnh: ứng dụng chỉ còn OCR, nhanh hơn khoảng 40 lần, RAM khoảng 0,7 GB. Thêm model đọc chỉ số fine-tune T3, bỏ phiếu 4 lần đọc, luật màn hình số nguyên (chờ mentor xác nhận), luật số công tơ không giới hạn độ dài. Code chia lại theo module, có log không chứa nội dung, CI và tài liệu kiến trúc/đánh giá; refactor được kiểm chứng không đổi kết quả trên 1.050 trường. Số liệu "Hiện tại" lấy từ benchmark trên máy phát triển.
