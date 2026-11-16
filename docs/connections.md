<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

<a id="connections-and-credential-storage"></a>

# Kết nối và lưu trữ thông tin xác thực

Auto-Ontology đọc cấu hình cơ sở dữ liệu nguồn từ hai nơi: các kết nối được thêm
trong giao diện và biến môi trường `CONNECTION_STRINGS`. Mỗi bản triển khai nên
chỉ dùng một cách, vì hai luồng xử lý ưu tiên chúng theo thứ tự khác nhau:

- **Trả lời câu hỏi** sử dụng các kết nối do giao diện quản lý khi có ít nhất một
  kết nối; nếu không có thì dùng `CONNECTION_STRINGS`.
- **Nạp dữ liệu (ingestion)** sử dụng `CONNECTION_STRINGS` bất cứ khi nào biến này
  có giá trị; chỉ đọc các kết nối do giao diện quản lý khi biến này để trống.

<a id="connections-added-in-the-ui"></a>

## Các kết nối được thêm trong giao diện

Thêm kết nối tại **Settings → Connections** (Cài đặt → Kết nối), hoặc qua API
(`POST /api/connections/test`, sau đó `POST /api/connections`; cần quyền
`connection:manage`).

Có thể giới hạn phạm vi từng kết nối bằng danh sách schema được phép, cùng
`table_allow_regex` / `table_deny_regex`. Các bộ lọc này được áp dụng khi trích xuất
catalog. Quy tắc từ chối được ưu tiên; tên bảng không kèm schema hoặc database
và có phân biệt chữ hoa, chữ thường.

Nơi lưu thông tin xác thực phụ thuộc vào việc Vault có được cấu hình hay không:

- **Không dùng Vault.** Toàn bộ đối tượng kết nối, bao gồm mật khẩu, được chuyển
  thành JSON và lưu không mã hóa trong bản ghi database của catalog trên Postgres.
- **Có dùng Vault.** Thông tin xác thực được ghi vào Vault với khóa là tên database;
  bản ghi catalog không lưu các thông tin này. Tên database là khóa dùng để tra cứu.

## `CONNECTION_STRINGS`

Danh sách URL kết nối, phân tách bằng dấu phẩy hoặc xuống dòng, được cung cấp khi
triển khai (`.env` đối với Docker Compose, `--set connectionStrings=...` đối với Helm).
Danh sách này không chứa bộ lọc schema hay bảng. `GET /api/connections/source`
cho biết biến đã được đặt hay chưa.

Scheme của URL quyết định connector được sử dụng:

| Hệ cơ sở dữ liệu | Scheme | Thêm trong giao diện |
| --- | --- | :---: |
| PostgreSQL | `postgresql://`, `postgres://` | Có |
| MySQL | `mysql://` | Có |
| Databricks | `databricks://` | Có |
| Snowflake | `snowflake://` | Có |
| ClickHouse | `clickhouse://` | Có |
| Trino | `trino://` | Có |
| Apache Kyuubi | `kyuubi://` | Có |
| HeavyDB | `heavydb://` | Có |
| DuckDB | `duckdb://` | Không |
| SQLite | `sqlite://` | Không |

Danh sách đăng ký connector nằm tại
[`auto_ontology/connectors/registry.py`](../auto_ontology/connectors/registry.py).

## Vault

Vault mặc định bị tắt. Tính năng này chỉ được bật khi cả bốn biến sau đều được đặt:

- `VAULT_ADDR`
- `VAULT_NAMESPACE`
- `VAULT_ROLE_ID`
- `VAULT_SECRET_ID`

`VAULT_AUTH_MOUNT` và `VAULT_KV_MOUNT` là các giá trị ghi đè tùy chọn. Nếu cấu hình
chưa đầy đủ, hệ thống sẽ bỏ qua cấu hình đó và ghi cảnh báo; thông tin xác thực
sẽ được lưu dưới dạng văn bản thuần trong Postgres.
