# Prototype — VinFast Auto-Agent

## Mô tả
Trợ lý chat tư vấn xe VinFast (bản giả lập, độc lập với site chính thức). Kiến trúc **LangGraph**: lớp **guardrail** lọc input (SENSITIVE / COMPETITOR / OFF_TOPIC / PASS), lớp **reasoning** gọi LLM chính kèm tool **Tavily search** (tối đa 2 lần/lượt). Trả lời dạng JSON với `answer`, `confidence`, `source_url`, `suggest_human`; UI hiện card **Gọi tư vấn viên** khi `confidence` dưới ngưỡng **hoặc** `suggest_human` true. Hỗ trợ đa phiên chat, nén bộ nhớ ngắn (tóm tắt sau vài lượt), form thu lead, phản hồi 👍/👎.

## Level: Mock prototype
- **UI:** Streamlit — `app.py` (chat), `main.py` (điều hướng multipage tới Chat + Admin).
- **Core:** LangGraph trong `engine.py`, LLM & prompt trong `constants.py`, search thực qua Tavily trong `search.py` (cần API key trong secrets/env).

## Links
- **Prototype (Link repo demo):** https://github.com/quaghien/Day06_VinFast_Demo ( Chạy local: `streamlit run app.py` từ thư mục `demo/`)
- **Knowledge / ngữ cảnh:** System prompt & guardrail trong `constants.py`; không có RAG vector DB trong bản này.
- **Video demo (backup):** https://drive.google.com/file/d/1EgSRLUJHFSdCQl3Z53rwmaR6pkgSmxTv/view?usp=sharing

## Tools
- **Frontend:** Streamlit
- **AI:** OpenAI API (model cấu hình trong `constants.py`, ví dụ guardrail mini + reasoning chính), LangChain / LangGraph
- **Search:** Tavily API (`search.py`)
- **Khác:** `requests`, tùy chọn LangSmith tracing (biến môi trường / secrets)

Tính năng chính
- **Guardrails:** Chặn & trả lời cố định theo nhóm SENSITIVE, COMPETITOR, OFF_TOPIC.
- **Reasoning + tool:** Gọi `search_web_tool` (Tavily, query prefix `VinFast`), phản hồi có `source_url`.
- **Độ tin cậy & tư vấn viên:** Điểm `confidence` + cờ `suggest_human` (OR) → card mời CSKH.
- **Đa session:** Sidebar đổi hội thoại; `memory_cache` + tóm tắt bằng LLM mini.
- **Booking:** Form ghi lead (append log).
- **Feedback:** 👍 / 👎 → `training_data.jsonl` (flywheel).
- **Admin:** Trang tổng quan + export JSONL (`admin.py`).

## Phân công
| Thành viên | Phần | Output |
|-----------|------|--------|
| Dương | UI/UX & luồng người dùng | `demo/app.py`, `demo/main.py`; hướng dẫn chạy trong `demo/README.md` |
| Hiển | AI Engine & System design | `demo/engine.py`, `demo/constants.py`; `demo/docs/langgraph-workflow.md` |
| Hiền | Tìm kiếm & tích hợp API | `demo/search.py`, `demo/requirements.txt` (Tavily / LangChain / Streamlit) |
| Hậu | Lưu trữ logs cho training / flywheel | `demo/logger.py`, `demo/data/training_data.jsonl`; `demo/docs/data_flywheel_memory.md` |
| An | Dashboard Admin & kịch bản kiểm thử | `demo/admin.py`, `demo/docs/test_cases.md` |
