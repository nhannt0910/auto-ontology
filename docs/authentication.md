<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

<a id="authentication-sso-and-api-tokens"></a>

# Xác thực, SSO và token API

Auto-Ontology xác thực thông qua [Better Auth](https://github.com/better-auth/better-auth)
và kiểm tra quyền của từng lời gọi theo vai trò của người gọi. Tính năng tự đăng ký
đã bị tắt: tài khoản đầu tiên là quản trị viên khởi tạo, lấy thông tin từ
`AUTO_ONTOLOGY_ADMIN_EMAIL` / `AUTO_ONTOLOGY_ADMIN_PASSWORD` (với Helm: `adminEmail` /
`adminPassword`).

Mọi route `/api/...` đều xác định người gọi tại cùng một nơi,
[`frontend/auth/resolve-user.ts`](../frontend/auth/resolve-user.ts), theo thứ tự:

1. Cookie phiên đăng nhập của trình duyệt.
2. Token API của Auto-Ontology (`x-api-key` hoặc `Authorization: Bearer`).
3. Bearer token OAuth do Auto-Ontology cấp cho client MCP.
4. ID token SSO được một bên gọi đáng tin cậy chuyển tiếp.

## SSO

Cấu hình SSO tại **Settings → Single Sign-On** (Cài đặt → Đăng nhập một lần).
Đăng ký URI chuyển hướng sau với nhà cung cấp danh tính:

```text
<APP_URL>/api/auth/sso/callback
```


<a id="api-tokens"></a>

## Token API

Trình duyệt nhận cookie phiên đăng nhập; script, notebook và tác vụ theo lịch
sử dụng token API thay thế.

1. Mở menu người dùng ở góc trên bên phải → **API Tokens** → **New token**.
2. Đặt tên, có thể chọn thời hạn hết hiệu lực (mặc định: không bao giờ), rồi sao chép token.
   Auto-Ontology chỉ lưu giá trị băm SHA-256 nên token chỉ được hiển thị một lần.
   Nếu mất token, hãy tạo token mới.

Gửi token qua `x-api-key` (hoặc `Authorization: Bearer <token>`) trong mọi lời gọi
`/api/...`:

```sh
curl -H "x-api-key: $AUTO_ONTOLOGY_API_TOKEN" https://auto-ontology.example.com/api/terms
```


```python
import os
import requests

session = requests.Session()
session.headers["x-api-key"] = os.environ["AUTO_ONTOLOGY_API_TOKEN"]
terms = session.get("https://auto-ontology.example.com/api/terms").json()
```


Với API hỏi đáp trả kết quả dạng luồng, xem
[conversation-api.md](./conversation-api.md).

Cách token hoạt động:

- **Token hoạt động với danh tính của chủ sở hữu.** Token không có quyền riêng;
  mỗi lời gọi được kiểm tra theo vai trò của chủ sở hữu, tương tự trong trình duyệt.
  Token của người có vai trò xem chỉ thực hiện được những gì vai trò đó cho phép.
- **Mọi route yêu cầu xác thực đều chấp nhận token.** Mọi thao tác yêu cầu xác thực
  trong [`openapi/auto-ontology-api.json`](./openapi/auto-ontology-api.json)
  đều có thể sử dụng token.
- **Việc thu hồi có hiệu lực ngay.** Xóa token, hoặc xóa hay cấm chủ sở hữu token,
  thì yêu cầu tiếp theo sẽ nhận mã 401.
- **Token không thể quản lý token khác.** Việc tạo và thu hồi token cần một phiên
  đã đăng nhập, vì vậy token bị lộ không thể tự tạo token thay thế.
