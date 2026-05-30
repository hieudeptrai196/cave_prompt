# Ví dụ 02 — Ambiguity chặn (dừng và hỏi lại)

## Input

```
Làm cho tôi cái app.
```

---

## ❌ Không dùng Cave Prompt — Gửi thẳng prompt thô vào LLM

**LLM thường làm gì:**

Model nhận yêu cầu gần như trống rỗng và có hai lựa chọn: hỏi lại, hoặc tự giả định rồi chạy tiếp. Hầu hết LLM chọn **tự giả định** để trông có vẻ hữu ích.

**Phản hồi LLM điển hình:**

> "Tôi sẽ giúp bạn tạo một ứng dụng web đơn giản! Đây là ứng dụng Todo List dùng React:
>
> ```javascript
> import React, { useState } from 'react';
> function App() { ... }
> ```
> ..."

**Chuyện gì đã sai:**

- Bạn hỏi "cái app" — bạn nhận được React todo list. Đó có phải thứ bạn muốn không? **Không ai biết.**
- LLM đã tự quyết định: React (không phải Vue/native), web (không phải mobile/desktop), todo list (không phải ecommerce/dashboard), JavaScript (không phải Python/Swift)
- Bạn phát hiện ra sự lệch pha **sau khi** đọc xong 200 dòng code được sinh ra
- Bạn prompt lại, chạy lại, lãng phí token và thời gian
- Trong pipeline hoặc hệ thống agent: **app sai được build, các bước tiếp theo fail ngầm**

**Vấn đề cốt lõi:**

Semantic density = **0.05**. Gần như không có thông tin nào trong prompt này. Một lần gọi LLM thô không thể compile "Làm cho tôi cái app" thành bất cứ thứ gì có nghĩa — nó tự lấp đầy mọi khoảng trống bằng những giả định bạn chưa bao giờ đồng ý.

---

## ✅ Dùng Cave Prompt — Ambiguity chặn được phát hiện trước khi thực thi

Cave Prompt phát hiện prompt không thể compile an toàn. Thay vì đoán mò, nó **dừng lại và hỏi** — trước khi một token nào bị lãng phí vào thứ sai.

**Output trên CLI:**

```
$ cave compile "Làm cho tôi cái app."

Blocking ambiguity — clarify before compiling:
  - App này dành cho nền tảng nào? (web, mobile iOS/Android, desktop, CLI?)
  - Chức năng chính của app là gì? (e.g. quản lý task, ecommerce, social, tool nội bộ...)
  - Stack công nghệ ưu tiên là gì, hoặc để tự chọn?

[exit code 2]
```

**Tại sao điều này quan trọng:**

| | Prompt thô | Cave Prompt |
|---|---|---|
| Phát hiện ambiguity | ✗ (chạy tiếp với giả định) | ✅ (dừng, hỏi lại) |
| Lãng phí token vào output sai | ✅ có | ✗ không sinh gì cả |
| Làm rõ giả định tường minh | ✗ | ✅ |
| Tín hiệu exit thân thiện với pipeline | ✗ | ✅ exit code 2 |
| Bắt buộc làm rõ ngay từ đầu | ✗ | ✅ |

**Exit code 2** là cố ý — báo hiệu "ambiguity, không phải lỗi hệ thống". Pipeline có thể rẽ nhánh theo đó:
```python
if exit_code == 2:
    ask_user_for_clarification()
elif exit_code == 1:
    raise SystemAlert()
```

Sau khi người dùng trả lời ba câu hỏi, Cave Prompt compile được một execution prompt sạch, chính xác — và LLM chính nhận brief thay vì một mớ giả định.

---

## Envelope đầy đủ (machine-readable)

```json
{
  "blocking_ambiguities": [
    "App này dành cho nền tảng nào? (web, mobile iOS/Android, desktop, CLI?)",
    "Chức năng chính của app là gì? (e.g. quản lý task, ecommerce, social, tool nội bộ...)",
    "Stack công nghệ ưu tiên là gì, hoặc để tự chọn?"
  ],
  "semantic_analysis": {
    "intent": "",
    "domain": "",
    "entities": [],
    "constraints": {},
    "priorities": [],
    "response_preferences": {},
    "ambiguities": [
      "Nền tảng chưa rõ",
      "Chức năng chưa rõ",
      "Tech stack chưa rõ"
    ],
    "hidden_requirements": []
  },
  "optimized_ir": {
    "task_type": "",
    "execution_requirements": [],
    "context_priority": {},
    "reasoning_mode": "",
    "tool_requirements": []
  },
  "entropy_analysis": {
    "semantic_density": 0.05,
    "redundant_spans": [],
    "low_information_spans": ["Làm cho tôi cái app"],
    "execution_critical_spans": [],
    "summary": "Mật độ ngữ nghĩa cực thấp (0.05). Toàn bộ nội dung là yêu cầu mơ hồ không thể compile."
  },
  "verbatim_spans": [],
  "fidelity_score": 0.0,
  "dropped_or_uncertain": [],
  "execution_prompt": ""
}
```

---

*Minh hoạ: chính sách blocking ambiguity — dừng và hỏi lại, không sinh execution prompt, exit code 2. Một lần gọi LLM thô sẽ tự giả định và sinh ra output sai.*
