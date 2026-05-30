# Ví dụ 01 — NestJS Chatbot (bảo toàn verbatim + trích xuất hidden requirement)

## Input

```
Tôi muốn xây chatbot cho hệ thống hỗ trợ khách hàng với NestJS, cần handle 100k user đồng thời,
dùng Redis để cache session, có stream token. Hạ tầng phải rẻ thôi, và cần response technical ngắn gọn.
```

---

## ❌ Không dùng Cave Prompt — Gửi thẳng prompt thô vào LLM

**LLM thường làm gì:**

Model nhận prompt và cố gắng trả lời ngay. Không có cấu trúc tường minh, nó phải đoán:

- "Rẻ" nghĩa là serverless? shared hosting? spot instance? → **đoán mò**
- "Stream token" là SSE hay WebSocket? → **chọn đại một cái**
- "100k user đồng thời" — người dùng muốn tư vấn kiến trúc hay code thật? → **thường cho cả hai, lãng phí token**
- "Tôi muốn" là filler — model vẫn xử lý như signal → **tốn attention budget vô ích**

**Kết quả bạn thường nhận được:**

> "Để xây chatbot NestJS hỗ trợ 100k user đồng thời, bạn cần chú ý những điều sau:
> Đầu tiên, cài NestJS bằng lệnh `npm install -g @nestjs/cli` và khởi tạo project..."

Vấn đề:
- Bắt đầu bằng hướng dẫn cài NestJS cơ bản → hoàn toàn không liên quan, lãng phí token output
- Không đề cập backpressure handling → hidden requirement bị bỏ sót hoàn toàn
- "Stream token" được hiểu là WebSocket, không nhắc đến SSE hay trade-off
- Diễn đạt lại prompt theo cách khác → nhận được cấu trúc và ưu tiên hoàn toàn khác

**Kiểm tra tính nhất quán — cùng intent, viết khác đi:**

| Biến thể prompt | LLM tập trung vào |
|---|---|
| "Tôi muốn xây chatbot NestJS..." | Bắt đầu bằng hướng dẫn setup |
| "Xây chatbot NestJS scale 100k user..." | Nhảy thẳng vào kiến trúc |
| "NestJS chatbot 100k concurrent, Redis, rẻ" | Trả về bullet list súc tích |

→ **3 cấu trúc khác nhau. 3 mức độ ưu tiên khác nhau. Không ai biết cái nào đúng.**

---

## ✅ Dùng Cave Prompt — Biên dịch trước khi thực thi

Cave Prompt đọc prompt và **hiện thức hoá sự hiểu biết của nó** thành một IR có cấu trúc trước khi bất cứ thứ gì chạy.

**Những gì nó phát hiện mà prompt thô bỏ sót:**

| Tín hiệu | Prompt thô | Cave Prompt |
|---|---|---|
| "Tôi muốn", "thôi" | xử lý như nội dung | gắn flag `redundant_spans`, loại bỏ |
| "stream token" | mơ hồ → đoán | ghi vào `ambiguities`, giữ nguyên văn |
| stateless session requirement | không đề cập → bị sót | trích xuất vào `hidden_requirements` |
| backpressure handling | không đề cập → bị sót | trích xuất vào `hidden_requirements` |
| "rẻ" | mơ hồ → đoán | ánh xạ thành `cost` constraint, `priorities: cost efficiency` |

**Fidelity score: 0.95** — 95% ý nghĩa gốc được bảo toàn và có thể kiểm chứng.

**Các span được bảo vệ verbatim:** `100k`, `Redis`, `NestJS`, `stream token` — không bao giờ bị paraphrase.

**Execution prompt gửi cho LLM chính:**

> Thiết kế và hướng dẫn implement chatbot hỗ trợ khách hàng trên NestJS chịu tải 100k concurrent users. Yêu cầu cứng: cache session bằng Redis, hỗ trợ stream token (SSE hoặc WebSocket) với backpressure handling, hạ tầng cost-optimized. Ưu tiên output: scalability architecture, Redis session strategy, streaming implementation. Bỏ qua intro cơ bản NestJS. Format: technical, súc tích.

Không filler. Không đoán mò. Không hướng dẫn boilerplate. LLM chính nhận được một brief chính xác.

---

## Envelope đầy đủ (machine-readable)

```json
{
  "blocking_ambiguities": [],
  "semantic_analysis": {
    "intent": "Xây dựng chatbot hỗ trợ khách hàng trên NestJS chịu tải cao, có streaming và Redis cache",
    "domain": "Backend engineering / conversational AI",
    "entities": ["NestJS", "Redis", "chatbot", "khách hàng"],
    "constraints": {
      "technical": ["NestJS framework", "Redis session cache", "token streaming"],
      "performance": ["100k concurrent users"],
      "cost": ["hạ tầng rẻ"]
    },
    "priorities": ["concurrency scalability", "cost efficiency", "streaming UX"],
    "response_preferences": {
      "tone": "technical",
      "verbosity": "concise"
    },
    "ambiguities": ["LLM provider chưa chỉ định", "database/storage chưa rõ"],
    "hidden_requirements": [
      "stateless or externalized session management",
      "backpressure handling for token streams",
      "horizontal scaling capability"
    ]
  },
  "optimized_ir": {
    "task_type": "system design + implementation guide",
    "execution_requirements": [
      "NestJS architecture for 100k concurrent users",
      "Redis session caching strategy",
      "SSE or WebSocket token streaming with backpressure",
      "cost-optimized infrastructure recommendations"
    ],
    "context_priority": {
      "high": ["scalability architecture", "Redis integration", "streaming implementation"],
      "low": ["basic NestJS setup", "boilerplate code", "introductory explanations"]
    },
    "reasoning_mode": "technical depth with concise output",
    "tool_requirements": []
  },
  "entropy_analysis": {
    "semantic_density": 0.82,
    "redundant_spans": ["Tôi muốn", "thôi"],
    "low_information_spans": ["cần response technical ngắn gọn"],
    "execution_critical_spans": ["100k", "Redis", "NestJS", "stream token"],
    "summary": "Mật độ ngữ nghĩa cao (0.82). Noise: filler đầu câu và qualifier cuối. Tất cả span kỹ thuật đều execution-critical và được giữ nguyên văn."
  },
  "verbatim_spans": ["100k", "Redis", "NestJS", "stream token"],
  "fidelity_score": 0.95,
  "dropped_or_uncertain": [
    "ngắn gọn — được hiểu là technical depth súc tích, không phải cắt ngắn đến mức mất thông tin"
  ],
  "execution_prompt": "Thiết kế và hướng dẫn implement chatbot hỗ trợ khách hàng trên NestJS chịu tải 100k concurrent users. Yêu cầu cứng: cache session bằng Redis, hỗ trợ stream token (SSE hoặc WebSocket) với backpressure handling, hạ tầng cost-optimized. Ưu tiên output: scalability architecture, Redis session strategy, streaming implementation. Bỏ qua intro cơ bản NestJS. Format: technical, súc tích."
}
```

---

*Minh hoạ: output_language=match-input (input tiếng Việt → output tiếng Việt), bảo toàn verbatim "100k", "Redis", "NestJS", "stream token", trích xuất hidden requirement (backpressure, stateless session), loại bỏ filler.*
