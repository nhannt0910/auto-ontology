<!--
SPDX-FileCopyrightText: Copyright (c) 2026, NVIDIA CORPORATION & AFFILIATES.
All rights reserved.
SPDX-License-Identifier: Apache-2.0
-->

<a id="auto-ontology-mcp-server"></a>

# Máy chủ MCP của Auto-Ontology

Cho phép mọi agent hỗ trợ MCP — Cursor, Claude Desktop hoặc agent nội bộ — đặt
câu hỏi về dữ liệu bằng ngôn ngữ tự nhiên và kiểm tra lớp ngữ nghĩa dùng để đưa
ra câu trả lời. Mọi ứng dụng hỗ trợ [Model Context
Protocol](https://modelcontextprotocol.io) đều có thể sử dụng.

<a id="how-it-works"></a>

## Cách hoạt động

Máy chủ là một HTTP client của API công khai Auto-Ontology. Nó không giữ thông
tin xác thực database, cấu hình model hay thông tin xác thực riêng:

```
agent harness  ──MCP──▶  auto-ontology-mcp  ──HTTPS──▶  Next.js (public API)
                                                   │
                                                   ▼
                                        FastAPI, Postgres
```


Việc xác thực và kiểm tra quyền đã được thực hiện ở lớp Next.js, nên máy chủ
kế thừa cơ chế đó thay vì triển khai lại. Điều này có hai hệ quả: máy chủ có thể
chạy ở bất cứ đâu truy cập được bản triển khai, và **mỗi phiên chỉ thực hiện được
những gì người dùng của phiên đó được phép làm** — người có vai trò xem nhận
các câu trả lời trong phạm vi quyền xem.

<a id="running-it"></a>

## Chạy máy chủ

Chạy một máy chủ để người dùng đăng nhập thông qua đó. Thông tin duy nhất cần
cung cấp là URL của bản triển khai Auto-Ontology:

```sh
AUTO_ONTOLOGY_API_URL=https://auto_ontology.example.com auto-ontology-mcp
```


Cài bằng `uvx`; công cụ này tải và dựng trực tiếp từ repository, nên không cần clone:

```sh
uvx --from "git+https://github.com/NVIDIA/auto-ontology.git#subdirectory=mcp" auto-ontology-mcp
```


> [!NOTE]
> Gói MCP được cài từ GitHub. Lệnh `uvx auto-ontology-mcp` đứng riêng chỉ tìm được
> gói sau khi gói đó được phát hành lên PyPI.

<a id="connecting-a-client"></a>

## Kết nối client

Trỏ client tới URL. Cấu hình không chứa thông tin xác thực:

```json
{
  "mcpServers": {
    "auto-ontology": {
      "url": "https://auto-ontology-mcp.example/mcp"
    }
  }
}
```


Lưu ý hậu tố `/mcp`: đây là endpoint, không phải đường dẫn gốc của máy chủ.
Khởi động lại client sau khi chỉnh cấu hình, vì phần lớn client chỉ đọc cấu hình
MCP khi khởi động.

Client nhận biết máy chủ yêu cầu cấp quyền và đề nghị đăng nhập — Cursor sẽ
hiển thị trạng thái cần đăng nhập. Người dùng được đưa tới trang đăng nhập thông
thường của Auto-Ontology, sử dụng cấu hình SSO của bản triển khai nếu có.
Sau đó client giữ token và tự quản lý token này. Không cần tự tạo hay dán token;
mọi lời gọi đều chạy dưới danh tính của người đã đăng nhập.

Sau đó có thể hỏi, chẳng hạn: *“Auto-Ontology định nghĩa khách hàng đang hoạt
động như thế nào, và quý trước có bao nhiêu khách hàng như vậy?”*

<a id="sign-in"></a>

## Đăng nhập

Auto-Ontology là máy chủ cấp quyền. Máy chủ MCP này không giữ client ID,
client secret hay URI chuyển hướng, nên không có thông tin tương ứng phải cấu
hình cho từng bản triển khai hoặc đăng ký với nhà cung cấp danh tính — client
tự đăng ký với Auto-Ontology.

Thông tin đăng ký và các quyền cấp nằm trong database của Auto-Ontology, nên
các bản sao dùng chung chúng và việc khởi động lại không làm người dùng mất
phiên đăng nhập. Quyền được lấy từ tài khoản ở mỗi lời gọi thay vì từ token,
nên việc đổi vai trò, cấm người dùng hoặc xóa quyền cấp có hiệu lực ngay.

Token có dạng opaque, tức không thể suy ra nội dung danh tính trực tiếp từ token.
Máy chủ gọi endpoint OAuth UserInfo của Auto-Ontology để kiểm tra từng token.
Endpoint xác thực bằng bearer token này không cần client secret và kiểm tra
thời hạn, trạng thái thu hồi, client cùng trạng thái phiên trình duyệt ở mỗi lời gọi.
Quyền cấp phải bao gồm scope `openid` (yêu cầu của UserInfo trong Better Auth);
Auto-Ontology công bố scope này trên máy chủ cấp quyền để các client sao chép
`scopes_supported` đưa nó vào yêu cầu. Nếu không truy cập được Auto-Ontology,
lời gọi trả lỗi thay vì yêu cầu “đăng nhập lại”; nhờ đó sự cố dịch vụ không đẩy
người dùng vào một luồng đăng nhập chắc chắn thất bại.

Không có cách cấu hình token riêng cho máy chủ MCP. Nếu có, mọi người gọi sẽ
hoạt động dưới danh tính chủ sở hữu token — dùng quyền và lịch sử hội thoại
của người đó — mà giao thức không thể hiện điều này.

<a id="pointing-at-an-auto-ontology-on-your-own-machine"></a>

## Trỏ tới Auto-Ontology trên máy của bạn

Chỉ cần thay đổi `AUTO_ONTOLOGY_API_URL`. Với stack Docker, ứng dụng web chạy
ở cổng `:3000`:

```sh
AUTO_ONTOLOGY_API_URL=http://localhost:3000 auto-ontology-mcp
```


Đăng nhập yêu cầu một phiên bản Auto-Ontology đủ mới để đóng vai trò máy chủ
cấp quyền; image đã dựng trước có thể chưa đáp ứng. Nên kiểm tra trước để tránh
lỗi chỉ xuất hiện khi client không đăng nhập được:

```sh
curl -s -o /dev/null -w '%{http_code}\n' $AUTO_ONTOLOGY_API_URL/.well-known/oauth-authorization-server
```


Kết quả mong đợi là `200`. Nếu nhận mã khác, hãy chạy frontend từ checkout của
bạn bằng `pnpm dev`, rồi trỏ `AUTO_ONTOLOGY_API_URL` tới cổng frontend đó.

<a id="tools"></a>

## Công cụ

Mọi công cụ ở đây đều chỉ đọc. Không công cụ nào sửa bảng thuật ngữ, catalog
hay các database bên dưới.

| Công cụ | Mục đích |
| --- | --- |
| `ask_question` | **Công cụ chính.** Đặt câu hỏi bằng ngôn ngữ thông thường; nhận câu trả lời, SQL Auto-Ontology đã chạy và các dòng kết quả. |
| `check_answerable` | Đánh giá lớp ngữ nghĩa có đủ thông tin để trả lời câu hỏi hay không. Đây là bước kiểm tra ít tốn tài nguyên trước `ask_question`. |
| `check_readiness` | Kiểm tra bản triển khai đã đủ điều kiện để trả lời bất kỳ câu hỏi nào hay chưa. |
| `search_terms` | Tìm kiếm trong bảng thuật ngữ nghiệp vụ. |
| `get_term` | Lấy một thuật ngữ: mô tả, từ đồng nghĩa và các thuật ngữ liên quan. |
| `get_term_columns` | Lấy các cột vật lý được ánh xạ tới thuật ngữ. |
| `get_term_sql_attributes` | Lấy các thuộc tính SQL được định nghĩa trong một thuật ngữ. |
| `get_sql_attribute` | Lấy một thuộc tính SQL: biểu thức và mục đích sử dụng. |
| `describe_table` | Lấy các cột, thuật ngữ liên quan và thuộc tính SQL của một bảng trong cùng kết quả. |

Hệ thống chủ động không cung cấp công cụ duyệt database, schema hoặc các cột
thô. Bên sử dụng cần tiếp cận dữ liệu thông qua lớp ngữ nghĩa; công cụ duyệt
catalog có thể khiến agent suy luận trực tiếp từ bảng thô. `describe_table`
đáp ứng nhu cầu hợp lệ vì trả về các thuật ngữ và thuộc tính SQL có liên quan
đến bảng đó.

Máy chủ cũng công bố **hướng dẫn** trong quá trình bắt tay, mô tả thứ tự dùng
các công cụ để model không phải tự suy ra — nếu không được hướng dẫn, model
thường gọi thẳng `ask_question`.

<a id="about-ask_question"></a>

### Về `ask_question`

Công cụ này gọi `POST /api/chat/completions`, cùng endpoint mà giao diện web
sử dụng, để chạy toàn bộ agent chuyển văn bản thành SQL: nhiều lời gọi model
nối tiếp, thường mất hàng chục giây. Kết quả gồm câu trả lời, SQL và các dòng
dữ liệu dưới dạng các trường riêng biệt, thay vì Markdown như giao diện nhận
được, nên agent có thể dùng SQL mà không cần phân tích phần diễn giải.
Kết quả bị giới hạn ở 100 dòng; `row_count` và `truncated` cho biết số lượng
và việc kết quả có bị cắt bớt hay không. Công cụ gửi thông báo tiến độ MCP
trong khi agent xử lý, tương ứng với dấu vết suy luận hiển thị trên giao diện web.

<a id="about-check_readiness"></a>

### Về `check_readiness`

Để trả lời câu hỏi cần hai điều kiện độc lập: lớp ngữ nghĩa đã được biên dịch
để diễn giải câu hỏi, và kết nối database còn hoạt động để chạy SQL.
Chỉ có một trong hai là trạng thái dễ gây hiểu nhầm nhất: mọi thứ có vẻ bình
thường — đọc được bảng thuật ngữ, xác định được thuật ngữ — nhưng câu hỏi vẫn
thất bại sau nhiều phút gọi model, vừa tốn tài nguyên vừa khó nhận biết nguyên
nhân. `check_readiness` phân biệt các tình huống này từ đầu và báo đồng thời
mọi điều kiện đang cản trở.

Lưu ý rằng `databases` có thể không rỗng trong khi `can_execute_sql` là false:
catalog vẫn tồn tại sau khi kết nối dùng để nạp nó không còn hoạt động, nên có
tên database chưa chứng minh được khả năng truy vấn. Mặc định, chỉ quản trị
viên được đọc cấu hình kết nối, vì vậy người có vai trò xem nhận mã 403 ở bước
đó dù chat và catalog vẫn đọc bình thường — điều này không có nghĩa bản triển
khai bị lỗi hay thông tin xác thực sai. Trường hợp này được ghi dưới
`unverified`, và kết luận dựa trên những gì thực sự kiểm tra được.

<a id="configuration"></a>

## Cấu hình

| Biến | Mặc định | Ý nghĩa |
| --- | --- | --- |
| `AUTO_ONTOLOGY_API_URL` | `http://localhost:3000` | URL gốc của ứng dụng web Auto-Ontology. |
| `AUTO_ONTOLOGY_MCP_PUBLIC_URL` | `http://localhost:3003` | Địa chỉ gốc MCP công khai, không có `/mcp`. Frontend phải nhận cùng giá trị để token được cấp chứa đúng URL tài nguyên mà client sử dụng. |

Các biến ít khi cần dùng: `AUTO_ONTOLOGY_MCP_HOST` và `AUTO_ONTOLOGY_MCP_PORT`
(địa chỉ và cổng lắng nghe, mặc định `0.0.0.0:3003`),
`AUTO_ONTOLOGY_MCP_TIMEOUT_S` và `AUTO_ONTOLOGY_MCP_CHAT_TIMEOUT_S`
(thời gian chờ yêu cầu, lần lượt `30` và `900`), cùng
`AUTO_ONTOLOGY_OPENAPI_SPEC` (ghi đè đặc tả dùng để sinh các công cụ).

<a id="extending-the-tool-surface"></a>

## Mở rộng tập công cụ

Tập công cụ là một danh sách cho phép được khai báo rõ ràng trong
[`mcp/auto_ontology_mcp/tools.py`](../mcp/auto_ontology_mcp/tools.py).
Đặc tả công bố 87 thao tác; đưa tất cả ra ngoài sẽ làm giảm đáng kể khả năng
chọn đúng công cụ. Để công bố thêm một thao tác, thêm `ToolSpec` xác định
phương thức, đường dẫn, tên công cụ cho agent và mô tả **khi nào nên sử dụng**.
Máy chủ sẽ báo lỗi và không khởi động nếu một mục trong danh sách được chọn
không còn tồn tại trong đặc tả, tránh việc đổi tên ở upstream âm thầm làm mất
công cụ.

Các tham số và mô tả lấy từ
[`auto-ontology-api.json`](./openapi/auto-ontology-api.json), được lưu trong Git ở
hai nơi: bản chuẩn trong `docs/openapi/` và một bản trong gói, vì máy chủ đọc
đặc tả lúc khởi động và phải hoạt động cả khi bản cài đặt không có thư mục
`docs/`. `pnpm openapi` ghi cả hai bản và CI đối chiếu chúng, vì vậy không chỉnh
tay bản nằm trong gói.

<a id="troubleshooting"></a>

## Khắc phục sự cố

**Client không tìm được cách đăng nhập** — kiểm tra
`$AUTO_ONTOLOGY_API_URL/.well-known/oauth-authorization-server` có trả về JSON
hay không. Nếu bị chuyển hướng tới trang đăng nhập, bản triển khai có trước
khi route đó được bổ sung.

**Lỗi “Protected resource ... does not match expected ...”** — máy chủ công bố
địa chỉ mà nó lắng nghe, nhưng client truy cập bằng một cách viết khác của cùng
host; hầu hết trường hợp là `127.0.0.1` và `localhost`. Đặt
`AUTO_ONTOLOGY_MCP_PUBLIC_URL` thành URL client sử dụng, hoặc bỏ giá trị của
`AUTO_ONTOLOGY_MCP_HOST`.

**Lỗi “This request carried no signed-in session”** — quyền cấp đứng sau lời gọi
đã hết hạn hoặc bị thu hồi trong Auto-Ontology. Hãy đăng nhập lại từ client.

**`ask_question` trả về câu trả lời rỗng, không có SQL và không báo lỗi** — lần
xử lý đã hoàn tất nhưng bước truy xuất không tìm được dữ liệu để dựng truy vấn.
Hãy gọi `check_readiness` trước: nguyên nhân thường là chưa cấu hình kết nối
database, nên diễn đạt câu hỏi thế nào cũng không thể trả lời được. Nếu bản
triển khai đã sẵn sàng, vấn đề nằm ở từ ngữ trong câu hỏi — dùng `search_terms`
để tra các danh từ trong đó.

**Thông báo “Auto-Ontology cannot answer right now”** — thường do lớp ngữ nghĩa
chưa từng được biên dịch; xác nhận bằng `check_readiness`. Thông báo này cũng
xuất hiện khi một `conversation_id` đã có lượt đang được xử lý.

**Client không hiển thị công cụ** — kiểm tra log MCP của client. Máy chủ ghi
log ra stderr.
