# SPEC — AI Product Hackathon

**Nhóm:** 30-403-Day05

**Track:** VinFast

**Problem statement:** *Khách hàng tìm hiểu mua xe VinFast gặp khó khăn khi tra cứu thông số, chính sách và thường phải chờ đợi lâu ngoài giờ hành chính mới được hỗ trợ; Web app tư vấn AI độc lập (Streamlit, giả lập) hoạt động 24/7, LangGraph + Tavily real-time search kết hợp LLM reasoning (`constants.REASONING_MODEL`, ví dụ gpt-5.4) trả lời kèm JSON có `confidence` và `source_url`, guardrails (`constants.GUARDRAIL_MODEL`, ví dụ gpt-5.4-mini) phân loại và chặn input không phù hợp, và luồng đặt lịch qua form ghi lead vào `data/training_data.jsonl`.*

---

## 1. AI Product Canvas

|   | Value | Trust | Feasibility |
|---|-------|-------|-------------|
| **Câu hỏi** | User nào? Pain gì? AI giải gì? | Khi AI sai thì sao? User sửa bằng cách nào? | Cost/latency bao nhiêu? Risk chính? |
| **Trả lời** | *Khách muốn mua xe. Pain: Chờ đợi CSKH lâu, tự đọc web khó. AI giải: Trả lời ngay (thông số, giá, chính sách — theo nguồn search) và form thu thập thông tin đặt lịch / tư vấn.* | *AI có thể báo sai giá. Ở bản demo: có trường `confidence` (0–10) và `suggest_human` trong JSON; UI hiện card “kết nối tư vấn viên” khi `confidence < CONFIDENCE_THRESHOLD (7)` **hoặc** `suggest_human` true (OR); người dùng 👎 ghi nhận vào JSONL để huấn luyện sau; chưa có bước “xin lỗi + flag phiên” tự động như story Failure.* | *Cost/latency phụ thuộc OpenAI + Tavily calls (tối đa 2 vòng search/lượt). Risk: sai lệch chính sách; guardrail giảm COMPETITOR/OFF_TOPIC/SENSITIVE nhưng không thay thế pháp lý.* |

**Automation hay augmentation?** ☐ Automation · ☑ Augmentation
Justify: *Augmentation — AI đảm nhiệm khâu lọc "phễu đầu" (tư vấn cơ bản, xin thông tin, lên lịch), quyết định chốt sale và tư vấn tài chính chuyên sâu vững nhất vẫn là nhân sự (Sales/CSKH) tiếp nhận phần Context sau đó.*

**Learning signal:**

1. User correction đi vào đâu? *👍/👎 trên từng câu assistant (và lead/blocked) được `logger.append_entry` ghi vào `demo/data/training_data.jsonl`; Admin (`admin.py`) xem bảng và export JSONL. Roadmap: pipeline “verified” / RAG ngoài phạm vi file này.*
2. Product thu signal gì để biết tốt lên hay tệ đi? *Tỷ lệ lead (`label: lead`) so với số lượt chat; tỷ lệ 👎; tần suất hiển thị card tư vấn viên (proxy từ `confidence`/`suggest_human`). Đo đầy đủ cần tích hợp analytics ngoài app.*
3. Data thuộc loại nào? ☐ User-specific · ☑ Domain-specific · ☐ Real-time · ☐ Human-judgment · ☐ Khác: ___
   Có marginal value không? (Model đã biết cái này chưa?) *Có. Base model không tự biết chắc cấu hình mới nhất, giá lăn bánh theo tỉnh và khuyến mãi thay đổi; Tavily + prompt VinFast bổ sung thời điểm gọi.*

---

## 2. User Stories — 4 paths

### Feature: *Chatbot tư vấn xe & Đặt lịch lái thử*

**Trigger:** *User vào trang chủ VinFast, bấm vào widget Chat và hỏi "Tư vấn cho mình VF 8"*

| Path | Câu hỏi thiết kế | Mô tả |
|------|-------------------|-------|
| Happy — AI đúng, tự tin | User thấy gì? Flow kết thúc ra sao? | *AI trả lời trong chat (có thể kèm tool search tối đa 2 lần). User có thể mở form “Để lại thông tin”: họ tên, SĐT, ngày — submit ghi một dòng `lead` vào `training_data.jsonl`. **Không** có HTTP API “đặt lịch” hay CRM trong repo demo.* |
| Low-confidence — AI không chắc | System báo “không chắc” bằng cách nào? User quyết thế nào? | *Model trả JSON gồm `confidence` và tùy chọn `suggest_human`. `engine.node_parse_answer` đặt cờ hiển thị khi `confidence < 7` **OR** `suggest_human` true. `app.py` hiện cảnh báo cố định và nút [Gọi tư vấn viên] (không render `reason` riêng; lý do có thể nằm trong văn bản `answer`). User tự chọn bấm hoặc bỏ qua.* |
| Failure — AI sai | User biết AI sai bằng cách nào? Recover ra sao? | *Người dùng dựa trên kiến thức / web và có thể 👎 để lưu cặp Q&A sai vào JSONL. **Chưa** có luồng chatbot tự động: xin lỗi, nhắc lại URL nguồn, hay “flag phiên” trong state.* |
| Correction — user sửa | User sửa bằng cách nào? Data đó đi vào đâu? | *Nút phản hồi là 👎 (“Sai”), gọi `append_entry` với cùng câu trả lời assistant; logger hỗ trợ tham số `correction` nhưng **UI chưa thu text “đáp án đúng”**. Admin xem log trên Dashboard và export; **Verified Answer Cache / serve lại từ cache chưa triển khai**.* |

---

## 3. Eval metrics + threshold

**Optimize precision hay recall?** ☑ Precision · ☐ Recall
Tại sao? *Vì thông số kỹ thuật, bảo hành, và đặc biệt là giá bán xe mang tính pháp lý/thương hiệu rất cao. Thà AI báo "Xin phép chuyển CSKH" (Low Recall) còn hơn là "Bịa ra mức giá không có" (Low Precision) gây khủng hoảng truyền thông.*

| Metric | Threshold | Red flag (dừng khi) |
|--------|-----------|---------------------|
| *Answer Accuracy (Độ chính xác nội dung so với Fact)* | *≥ 95%* | *< 85% trong 1 tuần liên tục (Cần Freeze để fix DB)* |
| *Booking Conversion Rate (Số lượng chat tạo ra Lead)* | *≥ 10%* | *< 2% (Logic luồng gọi API Tool bị fail)* |
| *Hallucination / Toxic Rate* | *≤ 1%* | *> 5% (Nguy cơ khủng hoảng thương hiệu)* |

---

## 4. Top 3 failure modes

| # | Trigger | Hậu quả | Mitigation |
|---|---------|---------|------------|
| 1 | *User gài bẫy: (a) prompt injection / hỏi về hệ thống / API key / dữ liệu người dùng, (b) so sánh tiêu cực xe đối thủ, (c) câu hỏi ngoài chủ đề dịch vụ.* | *(a) Lộ system prompt hoặc thông tin nội bộ; (b) AI nói xấu đối thủ hoặc chê hãng; (c) Chatbot trả lời lan man mất focus.* | *Guardrail layer riêng dùng gpt-5.4-mini phân loại mỗi input thành: SENSITIVE (injection/system/data) → "Câu hỏi thuộc loại nhạy cảm, em không thể hỗ trợ"; COMPETITOR → lịch sự bẻ hướng VinFast; OFF_TOPIC → "Em chỉ hỗ trợ tư vấn xe VinFast"; PASS → xử lý bình thường. Tất cả phản hồi đều lịch sự.* |
| 2 | *Tavily API trả về kết quả cũ hoặc từ nguồn không chính thống (báo lá cải, forum) — LLM tổng hợp thành câu trả lời sai thông số/giá.* | *Tư vấn sai giá, sai thông số cho khách hàng; LLM confidence score tự chấm vẫn cao dù nguồn sai.* | *Prompt bắt buộc JSON có `source_url`. Query Tavily luôn prefix `VinFast` trong `search.py`. **Chưa** có bước kiểm tra domain “chính thức” hay hạ `confidence` tự động theo domain; giảm rủi ro bằng ngưỡng `confidence` + `suggest_human` và card chuyển tư vấn viên.* |
| 3 | *Hệ thống API gọi CRM (Save lead) bị timeout hoặc down server.* | *Khách nhập số điện thoại nhưng AI báo lỗi thất bại, làm khách cụt hứng.* | *Hiện tại lead chỉ ghi file local (append JSONL). **Chưa** gọi CRM; khi tích hợp API cần thêm retry/queue và thông báo fail-safe như mô tả.* |

---

## 5. ROI 3 kịch bản

|   | Conservative | Realistic | Optimistic |
|---|-------------|-----------|------------|
| **Assumption** | *500 user/ngày, 5% đặt lịch* | *2000 user/ngày, 8% đặt lịch* | *5000 user/ngày, 12% đặt lịch* |
| **Cost** | *$3/ngày (Inference LLM)* | *$10/ngày* | *$25/ngày* |
| **Benefit** | *Tiết kiệm 4h trực chat (~$12), thêm 25 Lead* | *Giảm 16h CSKH, thêm 160 Lead* | *Bao trọn 80% L1 Support, thêm 600 Lead* |
| **Net** | *Dương nhẹ + Lead* | *Dương rất lớn* | *Đóng góp trực tiếp làm thay đổi Data Sale của hãng* |

**Kill criteria:** *Có hiện tượng chụp màn hình bóc phốt lan truyền MXH vì AI tư vấn sai luật (Critical Brand Harm). Hoặc sau 2 tháng Cost duy trì AI > Benefit lượng Lead mang về (Ví dụ: khách vào trêu đùa quá đông nhưng không ai đặt lịch).*

---

## 6. Mini AI spec 

**1. Sản phẩm & Giải pháp:**
VinFast Auto-Agent là Web app tư vấn xe AI độc lập (giả lập — không phải website chính thức của VinFast). Sử dụng kiến trúc 2 lớp LLM: **Guardrail layer** (gpt-5.4-mini) phân loại và lọc input trước, **Reasoning + LangGraph** (gpt-5.4) xử lý câu hỏi hợp lệ với **một tool gọi được:** `search_web_tool` → Tavily (`search.py`, query `VinFast {query}`, tối đa 3 snippet). Phần “**suggestConsultation**” trong thiết kế **không phải tool riêng**: model trả `suggest_human` trong JSON kết thúc; kết hợp **OR** với `confidence < 7` để `app.py` hiện nút [Gọi tư vấn viên] (không tự động gọi).

**2. Khách hàng/User:** 
Khách hàng tiềm năng đang trong giai đoạn tìm hiểu (Consideration) muốn tra cứu thông số, so sánh giá cả siêu tốc và lên lịch lái thử không bị làm phiền; Khách hàng hiện tại tìm kiếm trợ giúp kỹ thuật xe.

**3. Bản chất sản phẩm (Auto/Aug):**
Augmentation (Tiền tuyến - L1 Support). Tự động chặn và giải quyết nhanh 80% câu hỏi FAQ (cấu hình, giá cọc, so sánh bản Plus/Eco...). Ngay khi có mùi "Khó", "Tài chính phức tạp" hoặc "Khiếu nại", lập tức Freeze và đề nghị chuyển ngữ cảnh chat cho tư vấn viên để chốt sale/chăm sóc. Đảm bảo user có Human touch ở chặng cuối.

**4. Metrics & Quality Trade-offs:**
Hệ thống ưu tiên tuyệt đối vào **Precision > Recall**. Chấp nhận việc AI có lúc thoái thác trả lời để nhường cho Human, tuyệt đối tránh hiện tượng bịa giá (đảo lộn bảng giá). 

**5. Cơ chế Data Flywheel:**
*Trong bản demo hiện tại: phản hồi 👍/👎/lead/blocked được append `training_data.jsonl`, Admin xem và export; chưa có “verified” workflow hay cache phục vụ câu hỏi lặp.* Chu trình mục tiêu đầy đủ: Càng nhiều khách hàng hỏi → Lộ ra Edge cases LLM không trả lời được (confidence thấp, Tavily không trả về nguồn chính thức) → Các cặp Q&A Low-confidence được tự động ghi vào Dashboard → Admin review, chỉnh sửa và xác nhận câu trả lời đúng ("verified") → Câu đó được lưu vào **Verified Answer Cache** (JSON/localStorage) → Lần sau cùng câu hỏi tương tự, hệ thống serve thẳng từ Cache mà không cần gọi Search (giảm latency + tránh kết quả sai nguồn) → Coverage Cache rộng dần, tỷ lệ Low-confidence giảm, Cost/query giảm theo. Toàn bộ log có thể export ra `.jsonl` để fine-tune LLM ở giai đoạn sau.
