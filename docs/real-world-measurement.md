# Đo lại sau một ca việc thật

Nối tiếp [console-vs-official-improvements.md](./console-vs-official-improvements.md). Doc trước viết từ quan sát hành vi; doc này viết sau khi dùng bản đã sửa để làm một việc có thật: nhân bản toàn bộ design system của một file (19 component/set) sang một page khác, mỗi cái thêm một chiều variant `Mode=Default / Mode=Wireframe`.

## 1. Khối lượng thật

| | Số đo |
|---|---|
| Node trong kết quả | 595 |
| Variant wireframe tạo ra | 48 |
| Node bên trong các variant đó | 288 |
| Lần ghi fill cần thiết | 197 |
| Lần ghi stroke cần thiết | 85 |
| **Số call thực tế** | **~25** (13 `run_script`) |
| Ước tính nếu chỉ có tool nguyên tử | ~80+, kèm việc kéo fill của 288 node về context |

Điểm mấu chốt không nằm ở số call: màu xám của mỗi node được **tính từ luminance màu gốc**. Phép tính per-node đó không biểu diễn được bằng tool nguyên tử, dù có `node_ids[]`. Đây là lý do `run_script` tồn tại.

## 2. `run_script` chạy được

Doc trước dự đoán sandbox Figma chặn `new Function`. **Sai.** Đã kiểm chứng: eval được, `await` trong thân script được, lỗi ném ra trả về sạch, trả về node sống bị chặn đúng cách mà không làm chết bridge. Toàn bộ ca việc trên chạy bằng `run_script`.

## 3. Lỗi phát hiện khi làm thật (đã sửa)

- **`combine_as_variants` xếp chồng variant lên nhau** và **COMPONENT_SET không tự nới bounds** — hai lỗi im lặng chồng nhau: variant đè nhau, phần thừa bị cắt, ảnh chụp chỉ thấy một cái. Mất 3 vòng mới tìm ra vì triệu chứng giống hệt "clone hỏng". → handler tự xếp cột và `resize()`.
- **Không nhắm được page.** Mọi tool bám `figma.currentPage`. Giữa session, current page đổi và `find_nodes` trả về node của page khác trong khi page đích đang rỗng — sai scope, không cảnh báo. → thêm `list_pages`, `set_current_page`, `page` cho `find_nodes`, và kết quả `find_nodes` tự khai scope nó vừa tìm.
- **`get_screenshot` hạ scale âm thầm.** Frame cao 9336px ra ảnh 139×2000, không đọc được gì nhưng trông như ảnh thật. → trả kèm cảnh báo, scale đã dùng, chiều cao gốc, và thêm `max_dimension`.

## 4. Không sửa

- **Id từ `create_*` biến mất một lần** (`40:309` → frame thật là `40:316`). Đã thử tái hiện: tạo frame, đọc lại ngay và sau 4s, id ổn định. **Không reproduce được**, nhiều khả năng do undo/redo trong Figma lúc script đang chạy. Ghi lại đây để lần sau gặp thì có đầu mối, chưa đủ cơ sở để sửa.

## 5. Kiểm chứng trên file thật

Chạy lại sau khi sửa, trên chính file đã dùng để đo:

| Kiểm tra | Kết quả |
|---|---|
| `list_pages` | 3 page, đánh dấu đúng page đang mở |
| `find_nodes` nhắm page không mở | `scope: "page Components"`, 5 kết quả |
| `find_nodes` không tham số | `scope: "page Page 3"` — tự khai, hết đoán |
| `set_current_page` | đổi qua lại `Components` ↔ `Page 3` |
| Tên page sai | `page not found: Nope (see list_pages)` |
| `combine_as_variants` | set `120x152` thay vì `120x60` — hết cắt variant |
| `get_screenshot` frame 9336px | kèm `scaled to 0.21x to fit; the node is 9336 px tall, so detail is lost` |
| `compact` | 39.821 → 24.092 ký tự (39%) |

## 6. Bẫy vận hành khi dev (không phải lỗi code)

Mất khá nhiều thời gian cho ba thứ không nằm trong code:

- **Figma nạp plugin từ thư mục đã import, không phải từ repo.** Sửa `plugin/code.js` trong repo mà Figma đang trỏ vào một bản copy thì không có gì thay đổi, và triệu chứng là `unknown command` — trông hệt như thiếu tính năng. Đường dẫn thật hiện ngay dưới tên plugin trong menu Development.
- **Plugin chỉ nạp code lúc mở cửa sổ.** Sửa xong phải đóng/mở lại, không có hot reload.
- **Một server cũ vẫn giữ bridge.** Server của một session Claude Code khác (build cũ hơn) chiếm port 2000, nên plugin mới nối vào nghe protocol cũ và báo mismatch. `lsof -nP -iTCP:2000 -sTCP:LISTEN` ra thủ phạm.

Cái thứ ba còn tệ hơn vì **thông báo đổ lỗi nhầm phía**: nó luôn ghi "Plugin outdated" kể cả khi server mới là bên cũ, khiến người ta đi re-import một plugin vốn đã mới. Đã sửa để nói đúng bên nào lạc hậu. Quy trình đầy đủ nằm trong [CONTRIBUTING.md](../CONTRIBUTING.md).

## 7. Còn lại

`compact` giảm 39–41% payload, không phải ~10× như doc trước kỳ vọng. Muốn giảm sâu hơn thì cần `fields` cho `get_design_context` chứ không phải tinh chỉnh thêm default.
