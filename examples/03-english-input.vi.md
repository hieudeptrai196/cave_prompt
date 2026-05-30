# Ví dụ 03 — Input có code (khớp ngôn ngữ + bảo toàn verbatim code theo A2)

## Input

```
Tôi cần refactor hàm Python này để hiệu quả hơn. Hiện tại đang là O(n²) và
cần xử lý được tới 1M bản ghi. Đây là code hiện tại:

```python
def find_duplicates(items):
    duplicates = []
    for i in range(len(items)):
        for j in range(i + 1, len(items)):
            if items[i] == items[j] and items[i] not in duplicates:
                duplicates.append(items[i])
    return duplicates
```

Cần giữ nguyên pure Python (không dùng numpy), chạy trên Python 3.10+,
và thứ tự output không quan trọng.
```

---

## ❌ Không dùng Cave Prompt — Gửi thẳng prompt thô vào LLM

**LLM thường làm gì:**

Model đọc prompt và thử refactor. Hầu hết lần đều gần đúng — nhưng các constraint tinh tế là nơi nó thất bại.

**Những lỗi phổ biến khi gửi prompt thô:**

**1. Vi phạm constraint tường minh:**
```python
# LLM "nhiệt tình" import numpy để tối ưu hiệu năng
import numpy as np
def find_duplicates(items):
    return list(np.unique(...))
```
→ Constraint "pure Python (không dùng numpy)" có trong prompt — nhưng nằm trong prose, dễ bị bỏ qua.

**2. Tự ý đổi tên hàm hoặc signature:**
```python
# LLM đổi tên cho "rõ hơn"
def get_duplicate_values(lst):
    ...
```
→ Nếu hàm này được gọi ở 200 chỗ trong code production, mọi chỗ đều bị break. Tên gốc `find_duplicates` phải được bảo vệ verbatim.

**3. Thay đổi semantics mà không nói:**
```python
# LLM trả về set thay vì list
def find_duplicates(items):
    seen = set()
    return list({x for x in items if items.count(x) > 1})
```
→ Kiểu trả về bị thay đổi. Các caller đang expect list sẽ bị break. "Thứ tự output không quan trọng" ≠ "đổi kiểu trả về".

**4. Không có fidelity signal — bạn không biết cái gì đã bị bỏ:**
LLM cho bạn code. Bạn không biết:
- "Python 3.10+" có được đảm bảo không? (có dùng walrus operator? match statement?)
- Độ phức tạp có thực sự giảm từ O(n²) xuống O(n) không, hay chỉ được nói vậy?
- Semantics của `items[i] not in duplicates` có được giữ nguyên không?

---

## ✅ Dùng Cave Prompt — Constraint được trích xuất, code được bảo vệ verbatim

Cave Prompt đọc prompt trước khi LLM chạm vào. Nó **khoá chặt mọi constraint** và **copy nguyên xi code block** để LLM chính nhận được một brief chính xác, không còn chỗ cho sự sáng tạo tuỳ tiện.

**Những gì Cave Prompt khoá lại:**

| Constraint | Trong prompt thô | Cave Prompt xử lý |
|---|---|---|
| `pure Python (không dùng numpy)` | Trong prose, dễ bị bỏ qua | Trích xuất vào `constraints.technical` |
| `Python 3.10+` | Trong prose | Trích xuất, bảo vệ verbatim |
| `1M bản ghi` | Trong prose | Trích xuất vào `constraints.performance`, bảo vệ verbatim |
| `O(n²)` | Trong prose | Trích xuất vào `execution_requirements`, bảo vệ verbatim |
| Tên hàm `find_duplicates` | Trong code | Bảo vệ verbatim — không bao giờ đổi tên |
| Toàn bộ code block | Trong prompt | Copy nguyên xi vào execution prompt — không bao giờ paraphrase |
| `thứ tự output không quan trọng` | Trong prose | Ánh xạ thành constraint, KHÔNG hiểu là "đổi kiểu trả về" |

**Fidelity score: 0.96** — những gì bị bỏ và lý do được liệt kê tường minh trong `dropped_or_uncertain`.

**Hidden requirement được phát hiện:**
- "preserve semantics: trả về list các giá trị xuất hiện nhiều hơn một lần, mỗi giá trị một lần" — điều này được ngụ ý trong code gốc nhưng không được nói ra. Cave Prompt đọc code và trích xuất ra.

**Execution prompt gửi cho LLM chính:**

> Refactor hàm Python sau để giảm độ phức tạp từ O(n²) xuống O(n). Constraint: pure Python only (không numpy), Python 3.10+, xử lý 1M bản ghi, thứ tự output không quan trọng. Giữ nguyên semantics: trả về list các giá trị xuất hiện nhiều hơn một lần, mỗi giá trị một lần.
>
> ```python
> def find_duplicates(items):
>     duplicates = []
>     for i in range(len(items)):
>         for j in range(i + 1, len(items)):
>             if items[i] == items[j] and items[i] not in duplicates:
>                 duplicates.append(items[i])
>     return duplicates
> ```
>
> Cung cấp hàm đã refactor kèm giải thích ngắn gọn về thay đổi thuật toán.

LLM chính giờ không còn ambiguity. Mọi constraint đều tường minh. Code là nguyên bản. Không có prose nào để hiểu sai.

---

## Envelope đầy đủ (machine-readable)

```json
{
  "blocking_ambiguities": [],
  "semantic_analysis": {
    "intent": "Refactor hàm tìm duplicate từ O(n²) thành thuật toán hiệu quả hơn, xử lý 1M bản ghi",
    "domain": "Python algorithm optimization",
    "entities": ["find_duplicates", "items", "duplicates"],
    "constraints": {
      "technical": ["pure Python", "Python 3.10+", "no numpy"],
      "performance": ["xử lý tới 1M bản ghi", "giảm từ O(n²)"],
      "business": ["thứ tự output không quan trọng"]
    },
    "priorities": ["giảm độ phức tạp thời gian", "hiệu quả bộ nhớ ở quy mô lớn", "chỉ dùng standard library"],
    "response_preferences": {
      "tone": "technical",
      "verbosity": "focused"
    },
    "ambiguities": ["memory budget chưa chỉ định — giả định không giới hạn cho set-based approach"],
    "hidden_requirements": [
      "giữ nguyên semantics: trả về list các giá trị xuất hiện nhiều hơn một lần",
      "mỗi giá trị duplicate chỉ xuất hiện một lần trong kết quả (code hiện tại đảm bảo điều này)"
    ]
  },
  "optimized_ir": {
    "task_type": "code refactoring / algorithm optimization",
    "execution_requirements": [
      "thay vòng lặp lồng O(n²) bằng thuật toán set-based O(n)",
      "xử lý 1M bản ghi không dùng numpy",
      "tương thích Python 3.10+",
      "output: list các giá trị duplicate (thứ tự không quan trọng)"
    ],
    "context_priority": {
      "high": ["giảm độ phức tạp thuật toán", "giữ đúng correctness", "scale đến 1M bản ghi"],
      "low": ["style preferences", "docstring format"]
    },
    "reasoning_mode": "phân tích thuật toán + code thay thế drop-in",
    "tool_requirements": []
  },
  "entropy_analysis": {
    "semantic_density": 0.78,
    "redundant_spans": ["Tôi cần", "hiện tại", "Đây là code hiện tại"],
    "low_information_spans": ["hiệu quả hơn", "tới"],
    "execution_critical_spans": ["O(n²)", "1M bản ghi", "pure Python", "Python 3.10+", "find_duplicates", "items[i] == items[j]", "items[i] not in duplicates"],
    "summary": "Mật độ ngữ nghĩa tốt (0.78). Code block hoàn toàn execution-critical và được copy nguyên xi. Noise nhỏ trong các câu dẫn dắt."
  },
  "verbatim_spans": [
    "O(n²)",
    "1M bản ghi",
    "pure Python",
    "Python 3.10+",
    "def find_duplicates(items):\n    duplicates = []\n    for i in range(len(items)):\n        for j in range(i + 1, len(items)):\n            if items[i] == items[j] and items[i] not in duplicates:\n                duplicates.append(items[i])\n    return duplicates"
  ],
  "fidelity_score": 0.96,
  "dropped_or_uncertain": [
    "memory budget — chưa chỉ định, giả định không giới hạn"
  ],
  "execution_prompt": "Refactor hàm Python sau để giảm độ phức tạp từ O(n²) xuống O(n). Constraint: pure Python only (không numpy), Python 3.10+, xử lý 1M bản ghi, thứ tự output không quan trọng. Giữ nguyên semantics: trả về list các giá trị xuất hiện nhiều hơn một lần, mỗi giá trị một lần.\n\nCode hiện tại:\n\n```python\ndef find_duplicates(items):\n    duplicates = []\n    for i in range(len(items)):\n        for j in range(i + 1, len(items)):\n            if items[i] == items[j] and items[i] not in duplicates:\n                duplicates.append(items[i])\n    return duplicates\n```\n\nCung cấp hàm đã refactor kèm giải thích ngắn gọn về thay đổi thuật toán."
}
```

---

*Minh hoạ: output_language=match-input (input tiếng Việt → output tiếng Việt), bảo toàn verbatim A2 code block và các span kỹ thuật ("O(n²)", "1M bản ghi", "Python 3.10+"), trích xuất hidden semantic từ code, nén surgical các câu dẫn dắt.*
