<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

<a id="openapi-specs"></a>

# Các đặc tả OpenAPI

Đây là các tệp được sinh tự động — không chỉnh sửa bằng tay. Mỗi tệp được lưu
trong Git để có thể review phần khác biệt trong pull request và sử dụng mà
không cần máy chủ đang chạy. CI sinh lại các tệp và báo lỗi nếu bản đã commit
không còn cập nhật.

| Tệp | Nội dung mô tả | Công cụ sinh |
| --- | --- | --- |
| [`auto-ontology-api.json`](./auto-ontology-api.json) | **API công khai.** | `pnpm openapi` (chạy từ `frontend/`) |
| [`backend.json`](./backend.json) | Ứng dụng FastAPI nội bộ của Auto-Ontology. | `uv run python -m dev_tools.generate_backend_openapi` |
| [`ingestion.json`](./ingestion.json) | Ứng dụng FastAPI nội bộ của dịch vụ nạp dữ liệu. | Như trên |

<a id="which-one-do-consumers-want"></a>

## Bên sử dụng cần tệp nào?

`auto-ontology-api.json`. Các dịch vụ Python chỉ được công bố dưới dạng ClusterIP
và chỉ truy cập được qua proxy Next.js, nên `backend.json` và `ingestion.json`
mô tả các API nội bộ mà bên ngoài cluster không thể gọi trực tiếp. Chúng tồn
tại vì `auto-ontology-api.json` được xây dựng một phần từ các tệp này.

`auto-ontology-api.json` còn chứa những thông tin mà đặc tả FastAPI không thể biết:

- `x-auto-ontology-permissions` — quyền người gọi cần có, được đọc từ lớp bọc
  `withPermission({...})` của từng route (hoặc `"public"` đối với `withPublic`),
  cùng các phản hồi 401/403 tương ứng. Cùng lớp bọc này xác định `security` cho
  từng thao tác: cơ chế `ApiToken`, hoặc `[]` với route dùng `withPublic`.
- `x-auto-ontology-source` — tệp `route.ts` phục vụ thao tác.
- `x-auto-ontology-proxied` — cho biết handler chuyển tiếp tới FastAPI hay tự trả lời.
- `x-auto-ontology-upstream` — với các route chat, đường dẫn backend có phản hồi
  được chuyển tiếp.

<a id="how-it-is-built"></a>

## Cách xây dựng đặc tả

`frontend/scripts/generate-openapi.ts` duyệt `frontend/app/api/**/route.ts`
bằng ts-morph — cơ chế định tuyến theo tệp giúp xác định đường dẫn một cách
nhất quán — rồi điền schema từ hai nguồn:

- **Các handler chuyển tiếp** (`proxyToBackend`) chuyển nguyên đường dẫn,
  nên schema được hợp nhất từ `backend.json` bằng cách đối chiếu đường dẫn
  và phương thức.
- **Các handler còn lại** — gồm handler dùng Prisma không có đối ứng FastAPI,
  và handler chat tự dựng yêu cầu gửi tới upstream — khai báo cấu trúc dữ liệu
  trong module `app/api/**/openapi.ts` nằm cùng khu vực, sử dụng zod.
  Xem quy ước tương ứng tại `frontend/types/openapi.ts`. Phần khai báo nằm cạnh
  `route.ts` thay vì bên trong nó vì script Node thông thường không thể import
  một route handler (`next/headers`, `after()`, `getPrisma()`).

Chỉ các schema thành phần mà đặc tả công khai thực sự tham chiếu mới được sao
chép, bao gồm đầy đủ các schema được tham chiếu gián tiếp.

<a id="regenerating"></a>

## Sinh lại đặc tả

```sh
uv run python -m dev_tools.generate_backend_openapi   # backend.json, ingestion.json
cd frontend && pnpm openapi                           # auto-ontology-api.json
```


Chạy theo đúng thứ tự này — đặc tả công khai đọc bản `backend.json` đã lưu trong Git.

<a id="viewing"></a>

## Xem đặc tả

Đặc tả được triển khai trên GitHub Pages tại:
<https://friendly-adventure-38857y4.pages.github.io/>
Sau này sẽ được đưa vào bộ tài liệu Auto-Ontology trên website NVIDIA.

Hoặc dán tệp vào <https://docs.scalar.com/swagger-editor> — lưu ý rằng thao tác
này cung cấp toàn bộ bề mặt API cho một bên thứ ba.

<a id="conventions"></a>

## Quy ước

**Toàn bộ API dùng snake_case** — từ các đoạn đường dẫn, tham số đường dẫn và
truy vấn, đến nội dung yêu cầu và các trường phản hồi. Endpoint mới cần tuân
theo quy ước này, bất kể cách đặt tên của phần mã phía sau:

- Thư mục route động được đặt theo tên tham số mà chúng chứa, dùng snake_case:
  `app/api/terms/[term_id]/route.ts` → `/api/terms/{term_id}`.
- Các model Prisma phía sau route chỉ có ở frontend dùng tên trường snake_case,
  nên bản ghi được chuyển thẳng thành cấu trúc API mà không cần lớp ánh xạ.
- Nếu mã phía sau route dùng camelCase và không thể thay đổi, handler thực hiện
  ánh xạ tại ranh giới API thay vì để cách đặt tên đó lọt ra ngoài: các model
  Better Auth (`User`, `Session`, `Account`, `SsoProvider`) được sinh dưới dạng
  camelCase, còn `GET /api/sso-providers` công bố `provider_id`.
  Tương tự, `GET /api/sso-discovery` chuyển tiếp tài liệu OpenID Connect
  Discovery theo tên trường chuẩn riêng của tài liệu đó.
- TypeScript nội bộ vẫn dùng camelCase thông thường cho biến cục bộ và kiểu
  dữ liệu giao diện; chỉ dữ liệu truyền qua mạng dùng snake_case.

Bộ sinh đặc tả thực thi một phần các quy ước này: nó từ chối công bố route nếu
không đọc được đối số của `withPermission(...)`, và báo lỗi với mọi `route.ts`
không export phương thức HTTP nào mà nó nhận biết, để không âm thầm bỏ sót route.

<a id="known-limitations"></a>

## Các giới hạn đã biết

- Chưa có khối `servers`: đặc tả mô tả cách xác thực nhưng chưa chỉ ra host
  cần gọi.
- Phần lớn schema có `additionalProperties: true`. Các model phản hồi backend
  được chủ động thiết kế linh hoạt — FastAPI _loại bỏ_ trường không khai báo
  khi có `response_model`, nên model chặt chẽ sẽ âm thầm làm hỏng phản hồi
  thay vì chỉ mô tả chúng. Các trường được liệt kê là đáng tin cậy, nhưng
  danh sách không được bảo đảm là đầy đủ.
- `PATCH /api/nodes/{node_id}` trả về `NodeUpdateResult | null`, với tất cả
  các trường đều tùy chọn: nội dung phản hồi chỉ chứa các thuộc tính mà yêu cầu
  đã chỉnh sửa. Route cũng chấp nhận mọi thuộc tính đó cho mọi loại node —
  `sample_values` có ý nghĩa với Column nhưng không bị từ chối khi gửi cho Database.
- `/api/auth/{all}` là route bắt mọi đường dẫn con của Better Auth. Route này
  có trong đặc tả, nhưng cấu trúc yêu cầu và phản hồi của từng endpoint thuộc
  thư viện và không được sao chép vào đây.
