# figma-mcp-console vs Figma MCP official — ghi nhận từ thực chiến

Nguồn dữ liệu: một session dựng design system cho file Figma 20 frame (10 wireframe + 10 prototype)
— tạo 8 color variable, 7 component/component set, bind token, apply vào prototype.
Nửa đầu dùng MCP official (`use_figma`), nửa sau dùng `figma-mcp-console` sau khi official dính
rate limit Starter plan và yêu cầu re-auth giữa chừng.

## 1. Khác biệt cốt lõi

| | official | console |
|---|---|---|
| Mô hình | 1 tool `use_figma` chạy JS Plugin API tuỳ ý | ~44 tool nguyên tử, mỗi tool 1 node |
| Auth | OAuth, hết hạn giữa session | không cần token |
| Rate limit | Starter plan chặn sau ~30 call, mất luôn khả năng làm việc | không |
| Batch | 1 call = N thao tác | 1 call = 1 thao tác |
| Payload | tự quyết định return gì | server quyết định, luôn trả full node summary |

Số liệu thật: bind variable cho component — official **1 call / 130 paint**. Cùng việc đó với
console phải **27 call** và phải bỏ dở ~200 node text/ellipse vì không kham nổi.

## 2. Thiếu sót chặn việc (xếp theo mức độ đau)

### P0 — Không có đường thoát scripting
Mọi thao tác hàng loạt đều chết. Không có cách nào "cho tất cả node có fill #FFFFFF trong frame X
bind vào variable Y".

**Đề xuất**: thêm `run_script` — eval JS trong sandbox plugin, trả về giá trị `return`.
Đây là tool duy nhất xoá được toàn bộ khoảng cách với official. Nếu ngại rủi ro, bước đệm:
cho mọi tool mutating nhận `node_ids: string[]` thay vì `node_id: string`
(`set_bound_variable`, `apply_style`, `set_fills`, `set_strokes`, `set_corner_radius`,
`set_characters`, `set_effects`).

### P0 — Không có `rename_nodes`
Chặn workflow variant 2 lần. Phải `clone_node(name)` + `remove_nodes` để đổi tên — cách này
**phá instance** nếu component đã được dùng. Trong session đã một lần suýt hỏng, và một lần
làm mất tiền tố `Style=` của 11 component typography.

**Đề xuất**: `rename_nodes({ items: [{node_id, name}] })`. Rẻ, 20 dòng code, gỡ được nhiều ca nhất.

### P1 — Không có API component nâng cao
Thiếu: `combine_as_variants`, `add_component_property` (TEXT/BOOLEAN/INSTANCE_SWAP),
`set_instance_properties`. Hệ quả: phải đẩy việc sang người dùng 3 lần
("bạn chọn 7 component → Combine as variants"), và component không có text property —
đổi chữ phải chọn layer con thay vì sửa ở panel.

### P1 — `find_nodes` mù về style
Chỉ lọc được theo name/text/type. Không lọc được theo fill, fontSize, cornerRadius.
Hệ quả thực tế: phải **đoán vai trò node theo kích thước** (328×88 = list card, 328×40 = filter bar)
để bind màu — đúng trong file này nhưng là heuristic dễ sai.

**Đề xuất**:
- `fields: ["fills","fontSize","fontName","cornerRadius","effects"]` — trả kèm trong kết quả
- filter: `fill: "#FFFFFF"`, `font_size: 13`, `font_style: "Bold"`

### P2 — `get_design_context` quá nặng
`depth: 1` trên frame 27 con ≈ 15k token JSON: màu ở dạng float 17 chữ số,
`gradientTransform` đầy đủ, mọi field mặc định. Dùng 2–3 lần là ngập context.

**Đề xuất**: `compact: true` — màu dạng hex, bỏ field mặc định, bỏ `boundVariables` rỗng;
hoặc `fields` để chọn thứ cần.

### P2 — Kết quả tool quá dài dòng
Mỗi call mutating echo lại full node summary. Batch 50 call = 50 block JSON vô dụng.

**Đề xuất**: `verbose: false` mặc định cho mutating tool — chỉ trả `{id, ok}`.

## 3. Bẫy đã dính (bug/UX, không phải thiếu tính năng)

- **`create_frame` mặc định fill trắng.** Component `List Row` bị dải trắng sau lưng ở 7 dòng,
  chỉ phát hiện khi screenshot. → nên mặc định **transparent**, hoặc có `fill: "none"`.
- **`set_fills` không xoá được fill.** Phải lách bằng `opacity: 0`. → nhận `color: null` = xoá.
- **Không có tool đổi font của text đã tạo.** `create_text` set font một lần rồi thôi.
  Chặn hẳn việc làm variant Active cho tab bar (cần toggle Bold/Regular).
  → `set_text_properties({node_id, font_family, font_style, font_size, line_height, letter_spacing})`.
- **ID node con trong instance (`I<instance>;<child>`) không được tài liệu hoá.**
  Mình suy ra và dùng được, nhưng `create_instance` nên trả luôn map `children: {childId: instanceChildId}`.
- **`append_children` vào frame làm component rời khỏi component set và bị đổi tên**
  thành `<Set>/<Variant>`. Không có cảnh báo. → nên trả warning trong kết quả.
- **`resize_nodes` trên component không tự đẩy constraint cho con** — phải tự `move_nodes` lại
  từng child. Đúng với Figma API nhưng nên ghi rõ trong description.

## 4. Thứ console làm tốt hơn official (giữ nguyên)

- **Không rate limit, không auth** — official chết giữa session, console chạy tiếp tới cuối.
  Đây là lý do tồn tại, đừng đánh đổi.
- **`get_screenshot` nhanh, trả ảnh inline**, không cần tải URL ngắn hạn như official.
- **Multi-file bridge** — official phải truyền `fileKey` mọi call, console tự nhận diện.
- **Thao tác nguyên tử dễ debug** — khi sai chỉ sai 1 node, không hỏng cả script.

## 5. Chạm vào đâu trong repo

Mỗi tool mới = 3 chỗ, theo đúng quy ước hiện có:

- `internal/tools/tools.go` — args struct (embed `fileArg`) + `registerBridged[In]`
- `plugin/code.js` — handler cùng tên trong object `handlers`
- `bridge.ProtocolVersion` — bump nếu đổi shape message (và hằng tương ứng trong `plugin/ui.html`)

Riêng `run_script` nằm trọn trong plugin sandbox (`code.js`), không đụng tới Go server —
sandbox không có filesystem/network nên rủi ro giới hạn trong phạm vi document.
Các thay đổi còn lại (`node_ids[]`, `verbose`, `fields`, `compact`) chỉ sửa args struct +
handler, không đổi protocol.

## 6. Lộ trình gợi ý

1. `rename_nodes` + `set_text_properties` + `create_frame` transparent *(1 buổi, gỡ 3 ca chặn)*
2. `node_ids[]` cho mọi tool mutating + `verbose: false` *(giảm 70–90% số call)*
3. `fields`/filter cho `find_nodes`, `compact` cho `get_design_context`
4. `combine_as_variants` + component properties
5. `run_script` *(xoá hẳn khoảng cách với official)*

Nếu chỉ làm được một thứ: **`run_script`**. Nếu chỉ làm được một thứ rẻ: **`rename_nodes`**.

## 7. Tình trạng triển khai

Đã implement toàn bộ lộ trình 1–5 (`ProtocolVersion` 1 → 2, plugin cũ sẽ hiện cảnh báo outdated).

| Hạng mục | Trạng thái |
|---|---|
| `rename_nodes`, `set_text_properties` | xong |
| `create_frame` transparent mặc định, `fill_color: "none"` | xong |
| `set_fills`/`set_strokes` nhận `color: "none"` để xoá | xong |
| `node_ids[]` cho set_fills / set_strokes / set_corner_radius / set_effects / apply_style / set_bound_variable / set_characters / set_text_properties | xong |
| `verbose: false` mặc định (trả `{ok, ids}`) cho mọi tool mutating | xong |
| `find_nodes`: filter `fill`/`font_size`/`font_style` + `fields` | xong |
| `get_design_context`: `compact: true` | xong |
| `combine_as_variants`, `add_component_property`, `set_instance_properties` | xong |
| `append_children` cảnh báo khi variant rời component set | xong |
| `resize_nodes` ghi rõ chuyện constraint trong description | xong |
| ID node con trong instance (`I<inst>;<child>`) | ghi vào description `create_instance`, không trả map |
| `run_script` | **cần smoke test trong Figma** — xem dưới |

**`run_script` chưa chắc chạy được.** Sandbox main thread của Figma plugin chặn dynamic
code (`eval` / `new Function`) — đó chính là lý do mọi plugin cộng đồng đều đi theo hướng
tool nguyên tử, còn MCP official chạy JS tuỳ ý vì nó là kênh đặc quyền của Figma Desktop
chứ không phải plugin. Handler đã bắt lỗi đó và trả về thông báo rõ ràng thay vì
`ReferenceError`. Việc cần làm: chạy thử trong Figma Desktop
(`run_script` với `code: "return figma.currentPage.name"`).
Nếu sandbox từ chối → xoá tool, và cách thay thế rẻ nhất chính là `node_ids[]` (đã có),
cộng thêm `find_nodes` trả id để feed thẳng vào batch.
