# Plugin Veilus cho Claude Code

Bạn nói với Claude một việc cần làm trên một website, Claude tự làm trọn trong Veilus qua MCP:

- nhập proxy;
- tạo hồ sơ có múi giờ và ngôn ngữ khớp từng proxy;
- học site trên một hồ sơ thật;
- viết và chạy thử script Playwright;
- lên lịch;
- chạy và báo lại kết quả.

Bạn vẫn giữ quyền ở ba chỗ:

1. Bạn **duyệt script** trong app Veilus trước khi nó được chạy không người trông.
2. Bạn **đồng ý kế hoạch lịch** trước khi lịch được tạo.
3. Bạn **duyệt lại** mỗi khi script phải sửa.

English: [README.md](README.md).

## Cần có

- **App Veilus bản mới hơn 0.2.1** (bản 0.2.1 chưa có các tool chiến dịch mà skill dùng), đang chạy, dùng **gói trả phí hoặc bản dùng thử**. Gói Free không có API.
- **Claude Code**: https://claude.com/claude-code.
- Proxy, nếu việc cần. Nên dùng proxy tĩnh hoặc sticky, **ghim theo bang hoặc thành phố** chứ không chỉ theo quốc gia (xem [Proxy và múi giờ](#proxy-và-múi-giờ)).

## Cài đặt

### 1. Bật API trong Veilus

1. Mở Veilus → thanh bên, chọn **API & MCP**.
2. Bấm **Bật cổng**.
3. Ở mục **Token**, đặt tên (ví dụ `claude-code`) rồi bấm **Tạo token**. **Chép token ngay**, vì nó chỉ hiện một lần.

### 2. Nối Claude Code với Veilus (MCP)

Cũng trên trang đó, ở mục **Kết nối LLM → Claude Code**, bấm **Sao chép**. Chạy lệnh vừa chép trong terminal, **nhớ thêm `--scope user` ngay sau `add`**. Có `--scope user` thì Claude Code thấy Veilus ở mọi thư mục; không có thì chỉ thấy ở đúng thư mục bạn đứng lúc chạy lệnh:

```bash
claude mcp add --scope user veilus -e VEILUS_API_TOKEN=<token của bạn> -- "<đường dẫn tới Veilus>" mcp
```

Nếu lệnh báo `veilus` đã tồn tại, chạy `claude mcp remove veilus` trước rồi chạy lại.

Kiểm tra:

```bash
claude mcp list
```

Phải thấy `veilus` ở trạng thái đã kết nối. Mỗi khi Claude dùng Veilus thì app Veilus phải đang chạy.

> Dùng Cursor hoặc Claude Desktop? Chép khối JSON ở mục **Cursor / Claude Desktop (JSON)** vào phần cài đặt MCP của app đó. Các skill dưới đây dành cho Claude Code.

### 3. Cài plugin

```bash
claude plugin marketplace add veilus/claude-plugin
claude plugin install veilus@veilus
```

Hoặc gõ trong Claude Code: `/plugin marketplace add veilus/claude-plugin`, rồi `/plugin install veilus@veilus`.

Cập nhật sau này: `claude plugin marketplace update veilus`.

Khởi động lại Claude Code, gõ `/veilus:` là thấy bốn skill.

### 4. Dùng thử

Trong Claude Code, gõ:

```
/veilus:campaign
```

Hoặc chỉ cần kể việc bạn muốn, Claude sẽ tự chọn skill:

```
Dùng pool proxy "Singapore" của tôi, tạo 3 hồ sơ, sáng nào 9:00 cũng mở
https://example.com và in tiêu đề trang. Báo tôi lượt chạy đầu thế nào.
```

## Các skill

| Skill | Làm gì |
|---|---|
| `/veilus:campaign` | Trọn việc: hỏi những gì còn thiếu, chuẩn bị proxy và hồ sơ, rồi dùng ba skill dưới. |
| `/veilus:script` | Học site trên hồ sơ thật, viết script, chạy thử trên tối đa 3 hồ sơ, sửa, rồi nhờ bạn duyệt. |
| `/veilus:schedule` | Tính lịch cho vừa sức máy, trình bạn bảng kế hoạch, tạo lịch khi bạn đồng ý. |
| `/veilus:run` | Chạy ngay, theo dõi, giải thích lỗi từng hồ sơ, sửa và nhờ duyệt lại nếu cần, rồi báo cáo. |

Mỗi skill dùng riêng cũng được. Ví dụ "xem đêm qua chạy thế nào" sẽ dùng `run`.

## Duyệt script

Khi Claude báo script đã sẵn sàng:

1. Mở Veilus → thanh bên **Veilus Flow** và mở script đó. Script do Claude viết có nhãn cho biết nó đến từ MCP.
2. Đọc mã nguồn.
3. Bấm **Duyệt script này**.
4. Báo lại cho Claude.

Vì sao có bước này: script đã duyệt sẽ chạy theo lịch mà không ai trông, trên hồ sơ và tài khoản của bạn. Claude chỉ được chạy thử script chưa duyệt trên tối đa 3 hồ sơ, và chỉ bạn mới duyệt được. Mỗi lần lưu bản mới là phải duyệt lại.

## Câu lệnh mẫu

- "Nhập các proxy này, thử chúng và cho tôi biết cái nào chết:" rồi dán danh sách proxy.
- "Tạo 10 hồ sơ Windows trên pool US của tôi, gắn thẻ `khuyen-mai`."
- "Viết script đăng nhập bằng biến `EMAIL` và `PASSWORD` của từng hồ sơ rồi kiểm tra trang dashboard mở được. Chưa lên lịch."
- "Lên lịch cho script đã duyệt `Kiểm tra hằng ngày` trên mọi hồ sơ thẻ `khuyen-mai`, 8:30 mỗi sáng, 2 hồ sơ cùng lúc."
- "Chạy `Kiểm tra hằng ngày` ngay bây giờ và cho tôi biết hồ sơ nào hỏng, vì sao."

## Proxy và múi giờ

Mỗi hồ sơ nhận múi giờ theo IP thoát của proxy ngay lúc được tạo. Khi mở hồ sơ, Veilus kiểm lại xem proxy còn thoát ở đúng múi giờ đó không. Nếu lệch thì Veilus chặn không mở, vì lệch múi giờ là dấu hiệu dễ bị website phát hiện.

Proxy chỉ ghim theo quốc gia có thể nhảy giữa các thành phố khác múi giờ, như New York và Chicago, hay Sydney và Perth. Veilus sẽ cảnh báo khi một pool trải nhiều múi giờ. Cách sửa: ghim theo **bang hoặc thành phố** ở nhà cung cấp proxy, để mọi proxy trong pool cùng một múi giờ.

## An toàn

- Claude **không xoá được** gì trong Veilus qua MCP.
- Muốn thay proxy của hồ sơ đã có thì phải xác nhận rõ, và Claude sẽ hỏi bạn trước.
- Claude không bao giờ in mật khẩu proxy hay token.
- Chạy mã trên trang (`evaluate_js`) có đủ quyền của trang, giống một đoạn mã bạn chạy trong console trình duyệt. Claude chỉ dùng nó để đọc trang, không dùng để thao tác trên tài khoản của bạn.
- API giới hạn 30 lời gọi nặng mỗi phút cho mỗi token.
- Thu hồi token bất cứ lúc nào: **API & MCP → Token → Thu hồi**.

## Xử lý sự cố

| Vấn đề | Cách xử lý |
|---|---|
| `claude mcp list` không có `veilus`, hoặc Claude không thấy tool | Veilus mới chỉ được thêm cho một thư mục khác. Thêm lại với `--scope user` (bước 2 phần cài đặt), rồi khởi động lại Claude Code. |
| Không kết nối được tới Veilus | Veilus chưa chạy, hoặc cổng đang tắt: vào **API & MCP → Bật cổng**. |
| Lỗi `401` hoặc token bị từ chối | Token đã bị thu hồi hoặc chép sai. Tạo token mới rồi chạy lại lệnh nối với token đó. |
| "API access requires a paid plan" | API cần gói trả phí hoặc bản dùng thử. |
| Báo "cần bạn duyệt" hoặc "not approved" | Duyệt script trong Veilus Flow (xem [Duyệt script](#duyệt-script)). |
| Không mở được hồ sơ vì lệch múi giờ | Proxy của hồ sơ đã đổi sang múi giờ khác. Ghim bang hoặc thành phố ở nhà cung cấp, hoặc dùng pool chỉ có một múi giờ. |
| Một hồ sơ không được tạo: "geo lookup failed" | Claude không biết proxy đó thoát ở đâu. Nhờ Claude thử lại pool, và thay proxy chết. |
| Lỗi `409` pool đầy khi mở hồ sơ | Quá nhiều trình duyệt mở cùng lúc. Giảm concurrency của lịch, hoặc đóng bớt hồ sơ. |
| Lỗi `429` | Chạm rate limit. Claude sẽ tự chờ rồi chạy tiếp. |
| Lịch không chạy | Máy phải bật và Veilus phải đang chạy vào giờ đó; kiểm tra lịch đang bật. |
