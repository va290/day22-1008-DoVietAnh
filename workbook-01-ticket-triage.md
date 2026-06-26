# Workbook — Case 1: Support Ticket Triage (bài làm)

> Người làm: Đỗ Việt Anh · 2A202601008 · Day 22 — Eval Decision Workbook
> Đề gốc: [`01-ticket-triage.md`](01-ticket-triage.md)

## 1. Unit of Work
**Lát cắt:** *Một ticket support đi vào → AI gán `category`, `urgency`, `route_to`, cờ `requires_human`, kèm `reason_codes` + `confidence`.* Output dùng bởi **hệ thống inbox nội bộ + nhân viên hỗ trợ** (không gửi khách).
Đây là đơn vị đủ nhỏ để eval vì: input rõ (1 ticket JSON), output quan sát được (mấy field enum), và failure xác định được (route sai hàng, thiếu escalation). Nếu sai → ticket trễ, bỏ sót khách enterprise đang bị chặn việc, hoặc đẩy sang team không xử lý được.

## 2. Quality Question
> **"AI có gán đúng category + urgency + route và bật `requires_human` đúng lúc để ticket không đi sai hàng và không bỏ sót escalation cho khách enterprise đang bị chặn việc không?"**

Fail ở đây → ticket billing/critical bị chấm "Medium, không cần người" (đúng như mock T-002): khách enterprise locked-out chờ vô thời hạn, mất trust, vi phạm SLA. Behavior **bắt buộc**: enterprise + (high|critical) ⇒ `requires_human=true`. Behavior **bị cấm**: tự tin gán nhãn khi tín hiệu mơ hồ; bịa `reason_codes` không có trong ticket.

## 3. Output Contract tối thiểu
| Field | Vì sao cần |
|---|---|
| `category` (enum) | Render UI + chọn hàng đợi; sai → route sai team |
| `urgency` (enum: low/medium/high/critical) | Quyết định độ ưu tiên + điều kiện escalation |
| `route_to` (enum team) | Đẩy ticket vào đúng queue; trực tiếp gây hậu quả vận hành |
| `requires_human` (bool) | Cờ escalation — quyết định người thật có nhảy vào không |
| `reason_codes` (list, từ tập cố định) | Giải thích + cho phép eval "có bám nội dung ticket không" |
| `confidence` (0–1) | Định tuyến case thấp tin cậy sang human review |
| `ticket_id`, `customer_tier` (echo) | Khoá để eval/đối chiếu + áp rule enterprise |

Bỏ qua: nội dung diễn giải dài, sentiment — không đổi UI/route/gate.

## 4. Eval Decision Map
| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
|---|:--:|:--:|:--:|:--:|---|
| Schema + enum hợp lệ, `confidence`∈[0,1] | ✅ | | | | Hoàn toàn deterministic |
| Rule: enterprise+high/critical ⇒ requires_human | ✅ | | | | Logic cứng, if-then |
| Rule: billing không route `product_team`; "locked out/blocking" không `low` | ✅ | | | | Rule chính sách rõ ràng |
| `category`/`route` có khớp **ý nghĩa** ticket | | ✅ | ✅(spot) | | Cần đọc hiểu nội dung; code không bắt được |
| `reason_codes` phản ánh đúng ticket, không bịa | | ✅ | | | Phán đoán ngữ nghĩa (grounding) |
| Urgency hợp lý với mức "chặn việc" | | ✅ | ✅(biên) | | Mức nghiêm trọng là semantic |
| Case low-info/mơ hồ ("Help asap") xử đúng (unknown/review) | | ✅ | ✅ | | Cần đánh giá agent có "khiêm tốn" không |

## 5. Kiểm tra tự động bằng code (đầy đủ)
- Kiểm tra: output đúng JSON schema, mọi field bắt buộc có mặt. *Code: cấu trúc cố định.*
- Kiểm tra: `category`, `urgency`, `route_to` ∈ tập enum cho phép. *Code: so khớp tập.*
- Kiểm tra: `confidence` là số trong [0,1]. *Code: kiểm tra miền giá trị.*
- Kiểm tra: nếu `customer_tier=enterprise` và `urgency∈{high,critical}` thì `requires_human=true`. *Code: rule if-then, fail = bỏ sót escalation.*
- Kiểm tra: nếu `category=billing` thì `route_to≠product_team`. *Code: rule chính sách.*
- Kiểm tra: nếu ticket chứa từ khoá `blocking work|locked out|account disabled` thì `urgency≠low`. *Code: keyword + so sánh.*
- Kiểm tra: `reason_codes` ⊂ tập mã cho phép (không có mã lạ). *Code: tập hợp con.*
- Kiểm tra: không trùng/không rỗng `reason_codes` khi `requires_human=true`. *Code: ràng buộc hiện diện.*

## 6. Tiêu chí chấm bằng LLM (đầy đủ)
- Tiêu chí: `category` chọn có khớp vấn đề thực mà khách mô tả không. *Code không hiểu nghĩa ticket tự do.*
- Tiêu chí: `route_to` là team **thực sự** xử lý được loại vấn đề này. *Cần suy luận ánh xạ vấn đề→team.*
- Tiêu chí: `urgency` tương xứng mức nghiêm trọng/tác động (chặn việc, mất tiền). *Mức độ là phán đoán ngữ nghĩa.*
- Tiêu chí: `reason_codes` + lý do tóm tắt **có căn cứ** trong ticket, không bịa sự thật. *Grounding check — code không làm được.*
- Tiêu chí: với ticket low-info/mơ hồ, agent có tránh gán nhãn quá tự tin (confidence thấp / route review). *Đánh giá "sự khiêm tốn", semantic.*

## 7. Human / Expert Review
**Ai:** Trưởng ca/QA vận hành support review. **Review case nào:** (a) mọi case `requires_human=true` hoặc `confidence<0.6`; (b) mẫu ngẫu nhiên 10–15% case còn lại để bắt drift; (c) toàn bộ regression (case từng pass nay fail). Họ xác nhận quyết định **vận hành** (route/escalation) đúng thực tế đội ngũ — thứ chỉ người trong quy trình mới biết.
**Domain expert chuyên sâu:** *Không áp dụng.* Đây là triage hành chính SaaS B2B, không có rủi ro chuyên môn (y tế/pháp lý); QA vận hành nắm policy nội bộ là đủ để xác nhận chất lượng.

#### 7A. Màn hình Domain Expert — *Không áp dụng* (không cần expert; xem giải thích mục 7).
#### 7B. Tiêu chí Domain Expert — *Không áp dụng.*

## 8. Release Gate
Chỉ ship cấu hình mới khi **tất cả**:
- **Chặn cứng (0 lỗi):** 100% pass các code-rule an toàn — enterprise+critical⇒human; billing≠product_team; "blocking"≠low; schema/enum hợp lệ. **Bất kỳ vi phạm nào = block.**
- **Ngưỡng chất lượng:** LLM-judge `route_to` đúng ≥ 90%; `requires_human` recall ≥ 98% trên tập escalation; `reason_codes` grounded ≥ 95%.
- **Regression = 0** trên reference dataset (case từng pass không được fail).
- **Human review:** mọi case `confidence<0.6` route sang người; nếu tỷ lệ này >25% → coi như chưa đạt, cần cải thiện trước khi mở rộng.

## 9. Kế hoạch chạy thử & dự toán chi phí
**Giả định:** model agent + LLM-judge = **Claude Haiku 4.5** (giá thật: **$1/1M input, $5/1M output**, niêm yết Anthropic). Pilot **80 cases**, **40 lần chạy/lặp** (tinh chỉnh prompt + gate) → **3.200 lượt**. Mỗi lượt: agent (~800 tok in / 250 out) + judge (~1.200 in / 200 out).
- **Chi phí API:** agent ≈ $0.0021/lượt, judge ≈ $0.0022/lượt → ~$0.0043 × 3.200 ≈ **$14** (làm tròn ~$15 cho retry).
- **Giờ công:** PM/thiết kế eval 16h; Eng/vận hành (instrument + chạy harness) 16h; Human review (gán nhãn 80 case + soi regression qua các vòng) 12h.
- **Quy đổi (giả định VN):** PM/Eng 400k VND/h, Human 200k/h → 16×400k + 16×400k + 12×200k = **15,2 triệu VND** + API ~$15 (~0,4tr) ≈ **~15,6 triệu VND (~$625)**.
- **Tổng thời gian:** ~1,5–2 tuần.

> Mình lấy **giá thật Haiku 4.5 từ bảng giá Anthropic ($1/$5 per 1M)**. Với quy mô này, **chi phí máy chỉ ~$15 — không đáng kể; chi phí thật là giờ công người (~15tr)**. Plan này đủ để chứng minh case pilot được vì nó trả lời được 3 câu: độ chính xác route/escalation hiện tại (qua judge + code-rule), checkpoint còn thiếu (tỷ lệ confidence thấp), và với <20tr ta xác nhận được hướng làm có an toàn để đề xuất mở rộng hay không.
