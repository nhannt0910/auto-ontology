<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

<a id="conversation-api"></a>

# API hội thoại

Client đặt câu hỏi thông qua gateway Next.js có xác thực:

```text
POST /api/chat/completions
```


Xác thực bằng cookie phiên đăng nhập hoặc token API (xem
[authentication.md](./authentication.md)). Schema của yêu cầu và phản hồi nằm trong
[`openapi/auto-ontology-api.json`](./openapi/auto-ontology-api.json).

<a id="response-stream"></a>

## Luồng phản hồi

Phản hồi là một luồng sự kiện do máy chủ gửi (server-sent events). Dòng `data:`
của mỗi sự kiện chứa một đối tượng JSON. Luồng kết thúc bằng `data: [DONE]`,
một dòng cuối không phải JSON.

| `type` | Nội dung |
| --- | --- |
| `step` | Một bước của agent bắt đầu hoặc kết thúc (`node`, `phase` và `thought` khi kết thúc), dùng để hiển thị tiến độ. |
| `sql` | Truy vấn đã qua kiểm tra và chuẩn bị được thực thi. |
| `result` | Kết quả `answer` cuối cùng: `response` (phần diễn giải), `sql_code`, `sql_response_from_db` (các dòng dữ liệu), `thoughts` (dấu vết các bước). |
| `charts` | Biểu đồ hoặc bảng cho kết quả (`content`, `sql_response`). Được gửi sau `result` khi truy vấn đã chạy và yêu cầu có `conversation_id`. |
| `error` | Một `message` mô tả lý do xử lý thất bại. |

Mỗi lần xử lý thực hiện nhiều lời gọi model nối tiếp và thường mất hàng chục
giây, vì vậy hãy đọc luồng ngay khi dữ liệu đến:

```python
import json
import os

import requests

response = requests.post(
    "https://auto-ontology.example.com/api/chat/completions",
    headers={"x-api-key": os.environ["AUTO_ONTOLOGY_API_TOKEN"]},
    json={"question": "How many orders shipped last week?"},
    stream=True,
    timeout=900,
)
response.raise_for_status()

for line in response.iter_lines(decode_unicode=True):
    if not line or not line.startswith("data:"):
        continue
    payload = line.removeprefix("data:").strip()
    if payload == "[DONE]":
        break
    event = json.loads(payload)
    if event["type"] == "step":
        print("…", event.get("node"), event.get("phase"))
    elif event["type"] == "result":
        answer = event["answer"]
        print(answer["response"])
        print(answer["sql_code"])
    elif event["type"] == "error":
        raise RuntimeError(event["message"])
```


Công cụ `ask_question` của máy chủ MCP bao bọc cùng lời gọi này và trả về câu trả
lời, SQL cùng các dòng dữ liệu dưới dạng các trường riêng biệt; xem [mcp.md](./mcp.md).

<a id="conversations"></a>

## Hội thoại

`conversation_id` là tùy chọn:

- Bỏ qua trường này để gửi một yêu cầu đơn lẻ, không lưu trạng thái.
- Gửi UUID do client tạo để bắt đầu hoặc tiếp tục một hội thoại; dùng lại UUID đó
  cho mọi lượt trong cùng cuộc hội thoại.
- Chờ `[DONE]` trước khi gửi lượt tiếp theo. Yêu cầu chồng lấn trong cùng hội thoại
  sẽ nhận `409 Conversation in progress`.

```json
{
  "question": "What about last month?",
  "conversation_id": "d61d8aa3-d496-4fa7-97ce-4f831c162e7f"
}
```


Với một lượt hội thoại, backend tải tối đa năm lượt đã hoàn tất gần nhất
(giới hạn 12.000 ký tự), rồi viết lại câu hỏi tiếp nối phụ thuộc ngữ cảnh như
“Còn tháng Tám thì sao?” thành một câu hỏi độc lập trước khi chạy agent
chuyển văn bản thành SQL. Câu hỏi độc lập được giữ nguyên.
Chỉ câu hỏi, câu trả lời và SQL trước đó được dùng để viết lại; phần suy luận của
agent, payload biểu đồ, lỗi và các dòng kết quả thô không được đưa vào bước này.

Lịch sử hội thoại thuộc về người dùng đã xác thực. Hội thoại được tạo qua API
có thể mở trong giao diện tại `/chat?focus=<conversation_id>` đối với người dùng
sở hữu nó. Giao diện web tự quản lý ID hội thoại của mình.

<a id="trust-boundary"></a>

## Ranh giới tin cậy

Hãy gọi gateway Next.js (`APP_URL`, cổng 3000 trong Docker Compose). Gateway
xác định người dùng và chuyển tiếp header nội bộ `x-auto-ontology-user-id` tới
FastAPI (cổng 3001). FastAPI tin tưởng header đó mà không xác minh, vì vậy cần
giữ cổng 3001 trong mạng riêng. Yêu cầu gọi trực tiếp FastAPI không có
`conversation_id` sẽ không lưu trạng thái; yêu cầu có `conversation_id` cần
header danh tính đáng tin cậy nói trên.
