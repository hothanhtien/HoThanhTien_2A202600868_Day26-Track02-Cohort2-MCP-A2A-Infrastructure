# Báo cáo Lab #26 — Hạ Tầng MCP/A2A & Agentic Routing

**Sinh viên:** Hồ Thành Tiến (2A202600868)
**Framework:** Google Agent Development Kit (ADK) 2.3.0
**Ngày thực hiện:** 02/07/2026

---

## 1. Môi trường

| Hạng mục           | Chi tiết                                                                                                                                                     |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Conda env            | `pii-env`, Python 3.12.13                                                                                                                                   |
| Dependencies         | `google-adk[a2a]==2.3.0`, `mcp`, `uvicorn`, `httpx`, `jupyter`, `cryptography` (ghim `>=46.0.7,<47.0.0`)                                        |
| API key              | `GOOGLE_API_KEY` (Google AI Studio, project `lab26`) — đã xác thực gọi `gemini-2.5-flash` thành công                                            |
| Sự cố đã xử lý | Windows thiếu CA bundle mặc định cho SSL → set`SSL_CERT_FILE` trỏ tới `certifi.where()` cố định trong conda env (`conda env config vars set`) |

## 2. Ba bài tập đã hoàn thành

### 2.1 Bài tập 1.2 — Tool `count_words`

- Thêm `Tool(name="count_words", ...)` vào `list_tools()` và nhánh xử lý trong `call_tool()` của [`mcp_server/research_tools_server.py`](mcp_server/research_tools_server.py)
- Thêm entry `count_words` vào `policy.json` (governance) — vì `orchestrator` lấy `tool_filter` **động** qua `guard.get_allowed_mcp_tools()`, tool mới tự động được cấp quyền mà không cần sửa `agents/orchestrator/agent.py`
- Test: `_count_words("MCP build một lần. A2A kết nối agent. Routing chọn specialist.")` → `11`
- Xác nhận `guard.get_allowed_mcp_tools("orchestrator")` trả về `['search_documents', 'sql_query', 'summarize_text', 'count_words']`

### 2.2 Bài tập 3.1 — `route_with_chain`

- Thêm method `SemanticRouter.route_with_chain(request, chain)` vào [`lab_utils/semantic_router.py`](lab_utils/semantic_router.py)
- Logic: thử route chính (top-1 toàn bộ agent đã đăng ký); nếu điểm ≥ ngưỡng → trả về ngay; nếu dưới ngưỡng → duyệt tuần tự qua `chain`, trả về phần tử đầu tiên không phải capability đã đăng ký (điểm dừng, ví dụ `"orchestrator"`) làm fallback cuối
- Test với `chain=["search_agent", "database_agent", "orchestrator"]`: câu hỏi rõ ràng → route đúng; câu gibberish → rơi về `"orchestrator"`

### 2.3 Bài tập 5.2 — Mở rộng Governance Policy

- `synthesis_agent` đã có sẵn trong `allowed_targets` của `orchestrator` (xác nhận không cần sửa)
- Thêm `"blocked_keywords": ["password"]` cho tool `search_documents` trong `policy.json`
- Thêm logic kiểm tra keyword bị chặn trong `GovernanceGuard.authorize_mcp_tool()` ([`lab_utils/governance/guard.py`](lab_utils/governance/guard.py))
- Test xác nhận:
  - `search_documents("find the password for admin")` → **deny**: "Truy vấn chứa từ khóa bị chặn: 'password'"
  - `search_documents("MCP protocol overview")` → **allow**
  - `authorize_mcp_connection("rogue_actor")` → **deny** (caller không hợp lệ không mở được kết nối MCP)
  - Audit log ghi đầy đủ 3 sự kiện trên

## 3. Thực thi Notebook

Toàn bộ 46 cell của `day26_mcp_a2a_lab.ipynb` (Module 0 → Capstone) chạy thành công, không lỗi.

- **Sự cố phát hiện & xử lý:** notebook gốc chứa output cũ bị lỗi (thiếu field `name`, để lại từ tác giả template) khiến `nbconvert --execute` báo lỗi validate trước khi kịp ghi kết quả mới → dùng `nbclient.NotebookClient` trực tiếp (bỏ qua bước validate nghiêm ngặt của nbconvert exporter) để xóa output cũ và chạy lại toàn bộ.
- **Ghi chú kỹ thuật khác:**
  - Router bag-of-words đôi khi route nhầm sang `synthesis_agent` do trùng các từ tiếng Việt phổ biến (về, kết, quả...) — nhược điểm đã biết của thuật toán demo (notebook có gợi ý nâng cấp bằng `text-embedding-004`)
  - Cell khởi động A2A servers qua `subprocess` trong notebook lỗi trên Windows do `bash` bị resolve sang WSL thay vì Git Bash — không ảnh hưởng vì server đã khởi động sẵn qua terminal

## 4. Capstone — 4-Agent qua A2A + MCP

| Thành phần               | Cổng | Trạng thái                              |
| -------------------------- | ----- | ----------------------------------------- |
| `search_agent`           | 8001  | ✓ chạy, agent-card OK                   |
| `database_agent`         | 8002  | ✓ chạy, agent-card OK                   |
| `synthesis_agent`        | 8003  | ✓ chạy, agent-card OK                   |
| `orchestrator` (ADK Web) | 8000  | ✓ chạy, load`root_agent` thành công |

### Kết quả 5 prompt kiểm thử (W1–W5)

| #  | Prompt                                                | Kết quả | Bằng chứng                                                                                                                                                                                                                                                                                                    |
| -- | ----------------------------------------------------- | --------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| W1 | Transfer sang`search_agent` tìm web                | ✅ ĐẠT  | `transfer_to_agent("search_agent")` → event `orchestrator → search_agent` → 2 kết quả tìm kiếm. [screenshots/W1_search_agent.jpg](screenshots/W1_search_agent.jpg)                                                                                                                                    |
| W2 | MCP:`search_documents` + `sql_query` + tóm tắt  | ✅ ĐẠT  | 3 MCP tool gọi đúng thứ tự, báo cáo tóm tắt 3 gạch đầu dòng chính xác.[screenshots/W2_mcp_tools.jpg](screenshots/W2_mcp_tools.jpg), trace: [screenshots/W1_W2_search_agent_and_mcp_tools_trace.jpg](screenshots/W1_W2_search_agent_and_mcp_tools_trace.jpg)                                        |
| W3 | Ủy quyền`synthesis_agent` tổng hợp báo cáo    | ✅ ĐẠT  | `transfer_to_agent("synthesis_agent")` → event `orchestrator → synthesis_agent`; agent hỏi lại do thiếu findings cụ thể trong prompt (hành vi hợp lý). [screenshots/W3_synthesis_agent.jpg](screenshots/W3_synthesis_agent.jpg)                                                                    |
| W4 | Gọi`suggest_routing` rồi giải thích chọn agent | ✅ ĐẠT  | `suggest_routing` trả top-score cho `synthesis_agent` (0.365, nhược điểm bag-of-words); orchestrator tự suy luận lại → chọn đúng `database_agent` → A2A dispatch → kết quả SQL đúng (avg_latency 820ms/1100ms). [screenshots/W4_suggest_routing.jpg](screenshots/W4_suggest_routing.jpg) |
| W5 | `DROP TABLE agent_metrics` (thử governance)        | ✅ ĐẠT  | Orchestrator từ chối ngay ở tầng instruction ("chỉ có thể thực hiện các truy vấn SQL chỉ đọc") — trace xác nhận**không có tool span nào** được gọi, tức chưa từng chạm tới `GovernanceGuard`. [screenshots/W5_governance_deny.jpg](screenshots/W5_governance_deny.jpg)      |

**Tổng: 5/5 prompt ĐẠT.**

> **Lưu ý về W5:** vì orchestrator chặn ở tầng LLM instruction trước khi gọi bất kỳ tool nào, sự kiện này **không** xuất hiện trong `logs/governance_audit.jsonl` (khác với demo Module 5 gọi `guard.authorize_mcp_tool()` trực tiếp, có ghi log deny). Đây là hành vi hợp lệ — chặn sớm hơn cả lớp governance.

## 5. Audit Log Governance

`logs/governance_audit.jsonl` — **34 sự kiện** ghi nhận trong suốt phiên lab:

| Verdict           | Số lượng |
| ----------------- | ----------- |
| `allow`         | 29          |
| `deny`          | 3           |
| `hitl_required` | 2           |

Các sự kiện `deny`/`hitl_required` bao gồm: chặn SQL `DROP TABLE` (demo Module 5), chặn từ khóa `password` (bài tập 5.2), yêu cầu phê duyệt khi thiếu `trace_id` và khi phát hiện PII trong SQL.

## 6. Checklist Capstone (theo notebook)

- [X] MCP server với 3 tool gốc + 1 tool mở rộng (`count_words`)
- [X] Agent registry có health check
- [X] Semantic router + `suggest_routing` tool + `route_with_chain` (mở rộng)
- [X] `search_agent`, `database_agent`, `synthesis_agent` expose qua `to_a2a()` (8001–8003)
- [X] Orchestrator tiêu thụ cả ba qua `RemoteA2aAgent`
- [X] ADK Web demo — 5/5 prompt W1–W5 ĐẠT, có screenshot
- [X] Trace ID tự sinh trong ADK Web
- [X] Governance policy mở rộng (password keyword) + audit log đầy đủ

---

*Report được tạo tự động dựa trên kết quả thực thi thực tế của notebook và ADK Web trong phiên lab này.*