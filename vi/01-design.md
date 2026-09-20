# fRPC: lớp RPC của Fomoxa

[English](../en/01-design.md) · Tiếng Việt

Trạng thái: bản thiết kế, chưa phải đặc tả và chưa phải hướng dẫn triển khai.

Tài liệu này mô tả một tầng RPC đặt trên Fomoxa ở mức khái niệm và mức byte, độc lập với ngôn ngữ lập trình và không tham chiếu API hay mã nguồn của bản triển khai nào.

Từ khóa quy phạm viết hoa (BẮT BUỘC, PHẢI, KHÔNG ĐƯỢC, NÊN, CÓ THỂ) được dùng theo nghĩa thông thường trong văn bản đặc tả.

Hai tài liệu nền, đọc trước:

| Tài liệu | Nội dung được dùng ở đây |
|---|---|
| `specification` RFC-0002 | Model là danh sách field liền nhau, không metadata; §9.1 chênh lệch phiên bản; Enum luôn 4 byte |
| `implementation-guide` `01-overview.md`, `02-flows.md` | Ba tầng, frame DATA, handshake fingerprint, nhịp tick, ô chờ một frame, vòng đời dữ liệu sự kiện |

---

## 0. Phạm vi

fRPC là một repo riêng. Nó không phải một phần của fomoxa-net và không nằm trong lộ trình của fomoxa-net. Nó đứng trên specification và sử dụng một bản triển khai net theo đúng cách một ứng dụng sử dụng net.

Ranh giới này có một phép thử: nếu fRPC đòi hỏi một thay đổi trong fomoxa-net thì logic đang được đặt sai tầng. `implementation-guide` §11 bước 5 phát biểu cùng điều kiện này cho người viết transport. Mọi thông tin fRPC cần đều nằm trong phần dữ liệu của frame DATA, phần mà `02-flows.md` §2.2 định nghĩa là byte mờ đối với tầng frame.

Mục tiêu của fRPC là cung cấp một lớp RPC tích hợp với các tính chất sẵn có của Fomoxa. Tiêu chí đánh giá mỗi quyết định trong tài liệu này là: quyết định đó khai thác được tính chất nào mà specification và các bản triển khai đã cung cấp.

Các hệ thống RPC khác được nhắc tới ở §3.4 và §17 để giải thích vì sao một lựa chọn thiết kế phổ biến không áp dụng được trong ràng buộc của Fomoxa.

---

## 1. Vị trí trong kiến trúc

```
   ỨNG DỤNG            service implementation · client stub
       │
   ────┼──────────────────────────────────────────────────────
       │
   fRPC                call id · dispatch · deadline
                       interceptor · identity · stream
       │
       ├── gọi CODEC       model ↔ byte            (RFC-0002)
       ├── nén (tùy chọn)  chỉ phần thân            §11
       │
   ────┼──────────────────────────────────────────────────────
       │
   FOMOXA NET          frame · handshake · heartbeat · tick
       │
   ────┼──────────────────────────────────────────────────────
       │
   TRANSPORT           TCP · UDP · WebSocket · QUIC
```

Quan hệ phụ thuộc: fRPC phụ thuộc net và codec. Net không phụ thuộc fRPC. Codec không phụ thuộc fRPC. Transport không phụ thuộc tầng nào phía trên.

Phân chia trách nhiệm: net xác định một chuỗi byte thuộc loại message nào. fRPC xác định chuỗi byte đó thuộc cuộc gọi nào và thành phần nào xử lý nó.

### 1.1 Tính chất thừa hưởng

Bảng sau liệt kê các tính chất fRPC nhận được từ specification và từ các bản triển khai net. fRPC không tạo ra tính chất nào trong số này; yêu cầu đối với fRPC là không làm mất chúng.

| Nguồn | Tính chất |
|---|---|
| Handshake so fingerprint từng message ở mức field (`02-flows` §3.3) | Schema hai bên lệch nhau ở phần giao nhau thì kết nối bị từ chối kèm mã lý do, tại thời điểm kết nối. Không tồn tại trạng thái hai bên deploy lệch nhau rồi giải mã ra giá trị khác nhau lúc chạy |
| Chênh lệch phiên bản RFC-0002 §9.1 | Nối field vào cuối một request model là thay đổi hợp lệ, và hợp lệ độc lập cho từng method (§3.3). Không cần đánh số phiên bản endpoint |
| Wire format tất định (RFC-0001 §6.1) | Ký payload, sinh idempotency key bằng cách băm nội dung, đối chiếu bản ghi audit, phát lại một chuỗi call. Các thao tác này yêu cầu hai bản triển khai sinh ra byte giống hệt nhau |
| Ranh giới transport của net (`01-overview` §3) | Chạy trên mọi transport net hỗ trợ, không phụ thuộc một stack giao thức nào. Thêm transport mới không thay đổi fRPC |
| Heartbeat và phát hiện peer chết (`02-flows` §4) | fRPC không cài đặt cơ chế liveness. Thời gian xử lý của một call không ảnh hưởng tới việc phát hiện peer chết |
| `fomoxa-inspect` | Đọc được thân của frame RPC sau khi bỏ qua phần đầu 5 byte và khối ctx nếu có (§4.4). Không cần bộ giải mã riêng cho tầng RPC |
| `fomoxac` và schema có sẵn | Kiểu request, response và item của method là model trong cùng schema đang dùng cho message thường. Không có ngôn ngữ mô tả interface thứ hai |
| Thời gian bơm từ ngoài (B8) | Chu kỳ deadline, timeout và hủy chạy hết được trong kiểm thử mà không cần chờ thời gian thực |

Hai dòng đầu là hệ quả của hai quyết định trong specification: *vị trí là định danh duy nhất* và *chuỗi fingerprint tiền tố*. Chúng không phải tính năng của tầng RPC.

Cùng hai quyết định đó tạo ra các giới hạn: không có bảng metadata tự do (§3.4), không có field optional, không xóa được field ở giữa.

### 1.2 Giả định vận hành: kết nối sống lâu

fRPC được thiết kế cho kết nối sống lâu:

```
   client  ──────── kết nối sống suốt phiên mở app ────────> backend
   service ──────── kết nối sống suốt vòng đời một việc ───> service
```

Giả định này là hệ quả của dòng đầu bảng §1.1. Handshake xác minh schema nghĩa là mỗi kết nối gửi greeting liệt kê toàn bộ schema, 14 byte mỗi message:

```
   ~1 100 message  ×  14 B  ≈  15 KB, trả trước khi READY
```

Chi phí này được khấu hao theo số lời gọi trên cùng kết nối.

Ba hệ quả:

- Ưu tiên chi phí mỗi lời gọi hơn chi phí mỗi kết nối. Các đánh đổi byte trong tài liệu này theo hướng đó; ví dụ bỏ `method id` (§3.3) làm greeting lớn hơn và mọi frame nhỏ hơn.
- Không tối ưu greeting khi chưa có số đo. Đây là giả định kiến trúc, không phải khiếm khuyết giao thức. Cache schema theo fingerprint là phương án để ngỏ, chỉ xem xét khi có số liệu.
- Kết nối lại hàng loạt. Giả định trên không áp dụng cho thời điểm deploy: mười nghìn peer kết nối lại đồng thời, mỗi peer 15 KB, tạo một đợt 150 MB. Phía kết nối lại PHẢI rải bằng jitter. Đây là yêu cầu vận hành, không phải yêu cầu giao thức.

---

## 2. Ràng buộc kế thừa từ Fomoxa

Mọi quyết định thiết kế ở các mục sau truy ngược về một dòng trong bảng này.

| # | Ràng buộc của Fomoxa | Nguồn | Hệ quả bắt buộc cho fRPC |
|---|---|---|---|
| C1 | Frame DATA chỉ có một trường điều khiển: mã định danh message. Không cờ, không số thứ tự | `02-flows` §2.2 | Mọi thông tin RPC PHẢI nằm trong phần dữ liệu. fRPC định nghĩa một phần đầu cố định §4 |
| C2 | Phần dữ liệu là byte mờ với net | `02-flows` §2.2 | fRPC được phép đặt phần đầu của riêng mình mà không phá tương thích |
| C3 | Handshake đã xác minh schema hai bên khớp ở mọi message chung | `02-flows` §3.3 | Không thương lượng kiểu lúc chạy. Kiểu không xác định không phải lỗi runtime của fRPC |
| C4 | Handshake không phải cơ chế bảo mật | `02-flows` §3.3 | Authentication thuộc fRPC, tách khỏi handshake §10 |
| C5 | Core giữ một ô chờ gửi. Gửi tiếp khi còn kẹt → lỗi tắc nghẽn | `01-overview` §5 | fRPC PHẢI có hàng đợi gửi riêng, có trần, và PHẢI báo tắc nghẽn lên ứng dụng §7 |
| C6 | Không luồng ngầm, không async, mọi thứ chạy trong `tick` | `01-overview` §2, §8 | Mọi xử lý của lõi fRPC chạy trong tick; một call kéo dài qua nhiều tick. Handler chạy trên luồng lái KHÔNG ĐƯỢC chặn; việc dài chia qua nhiều tick §8, §19 |
| C7 | Dữ liệu trong sự kiện chỉ sống tới tick kế tiếp | `02-flows` §6.2 | fRPC PHẢI giải mã hoặc sao chép ngay trong tick nhận. KHÔNG ĐƯỢC giữ tham chiếu trong call đang chờ |
| C8 | Mốc thời gian truyền từ ngoài vào, đồng hồ đơn điệu | `01-overview` B8, B9 | Deadline dùng `now` của tick. Lõi fRPC KHÔNG ĐƯỢC tự đọc đồng hồ; vỏ đọc đồng hồ và bơm `now` vào lõi §19 |
| C9 | Fomoxa không truyền lại, không sắp xếp lại, không lọc trùng | `01-overview` §7 | fRPC cũng không. Trên transport kiểu gói, ngữ nghĩa RPC suy giảm §5.7 |
| C10 | Trần 16 MiB cho một message | `02-flows` §2.7 | Thân vượt trần PHẢI cắt thành stream. fRPC kiểm trước khi gửi, không để transport trả ⊘ |
| C11 | Đúng một sự kiện kết thúc mỗi session | `01-overview` B6 | Hủy toàn bộ call đang chờ đúng một lần §5.8 |
| C12 | Không metadata trên dây, vị trí là định danh duy nhất | RFC-0001 §4.1 | Không có bảng metadata key/value cho mỗi call §3.4 |
| C13 | Enum luôn 4 byte và giá trị ngoài tập là luồng byte không hợp lệ | RFC-0002 §7 | Mã lỗi khai `UInt32`, không khai Enum, để mở rộng được §4.6 |
| C14 | Mọi bảng PHẢI có trần, và trần PHẢI là một con số cụ thể | `02-flows` §8 | fRPC công bố bảng trần §4.8 |

---

## 3. Mô hình khái niệm

### 3.1 Từ vựng

Mỗi khái niệm có đúng một tên, giữ nguyên tiếng Anh, theo quy ước của `02-flows.md`:

```
call        một lần gọi, có vòng đời riêng, định danh bằng call id
method      một thủ tục gọi được, định danh trên dây bằng mã định danh
            message của request model §4.4
kind        vai trò của một frame trong một call: REQUEST, RESPONSE,
            ERROR, ITEM, END, CANCEL, CREDIT
body        phần thân: một model Fomoxa đã mã hóa, hoặc rỗng
status      mã kết quả của một call đã kết thúc
deadline    mốc thời gian, sau đó call bị hủy ở phía gọi
pending     tập các call chưa kết thúc của một session
dispatcher  bảng mã định danh message của request → handler
```

Service không có mặt trên dây, và method không có định danh riêng. Xem §3.3.

### 3.2 Bốn loại call

```
   UNARY              REQUEST ──────────────>
                      <────────────── RESPONSE | ERROR

   SERVER STREAM      REQUEST ──────────────>
                      <────────────────── ITEM
                      <────────────────── ITEM
                      <─────────────── END | ERROR

   CLIENT STREAM      REQUEST ──────────────>
                      ITEM ─────────────────>
                      END ──────────────────>
                      <────────────── RESPONSE | ERROR

   BIDI STREAM        REQUEST ──────────────>
                      ITEM ─────────────────>
                      <────────────────── ITEM
                      END ──────────────────>
                      <─────────────── END | ERROR
```

Một call kết thúc bởi đúng một frame kết thúc: RESPONSE, END của bên bị gọi, hoặc ERROR. END của bên gọi chỉ đóng chiều gọi (§5.5). CANCEL và CREDIT không kết thúc call: CANCEL yêu cầu phía kia gửi frame kết thúc, CREDIT cấp thêm hạn mức item (§5.6).

### 3.3 Không có service id và không có method id

Service là cách gom nhóm method lúc khai báo và lúc đăng ký. Trên dây, thông tin cần xác định là handler nào xử lý frame, và service không tham gia vào việc đó.

Method đã có một định danh duy nhất trên dây: mã định danh message của request model. Fomoxa sinh định danh đó từ tên message, bảo đảm nó duy nhất trong schema, và handshake đã xác minh hai bên hiểu nó giống nhau. Một trường `method id` riêng sẽ tốn 4 byte trên mọi frame để lặp lại thông tin đã có trong frame.

Điều này khả thi vì chỉ REQUEST cần tra bảng dispatch. Sáu kind còn lại khớp bằng `call id`:

```
   REQUEST   → tra dispatcher bằng mã định danh message → handler
   RESPONSE  ┐
   ERROR     │
   ITEM      │
   END       ├→ tra pending bằng call id → bản ghi đã biết method
   CANCEL    │
   CREDIT    ┘
```

Ràng buộc kèm theo:

```
   Hai method KHÔNG ĐƯỢC dùng chung một kiểu request.
```

Ràng buộc này độc lập với cách dispatch. Dùng chung kiểu request làm hai method phụ thuộc nhau khi tiến hóa: chênh lệch phiên bản của RFC-0002 §9.1 xét theo kiểu, không theo method, nên nối một field cho method này là nối cho cả method kia.

Trường hợp cần chú ý là request rỗng: ba method không có tham số PHẢI khai ba model rỗng khác tên. Chi phí là ba mục 14 byte trong greeting, trả một lần lúc handshake.

Vi phạm ràng buộc này là lỗi lúc đăng ký (§4.10), không phải lúc chạy.

`FRpcVoid`, `FRpcError`, `FRpcCredit` và `FRpcCtx` cũng là model bình thường. Khai chúng trong schema cho ba kết quả: mã định danh của chúng do `fomoxac` sinh và bảo đảm không đụng, nên không cần dải id dành riêng; handshake xác minh hai bên đồng ý về hình dạng của `FRpcError` và `FRpcCtx` trước khi có lời gọi đầu tiên; và `FRpcVoid` là model rỗng dùng chung cho mọi method không có giá trị trả về, thay vì mỗi method khai một model rỗng riêng.

### 3.4 Khối context khai báo thay cho bảng metadata

Một số thông tin đi kèm mọi call mà không thuộc nghiệp vụ của call: deadline còn lại, trace id, token, tenant. Các hệ thống RPC phổ biến truyền chúng trong một bảng key/value tự do.

Phương án đó không áp dụng được. Một bảng key/value tự do là dữ liệu không khai báo, không có trong schema, không được handshake xác minh, và mâu thuẫn với RFC-0001 §4.1. Chấp nhận nó là mất tính chất ở dòng đầu bảng §1.1.

fRPC định nghĩa một khối context có khai báo, tùy chọn, tách khỏi request model:

```
   FRpcCtx
     deadline_ms       UInt32     thời gian còn lại, lan qua chuỗi gọi §5.3
     trace_id          Bytes
     span_id           Bytes
     token             String
     tenant            String
     idempotency_key   Bytes      khóa thử lại an toàn §20.4
     initial_credit    UInt32     lập hạn mức cho chiều trả về §5.6
```

RFC-0002 không có Optional và không có giá trị mặc định, nên mỗi field PHẢI có một giá trị "chưa khai" được quy định:

| Field | "Chưa khai" là | Nghĩa |
|---|---|---|
| `deadline_ms` | `0` | Không đặt deadline. Không mang nghĩa "đã hết hạn" |
| `trace_id`, `span_id` | `Bytes` rỗng | Không có ngữ cảnh trace |
| `token` | `String` rỗng | Không kèm credential cho call này §10.4 |
| `tenant` | `String` rỗng | Không khai tenant |
| `idempotency_key` | `Bytes` rỗng | Call không khai là lặp lại an toàn §20.4 |
| `initial_credit` | `0` | Chiều trả về chưa lập hạn mức, tức không giới hạn. Không mang nghĩa "hạn mức bằng không" §5.6 |

Ở bản này, "chưa khai" và "khai giá trị rỗng" là một. Field nào cần phân biệt hai trạng thái đó PHẢI có một field cờ đi kèm; KHÔNG ĐƯỢC dùng một giá trị đặc biệt khác để mã hóa sự phân biệt.

`FRpcCtx` là một model Fomoxa thông thường: handshake xác minh nó, chênh lệch phiên bản §9.1 áp dụng cho nó, byte của nó tất định, `fomoxa-inspect` đọc được nó. Interceptor đọc nó một cách tổng quát mà không đụng vào request model của method (§9). Khi không bật, nó chiếm 0 byte (§4.11).

Khác với bảng metadata, `FRpcCtx` có tập field cố định và được khai báo. Thêm một field vào `FRpcCtx` là một thay đổi schema, đi qua cùng quy trình với mọi thay đổi schema khác.

| Thông tin | Vị trí trong fRPC |
|---|---|
| Deadline | `FRpcCtx.deadline_ms`, lan qua chuỗi gọi §5.3 |
| Trace id / span id | `FRpcCtx` |
| Token xác thực | `FRpcCtx.token`, hoặc gắn một lần mỗi session §10 |
| Tenant / routing key | `FRpcCtx` |
| Cờ nén | Bit 7 của byte `kind` §11 |
| Phiên bản API | Do handshake fingerprint xử lý §2 C3 |
| Khóa tùy ý do người gọi đặt | Không hỗ trợ |

---

## 4. Định dạng trên dây

### 4.1 Toàn cảnh một message fRPC

```
   ┌─────────────── frame DATA của net (11 byte) ───────────────┐
   │ 00 │ 'F' │ 'O' │ message id │ độ dài dữ liệu │
   └────┴─────┴─────┴────────────┴────────────────┘
                                                   ┌── dữ liệu ──┐
                                                   │ đầu fRPC 5B │ thân │
                                                   └─────────────┴──────┘
```

Net xử lý frame này như một message thông thường. fRPC đọc một phần đầu cố định và một thân.

### 4.2 Phần đầu fRPC

```
   ┌──────┬──────────────┐
   │ kind │ call id      │
   │ 1B   │ 4B u32 LE    │
   └──────┴──────────────┘
    ^0     ^1-4

   Đúng 5 byte, cố định cho mọi kind, kể cả kind không có thân.
```

Method được xác định bằng mã định danh message của frame (§3.3, §4.4).

Byte đầu chia bit:

```
   bit 7     thân đã nén          §11
   bit 6     có khối ctx          §4.11
   bit 5-3   để dành, PHẢI = 0
   bit 2-0   kind, giá trị 0..6   §4.3
```

Bên nhận PHẢI kiểm ba bit để dành. Bỏ qua chúng làm một phần mở rộng tương lai bị diễn giải sai mà không có tín hiệu báo lỗi.

Phần đầu có độ dài cố định để bên đọc xác định được vị trí bắt đầu của thân mà không cần phân nhánh theo kind, kể cả khi đọc một gói bắt được trên dây mà không có ngữ cảnh.

`kind` chiếm 1 byte thay vì 4 byte như Enum của RFC-0002 vì phần đầu này không phải một Model. Nó là khung của tầng fRPC, cùng loại với hai byte `'F' 'O'` của net: do đặc tả quy định, không do schema sinh ra.

### 4.3 Bảng kind

| Giá trị | Tên | Thân | Ai gửi | Kết thúc call |
|---|---|---|---|---|
| 0 | REQUEST | request model | bên gọi | không |
| 1 | RESPONSE | response model | bên bị gọi | có |
| 2 | ERROR | `FRpcError` | bên bị gọi | có |
| 3 | ITEM | item model | bên gửi luồng | không |
| 4 | END | rỗng | bên gửi luồng | chiều đó |
| 5 | CANCEL | rỗng | bên gọi | không |
| 6 | CREDIT | `FRpcCredit` | bên nhận luồng | không |

ITEM và END chạy theo cả hai chiều; chiều nào phụ thuộc hình thái của method (§5.5). Với server streaming, bên bị gọi gửi; với client streaming, bên gọi gửi; với bidi, cả hai gửi. END chỉ kết thúc chiều gửi của bên gửi nó, không kết thúc call: một client stream kết thúc khi bên bị gọi trả RESPONSE, một bidi stream kết thúc khi bên bị gọi gửi END của chiều mình.

Giá trị kind 7 chưa dùng và không hợp lệ. Nhận giá trị đó, hoặc nhận một bit để dành khác 0, thì trả ERROR với mã `INTERNAL` cho call đó và KHÔNG ĐƯỢC đóng session. Đây là lỗi của tầng ứng dụng, không phải lỗi phân định frame; tầng duy nhất được đóng session vì byte không hợp lệ là tầng frame của net (`02-flows` §2.5).

### 4.4 Ngữ nghĩa của mã định danh message

Mã định danh message của frame DATA luôn mô tả phần thân, không mô tả call.

```
   REQUEST   → id của request model của method   ← đồng thời là khóa dispatch
   RESPONSE  → id của response model của method
   ITEM      → id của item model của chiều gửi                  §5.5
   ERROR     → id của FRpcError
   END       → id của FRpcVoid
   CANCEL    → id của FRpcVoid
   CREDIT    → id của FRpcCredit
```

Với REQUEST, một giá trị phục vụ hai mục đích: mô tả thân và xác định handler. Hai mục đích này không xung đột vì ràng buộc ở §3.3 làm ánh xạ `kiểu request → method` là song ánh. Sáu kind còn lại chỉ mô tả thân; method của chúng lấy từ bản ghi pending.

Hệ quả của quy tắc này:

- Handshake của net tiếp tục xác minh từng kiểu request, response và item ở mức field, theo đúng cơ chế `02-flows` §3.3.
- Chênh lệch phiên bản RFC-0002 §9.1 áp dụng cho từng kiểu: nối một field vào cuối request model là thay đổi hợp lệ và cổng ③ của handshake chấp nhận.
- `fomoxa-inspect` đọc được thân bằng cách bỏ qua 5 byte đầu, cộng `[độ dài][FRpcCtx]` khi bit 6 bật (§4.11). Thân có bit 7 bật phải giải nén trước (§11).

Kèm theo là một phép kiểm bắt buộc: bên nhận PHẢI đối chiếu mã định danh message với kiểu mong đợi cho cặp `(method, kind)`, trong đó method lấy từ dispatcher với REQUEST và từ pending với các kind còn lại. Lệch thì trả ERROR `INVALID_ARGUMENT` và không đóng session. Luật này áp cho mọi kind, kể cả END, CANCEL và CREDIT.

### 4.5 Model hệ thống

fRPC khai báo bảy model. `FRpcVoid` và `FRpcError` PHẢI có trong mọi schema dùng fRPC. `FRpcCredit` chỉ cần khi bản triển khai hỗ trợ flow control (§5.6), `FRpcCtx` chỉ cần khi có dùng khối ctx (§3.4), và ba model reflection chỉ cần khi server bật reflection (§21).

Mọi model hệ thống PHẢI khai với codec tên `rpc`. Message id sinh từ tên model cộng tên codec (`FRpcVoid.rpc`), nên tên codec cố định là điều kiện để mọi bên tính ra cùng một id mà không cần trao đổi gì trước. Điều này bắt buộc cho reflection: công cụ bên ngoài phải tính được id và fingerprint của các model reflection trước khi biết gì về server. Model của ứng dụng đặt tên codec tùy ý.

```
   FRpcVoid
     (không field)

   FRpcError
     code     UInt32
     message  String

   FRpcCredit
     items    UInt32

   FRpcCtx
     (bảy field, §3.4)
```

`FRpcVoid` có `n = 0`. Theo `02-flows` §3.3 nhánh ⓓ, tiền tố rỗng là tiền tố của mọi chuỗi, nên model này không gây từ chối handshake trong bất kỳ trường hợp nào.

`FRpcError` mở rộng được theo RFC-0002 §9.1: nối field ở cuối là chênh lệch phiên bản hợp lệ, bên cũ đọc tới field nó biết rồi dừng. Đường mở rộng đã dự trù là cặp `details_msg_id: UInt32` + `details: Bytes`, cho phép đính một model do ứng dụng định nghĩa vào lỗi mà vẫn giữ nguyên tắc mọi dữ liệu trên dây đều có kiểu khai báo. Phần mở rộng này chưa đưa vào bản thiết kế hiện tại (§13.2).

### 4.6 Mã lỗi

Mã lỗi khai `UInt32`, không khai Enum. RFC-0002 §7 quy định giá trị Enum ngoài tập đã định nghĩa là luồng byte không hợp lệ; nếu mã lỗi là Enum thì một mã mới ở phiên bản sau làm bên cũ không giải mã được frame lỗi.

| Mã | Tên | Nghĩa |
|---|---|---|
| 1 | CANCELLED | Bên gọi đã hủy |
| 2 | DEADLINE_EXCEEDED | Hết hạn trước khi có frame kết thúc |
| 3 | UNIMPLEMENTED | Không có handler cho kiểu request này |
| 4 | INVALID_ARGUMENT | Thân sai kiểu hoặc giải mã thất bại |
| 5 | UNAUTHENTICATED | Chưa xác thực |
| 6 | PERMISSION_DENIED | Đã xác thực nhưng không đủ quyền |
| 7 | RESOURCE_EXHAUSTED | Chạm một trần ở §4.8 |
| 8 | FAILED_PRECONDITION | Trạng thái không cho phép |
| 9 | UNAVAILABLE | Session kết thúc khi call chưa hoàn tất |
| 10 | INTERNAL | Lỗi không phân loại được |

Mã `0` không dùng: RESPONSE và END đã biểu thị thành công, nên không cần mã "OK" trên dây. Bên nhận xử lý mã ngoài bảng như `INTERNAL` và giữ nguyên giá trị để ghi nhật ký.

### 4.7 Call id

Call id là số nguyên 32 bit, do bên gọi cấp, tăng dần, phạm vi là một session.

Hai bên đều được phép mở call. Để phân biệt bên cấp, id chia theo chẵn lẻ:

```
   client cấp call id chẵn    0, 2, 4, ...
   server cấp call id lẻ      1, 3, 5, ...
```

Quay vòng được phép. Tái dùng một id đang nằm trong `pending` KHÔNG ĐƯỢC phép. Tập `pending` bị chặn bởi trần ở §4.8 nên điều kiện này luôn thỏa được.

Chọn 32 bit thay vì 64 bit: phần đầu ngắn hơn 4 byte trên mọi message, đổi lại một quy tắc quay vòng. Trần ở §4.8 giới hạn số call sống đồng thời ở mức thấp hơn 2³² nhiều bậc.

### 4.8 Bảng trần

Theo yêu cầu của `02-flows` §8: mỗi bảng PHẢI có một con số.

| Số lượng | Trần đề xuất | Chạm trần thì |
|---|---|---|
| Call đang chờ, mỗi session, mỗi chiều | 65 536 | Bên gọi: từ chối tại chỗ. Bên bị gọi: ERROR `RESOURCE_EXHAUSTED` |
| Frame trong hàng đợi của một call | 64 | Lệnh gửi trả lỗi tắc nghẽn §7.4 |
| Frame đang xếp trên cả kết nối | 1 024 | Như trên. Cả hai con số đều bắt buộc. CREDIT không tính vào hai trần này §7.5 |
| Lệnh chờ trên hàng đợi vào luồng lái (chế độ self-driven) | 1 024 | Lệnh gửi từ luồng khác trả lỗi tắc nghẽn §19 |
| Hạn mức byte mỗi lượt của scheduler | 64 KiB | Nhường lượt cho call khác §7.2 |
| Thời gian hàng đợi một call đầy liên tục | 5 giây | Kết thúc stream đó với `RESOURCE_EXHAUSTED` §5.4 |
| Method đăng ký | 65 536 | Lỗi lúc đăng ký |
| Độ dài khối ctx | 64 KiB | Dừng phân bổ §4.11 |
| Độ dài thân | 16 MiB − 5 byte | Lỗi tại chỗ cho bên gọi, không gửi đi §2 C10 |

Các trần do bản triển khai đặt CÓ THỂ thay đổi. Trần độ dài thân suy ra từ net và KHÔNG ĐƯỢC thay đổi.

### 4.9 Ví dụ byte

Gọi `Player.Get`, call id 4, request model `GetPlayerRequest { id: UInt32 = 7 }`. Giả sử id của `GetPlayerRequest` là `0x4A2F8810`.

```
   00 46 4F              frame DATA, 'F' 'O'
   10 88 2F 4A           message id = 0x4A2F8810   (GetPlayerRequest)
   09 00 00 00           độ dài dữ liệu = 9
   ──────────────────────────────────────────────── hết phần của net
   00                    kind = REQUEST
   04 00 00 00           call id = 4
   ──────────────────────────────────────────────── hết phần đầu fRPC
   07 00 00 00           thân: GetPlayerRequest.id = 7

   Tổng 20 byte. Chi phí cố định: 11 của net + 5 của fRPC = 16 byte.
```

Frame END của cùng call:

```
   00 46 4F  <id FRpcVoid>  05 00 00 00  04  04 00 00 00
   Tổng 16 byte, thân rỗng.
```

### 4.10 Bảng dispatch

Khóa là mã định danh message của request model. fRPC không định nghĩa phép băm riêng, không tạo không gian id thứ hai và không có bước sinh id bổ sung.

Ba phép kiểm BẮT BUỘC thực hiện lúc đăng ký:

```
   1. Hai method dùng chung một kiểu request  → lỗi, dừng tiến trình §3.3
   2. Tập id của fRPC giao với tập id message
      thường của ứng dụng                     → lỗi, dừng tiến trình
   3. Kiểu request/response/item khai trong
      descriptor không có trong schema        → lỗi, dừng tiến trình
```

Tập id của fRPC gồm mọi kiểu request, response và item của cả hai chiều khai trong descriptor, cộng `FRpcError`, `FRpcVoid` và `FRpcCredit` nếu có. `FRpcCtx` không nằm trong tập này vì nó chỉ xuất hiện bên trong thân, không bao giờ làm mã định danh của frame. Bảng dispatch là tập con của tập id, gồm các kiểu request.

Phép kiểm 2 tồn tại vì một session được phép mang lẫn message thường và message RPC. Ứng dụng dùng message thường trên cùng session với fRPC PHẢI khai tập id của các message đó. Hai tập giao nhau tạo ra trạng thái không phân giải được.

Net giao mọi frame DATA lên thành sự kiện MESSAGE, không lọc theo schema (`02-flows` §6.1). Tiền đề "client có một message mà server không biết thì server không bao giờ nhận message đó" (`02-flows` §3.3) chỉ đúng khi hai bên hợp tác, và một client như công cụ dòng lệnh, hay một client deploy trước server, có thể gửi đúng loại message đó. Vì vậy bên nhận phân loại một frame DATA theo ba nhánh:

```
   id thuộc tập message thường đã khai   → giao cho ứng dụng
   id thuộc tập id của fRPC               → frame fRPC, đủ mọi luật
   id không thuộc tập nào                 → KHÔNG giao cho ứng dụng
        phần đầu 5 byte hợp lệ → xử lý như frame fRPC:
                                 REQUEST → UNIMPLEMENTED (§5.2)
                                 frame của call đang chờ → INVALID_ARGUMENT (§4.4)
                                 call không còn mở → bỏ qua (§5.9)
        phần đầu hỏng          → bỏ, không trả lời
```

Ứng dụng chỉ nhận những message nó đã khai. Frame không rõ là gì có phần đầu hỏng thì bị bỏ mà không trả lời, vì call id đọc từ một frame như vậy không đáng tin; trả ERROR cho một call id ngẫu nhiên có thể kết thúc nhầm một call thật ở đầu kia.

Không có cơ chế phát hiện ba lỗi này lúc chạy.

### 4.11 Khối ctx

Khi bit 6 của byte `kind` bật, phần thân bắt đầu bằng một khối ctx:

```
   ┌──────────────┬────────────────┬──────────────────┐
   │ độ dài ctx   │ FRpcCtx        │ request model    │
   │ 4B u32 LE    │ = độ dài byte  │                  │
   └──────────────┴────────────────┴──────────────────┘

   Bit 6 tắt → thân chỉ có request model, khối ctx chiếm 0 byte.
```

Chỉ REQUEST được mang ctx. Bit 6 bật trên kind khác là lỗi `INTERNAL` cho call đó.

Tiền tố độ dài là bắt buộc. Hai model nối tiếp nhau chỉ giải mã đúng khi model đứng trước có kích thước cố định. `FRpcCtx` không cố định vì nó được phép tiến hóa. Với bên gửi có 7 field và bên nhận biết 6 field, theo RFC-0002 §9.1 bên nhận dừng ở field 6 và coi phần dư là byte thừa hợp lệ. Không có tiền tố độ dài thì phần dư đó là các byte đầu của request model, và request model giải mã ra giá trị sai mà không có tín hiệu lỗi.

Phần đầu 5 byte ở §4.2 không cần tiền tố vì nó cố định, do đặc tả quy định, và không tiến hóa theo schema.

Trần độ dài khối ctx là 64 KiB, dùng làm chốt chặn phân bổ, cùng loại với trần 1 MiB của frame handshake.

Mã định danh message của frame vẫn mô tả request model, không mô tả ctx (R3). Kiểu của ctx cố định nên không cần định danh.

`FRpcCtx` không khai làm field đầu của mọi request model vì cách đó buộc mọi request model khai thêm một field; khi đó fingerprint và số field `n` của mọi method đổi mỗi lần `FRpcCtx` đổi, làm toàn bộ bảng method lệch phiên bản cùng lúc. Khối tách rời có tiền tố độ dài giữ hai trục tiến hóa độc lập.

---

## 5. Các luồng

### 5.1 Unary, đường thuận

```
   CLIENT                                          SERVER
     │                                               │
     │ (session READY - bắt buộc, §2 C3)             │
     │                                               │
     │ ── DATA[ REQUEST, call=4, msg=GetPlayerReq ] > │
     │                                               │ tra dispatcher bằng msg
     │                                               │ interceptor vào §9
     │                                               │ handler chạy
     │                                               │ interceptor ra
     │ <── DATA[ RESPONSE, call=4, msg=GetPlayerRes ]─│
     │                                               │
     │ xóa 4 khỏi pending, hoàn tất call             │
```

Bên nhận RESPONSE PHẢI kiểm hai điều kiện theo thứ tự, dừng ở điều kiện đầu tiên không thỏa:

```
   ① call id có trong pending?      không → bỏ qua, không sinh lỗi (§5.9)
   ② message id đúng kiểu response
      mà method của call đó mong đợi? không → hoàn tất call
                                             với INVALID_ARGUMENT
```

Điều kiện ① bỏ qua thay vì báo lỗi vì một RESPONSE đến sau khi call đã hết hạn là trạng thái bình thường, xem §5.9.

### 5.2 Unary, đường lỗi

Handler trả lỗi, interceptor chặn, hoặc không có handler:

```
     │ ── DATA[ REQUEST, call=4, msg=X ] ──────────> │
     │                                               │ X không có trong
     │                                               │ bảng dispatch
     │ <── DATA[ ERROR, call=4, FRpcError{3,…} ] ─── │
```

Không có handler không phải lỗi handshake. Handshake bảo đảm hai bên hiểu cùng một bộ kiểu (§2 C3) và chấp nhận cả trường hợp một bên có message mà bên kia không biết (`02-flows` §3.3). Nó không bảo đảm bên kia có cài đặt method. Vì vậy `UNIMPLEMENTED` cần tồn tại dù schema đã khớp.

Nếu `X` không nằm trong schema của bên nhận, nó không thuộc tập id nào của bên nhận. Theo §4.10, frame đó không được giao cho ứng dụng mà được xử lý như một REQUEST fRPC, và bên gọi nhận `UNIMPLEMENTED` ngay. Trường hợp thường gặp là client deploy một method mới trước server.

### 5.3 Deadline, lan truyền và cancel

Deadline truyền trên dây dưới dạng thời gian còn lại tính bằng mili giây: `FRpcCtx.deadline_ms`.

Dạng thời lượng được chọn thay vì mốc tuyệt đối vì hai máy không chung đồng hồ, và Fomoxa cấm đọc đồng hồ treo tường ở mọi tầng (B9). Một mốc tuyệt đối chỉ có nghĩa khi hai bên đồng bộ giờ.

```
   A ──[ deadline_ms = 2000 ]──> B
                                  B xử lý mất 300 ms
                                  B ──[ deadline_ms = 1700 ]──> C
```

C biết phần ngân sách còn lại, và không chặng nào xử lý một request mà A đã hủy.

Bên gọi giữ mốc tuyệt đối cục bộ, tính từ `now` của tick lúc gửi:

```
   tick N     gửi REQUEST, ghi pending{ call=4, deadline = now + 2s }

   tick N+k   bước đồng hồ: now > deadline
                ├── xóa 4 khỏi pending
                ├── hoàn tất call: DEADLINE_EXCEEDED
                └── đẩy CANCEL(call=4) vào hàng đợi gửi

   server     nhận CANCEL
                ├── call còn chạy → báo hủy cho handler, gửi ERROR CANCELLED
                └── call đã xong  → bỏ qua, không sinh lỗi
```

Ba quy tắc:

1. Bên gọi hoàn tất call trước, gửi CANCEL sau. Kết quả phía bên gọi không phụ thuộc việc CANCEL có tới nơi hay không.
2. Bên bị gọi dùng `deadline_ms` làm trần trên, không phải làm lệnh. Với unary, nó PHẢI có hạn riêng cho mỗi handler: một `deadline_ms` lớn hoặc bằng 0 không được phép giữ tài nguyên vô hạn, và trên transport kiểu gói CANCEL có thể không tới (§2 C9). Hạn thực tế là giá trị nhỏ hơn giữa hai con số. Stream theo luật riêng ở §5.4.
3. Deadline dùng `now` truyền vào tick (§2 C8). Bản triển khai KHÔNG ĐƯỢC đọc đồng hồ hệ thống tại đây; điều kiện này cần thiết để chạy hết một chu kỳ hết hạn trong kiểm thử.

Chặng giữa dựng ctx cho chặng sau từ ctx nó nhận được. Không phải field nào cũng chép:

| Field | Chặng giữa làm gì | Lý do |
|---|---|---|
| `deadline_ms` | Ngân sách còn lại tính tại thời điểm gọi tiếp | Quy tắc lan truyền ở trên |
| `trace_id`, `tenant` | Chép nguyên | Định danh của cả chuỗi, không của một chặng |
| `span_id` | Chép nguyên làm span cha | Bên nào có tracer thì ghi đè bằng span mới của mình |
| `token` | KHÔNG chép | Credential thuộc đúng một chặng §10.4. Chuyển tiếp là quyết định của ứng dụng, không phải mặc định của tầng |
| `idempotency_key` | KHÔNG chép | Khóa suy từ thân của chặng này §20.4, không dùng cho thân khác |
| `initial_credit` | KHÔNG chép | Hạn mức thuộc từng call §5.6 |

Hai trường hợp biên:

1. Ngân sách đã cạn thì chặng giữa KHÔNG ĐƯỢC gọi tiếp. Mã hóa ngân sách cạn thành `deadline_ms = 0` là sai, vì theo §3.4 giá trị đó nghĩa là không giới hạn; một ngân sách đã hết sẽ trở thành vô hạn ở chặng sau.
2. Bên gọi không khai `deadline_ms` thì ngân sách lan truyền là hạn handler của chặng giữa (quy tắc 2 ở trên). Chặng sau không được phép chạy lâu hơn thời gian chặng giữa còn chờ nó. Handler không có hạn tổng (stream, §5.4) thì chặng sau nhận `deadline_ms = 0`.

Deadline nằm trong ctx thay vì trong phần đầu vì đặt vào phần đầu sẽ tốn 4 byte trên mọi frame, kể cả RESPONSE, ITEM, END và CANCEL là các kind không có khái niệm deadline.

### 5.4 Server streaming

```
     │ ── DATA[ REQUEST, call=6, msg=WatchReq ] ───> │ mở stream
     │ <──────────── DATA[ ITEM, call=6 ] ────────── │
     │ <──────────── DATA[ ITEM, call=6 ] ────────── │
     │ ── DATA[ CANCEL, call=6 ] ─────────────────> │ (bên gọi hủy)
     │ <──────────── DATA[ ERROR, call=6, CANCELLED ]│
```

Quy tắc:

- ITEM đến sau frame kết thúc bị bỏ qua, không sinh lỗi.
- `ctx.deadline_ms` khác 0 ràng buộc toàn bộ call, không chỉ thời gian tới item đầu tiên.
- Stream với `deadline_ms = 0` không có hạn tổng. Nó sống tới khi có CANCEL, có frame kết thúc, hoặc session kết thúc. Hạn handler bắt buộc của unary (§5.3 quy tắc 2) không áp cho stream. Stream chạy trên transport kiểu dòng byte (§5.7), nên CANCEL không thể mất khi session còn sống; peer chết thì heartbeat của net kết thúc session và §5.8 dọn call. Một stream im lặng lâu ở cả hai chiều, ví dụ stream theo dõi sự kiện hiếm, là trạng thái hợp lệ.
- Bên bị gọi CÓ THỂ khai hạn rảnh theo từng method: khoảng tối đa không nhận được frame nào của call từ bên kia, tính cả CREDIT. Quá hạn thì kết thúc call với `DEADLINE_EXCEEDED`. Hạn rảnh là chính sách của bản triển khai để bắt lỗi ứng dụng, như một bên quên handle của call, không phải luật liên thông (§15.1).
- Bên sản xuất bị chặn bởi hàng đợi của call đó (§7.4), và PHẢI phân biệt hai tình huống:

```
   hàng đợi đầy tại thời điểm hiện tại → báo bên sản xuất chờ tick sau
                                         trạng thái bình thường

   hàng đợi đầy liên tục quá ngưỡng    → kết thúc stream, RESOURCE_EXHAUSTED
                                         bên nhận không tiêu thụ
```

  Tình huống thứ nhất xảy ra khi handler sản xuất nhanh hơn tốc độ đường truyền. Ngưỡng ở §4.8.

  Đây là áp lực ngược cục bộ: nó tác động tới bên sản xuất ở đầu này, không tới bên sản xuất ở đầu kia. Áp lực ngược xuyên đầu kia là hạn mức credit ở §5.6.
- Thứ tự item là thứ tự ITEM đến nơi. fRPC không đánh số và không sắp xếp lại (§2 C9).

### 5.5 Client streaming và bidi streaming

Hai hình thái còn lại dùng đúng tập kind ở §4.3, chỉ đổi chiều của ITEM và END.

```
   Client streaming                        Bidi streaming

   │ ── REQUEST, call=8 ──────> │          │ ── REQUEST, call=10 ─────> │
   │ ── ITEM, call=8 ─────────> │          │ ── ITEM, call=10 ────────> │
   │ ── ITEM, call=8 ─────────> │          │ <───────── ITEM, call=10 ─ │
   │ ── END, call=8 ──────────> │          │ ── ITEM, call=10 ────────> │
   │ <────── RESPONSE, call=8 ─ │          │ <───────── ITEM, call=10 ─ │
                                           │ ── END, call=10 ─────────> │
                                           │ <────────── END, call=10 ─ │
```

Quy tắc:

- Mã định danh message của ITEM do bên gọi gửi mô tả item model của chiều gọi, tách khỏi mã định danh của REQUEST mở call. Method khai cả hai. Method không khai riêng thì item của chiều gọi dùng lại mã của REQUEST.
- END từ bên gọi đóng chiều gọi. Sau đó bên gọi KHÔNG ĐƯỢC gửi thêm ITEM trên call đó.
- ITEM hoặc END đến sau khi chiều đó đã đóng bị bỏ qua, không sinh lỗi, cùng luật với §5.4.
- ITEM mang mã định danh không phải item model của chiều đó dẫn tới ERROR `INVALID_ARGUMENT` và kết thúc call. Đây là mức hai của §5.9.
- Bên bị gọi kết thúc call bằng RESPONSE (client streaming) hoặc bằng END của chiều mình (bidi). Trước thời điểm đó, call vẫn mở kể cả khi bên gọi đã đóng chiều gọi.
- Call id đã phân biệt bên mở call (§4.7), nên không cần thêm dấu hiệu nào để biết ITEM thuộc chiều nào.

### 5.6 Flow control bằng hạn mức credit

Áp lực ngược ở §5.4 chỉ tác động tới bên sản xuất cục bộ. Khi bên sản xuất ở đầu kia sinh item nhanh hơn bên nhận tiêu thụ, hàng đợi đầy nằm ở đầu gửi và bên nhận không có cách nào làm chậm nó lại. Frame CREDIT (kind 6, §4.3) là đường báo đó.

```
   FRpcCredit
     items   UInt32   số item bên nhận sẵn sàng nhận thêm
```

Hạn mức là của từng chiều. Mỗi chiều gửi item của một call có hạn mức riêng, do bên nhận chiều đó điều khiển, và hạn mức ở một trong hai trạng thái:

```
   CHƯA LẬP           → bên sản xuất gửi không giới hạn
        │
        │  tín hiệu lập đầu tiên của bên nhận chiều này, đúng một lần
        ▼
   ĐÃ LẬP (n)         → bên sản xuất được gửi tối đa n ITEM
                        mỗi ITEM gửi đi   → n giảm 1
                        n = 0             → bên sản xuất dừng, call vẫn mở
                        CREDIT tới        → n cộng thêm, bên sản xuất chạy tiếp
```

Không có đường quay lại từ ĐÃ LẬP về CHƯA LẬP. Sau khi đã lập, CREDIT chỉ làm hạn mức tăng.

Tín hiệu lập:

| Chiều | Bên nhận chiều đó | Tín hiệu lập |
|---|---|---|
| Chiều trả về (server stream, bidi) | Bên gọi | `FRpcCtx.initial_credit` khác 0 trên REQUEST; nếu không có thì CREDIT đầu tiên |
| Chiều gọi (client stream, bidi) | Bên bị gọi | CREDIT đầu tiên |

Chiều trả về lập bằng ctx vì bên bị gọi chạy handler ngay trong tick nhận REQUEST (§6 bước 2); một CREDIT gửi sau REQUEST tới nơi khi handler đã sinh item. Chiều gọi không cần đường này: bên bị gọi gửi CREDIT đầu tiên ngay trong tick nhận REQUEST.

Quy tắc:

- Một bên không cài flow control không gửi tín hiệu lập nào, nên mọi chiều nó nhận đều ở CHƯA LẬP và bên kia hành xử đúng như khi chưa có cơ chế này. Đây là điều kiện để thêm kind 6 không làm hỏng tương thích.
- `initial_credit = 0` là "chưa khai" theo §3.4, nên không lập hạn mức. Mở một chiều với hạn mức bằng 0 cần một field cờ đi kèm; phần này để dành ở §13.2.
- Bên nhận KHÔNG ĐƯỢC coi item vượt hạn mức là lỗi, và PHẢI giao chúng như item bình thường. Item vượt hạn mức có hai nguồn hợp lệ: item bên sản xuất đã gửi trước khi tín hiệu lập tới nơi, và item từ một bên không cài flow control mà vẫn nhận được `initial_credit`. Chiều gọi có thể nhận tối đa khoảng một vòng khứ hồi item trước khi CREDIT đầu tiên có hiệu lực.
- CREDIT cộng dồn, bão hòa ở trần của `UInt32`.
- Hạn mức bằng 0 KHÔNG kết thúc call và KHÔNG tính vào ngưỡng hàng đợi đầy ở §5.4. Một bên nhận cấp phát chậm là chủ đích, không phải bên nhận hỏng. Call vẫn chịu `deadline_ms` và hạn rảnh nếu có (§5.4).
- Trên bidi, CREDIT nhận được cộng vào hạn mức của chiều gửi của bên nhận frame đó.
- CREDIT cho một call không còn mở bị bỏ qua, không sinh lỗi. Đây là mức một của §5.9.
- `FRpcCredit` PHẢI có mã định danh message riêng, không dùng chung với `FRpcVoid` hay `FRpcError`. Bản triển khai không cài flow control thì không cần khai model này. Bản triển khai đó bỏ qua mọi frame CREDIT nó nhận: không trả lỗi, không kết thúc call, và không giao frame cho ứng dụng (§4.10). Bên gửi CREDIT không mất gì, vì bên kia vốn gửi không giới hạn.
- CREDIT là frame điều khiển và đi theo luật hàng đợi riêng ở §7.5.

### 5.7 Transport: mặc định và giới hạn

fRPC thừa hưởng tính độc lập transport của net (§1.1). Các transport khác nhau không cho ra cùng một ngữ nghĩa.

Mặc định của fRPC là transport kiểu dòng byte: TCP, TLS, WebSocket, QUIC stream. Mọi phát biểu về ngữ nghĩa trong tài liệu này áp dụng cho chế độ đó.

Trên transport kiểu gói, gói có thể mất, đến sai thứ tự hoặc đến hai lần. fRPC không xử lý các trạng thái đó, theo C9.

| Loại call | Transport kiểu dòng byte | Transport kiểu gói |
|---|---|---|
| Unary | Đúng ngữ nghĩa | REQUEST mất → DEADLINE_EXCEEDED. RESPONSE mất → như trên. REQUEST đến hai lần → handler chạy hai lần |
| Server stream, client stream, bidi | Đúng ngữ nghĩa | Không định nghĩa; item sai thứ tự không phát hiện được, END có thể đến trước ITEM cuối |
| CREDIT | Đúng ngữ nghĩa | CREDIT mất thì bên sản xuất dừng tới deadline hoặc hạn rảnh của call |

Stream và flow control yêu cầu transport kiểu dòng byte. Unary trên transport kiểu gói là at-most-once từ phía bên gọi và có thể lặp ở phía bên bị gọi, nên method dùng ở chế độ đó PHẢI idempotent. Chống lặp bằng bảng call id đã xử lý thuộc về ứng dụng; đưa nó vào fRPC là dựng lại tầng tin cậy mà RFC-0001 §2 đã loại bỏ.

### 5.8 Session kết thúc

Net sinh đúng một sự kiện kết thúc (§2 C11). fRPC xử lý như sau:

```
   DISCONNECT hoặc HANDSHAKE FAILED
        │
        ├── mọi call trong pending (bên gọi)  → hoàn tất UNAVAILABLE
        ├── mọi call đang chạy (bên bị gọi)   → báo hủy cho handler, không gửi gì
        ├── xóa hàng đợi gửi
        └── đặt cờ đã dọn - đường dẫn thứ hai KHÔNG ĐƯỢC dọn lần nữa
```

Không gửi ERROR cho các call đang chạy vì session đã kết thúc. Handler PHẢI nhận được tín hiệu hủy để giải phóng tài nguyên; vì vậy handler cần một tín hiệu hủy chứ không chỉ một giá trị trả về.

### 5.9 Ba mức bất thường

fRPC phân biệt ba mức. Mức thứ nhất xảy ra trong vận hành bình thường và KHÔNG ĐƯỢC xử lý như lỗi.

| Mức | Ví dụ | Hậu quả |
|---|---|---|
| Đua lành tính | RESPONSE cho call id không còn pending · ITEM đến sau END · CANCEL hoặc CREDIT cho call đã xong · REQUEST trùng call id đang chạy | Bỏ qua. Không sinh lỗi, không ghi nhật ký mức lỗi, không sinh sự kiện |
| Lỗi RPC | `UNIMPLEMENTED`, `PERMISSION_DENIED`, `INVALID_ARGUMENT`, lỗi nghiệp vụ của handler | ERROR, kết thúc một call |
| Vi phạm giao thức fRPC | kind 7 · bit để dành khác 0 · call id sai chẵn/lẻ · thân giải mã hỏng · ctx vượt trần | ERROR cho call đó, session vẫn hoạt động (R4) |

Mức thứ nhất phát sinh từ tình huống hai frame gặp nhau trên đường:

```
   client hết deadline → hoàn tất call, gửi CANCEL
        ↓ đồng thời
   server đã gửi RESPONSE
        ↓
   client nhận RESPONSE cho một call không còn trong pending
```

Không bên nào vi phạm giao thức. Xử lý tình huống này như vi phạm sẽ biến mọi lần timeout đua với response thành sự cố.

Hai quy tắc BẮT BUỘC, vì đây là nơi hai bản triển khai dễ lệch nhau:

Vi phạm giao thức fRPC KHÔNG ĐƯỢC đóng session. R4 quy định chỉ tầng frame của net được đóng session.

REQUEST trùng call id đang chạy PHẢI bị bỏ qua, không trả lỗi. Trả `INVALID_ARGUMENT` cho call id đó sẽ kết thúc call hợp lệ đang chạy. Bỏ qua cũng làm một REQUEST bị lặp trên transport kiểu gói không kích hoạt handler lần thứ hai chừng nào call gốc còn chạy (§5.7).

Nếu `FRpcError` trong frame ERROR giải mã hỏng, call hoàn tất với `INTERNAL`.

---

## 6. Nhịp tick của fRPC

fRPC sở hữu session của net và cung cấp một điểm vào `tick(now)`. Thứ tự sau là BẮT BUỘC; nó phản chiếu §8 của `02-flows.md`:

```
   frpc_tick(now):

       ── 1. net.tick(now) ──
       lấy danh sách sự kiện. Net đã đẩy ô chờ của nó trước mọi việc khác.

       ── 2. xử lý sự kiện, theo thứ tự net trả về ──
           READY        → cho phép gửi call
           MESSAGE      → id là message thường đã khai? §4.10
                          có    → giao cho ứng dụng
                          không → tách 5 byte đầu, phân loại kind
                                  (id lạ, phần đầu hỏng → bỏ, không trả lời)
                          giải mã thân tại đây (§2 C7)
                          REQUEST → dispatcher, có thể sinh frame ra hàng đợi
                          RESPONSE/ERROR/ITEM/END → hoàn tất hoặc nuôi call
                                    ở phía gọi; ITEM/END của chiều gọi
                                    giao cho handler ở phía bị gọi
                          CANCEL  → báo hủy handler
                          CREDIT  → cộng hạn mức gửi của call §5.6
           DISCONNECT   → dọn theo §5.8, đúng một lần
           POLL / REPLY → fRPC không xử lý

       ── 3. đồng hồ deadline ──
       duyệt pending, call quá hạn → hoàn tất + đẩy CANCEL vào hàng đợi

       ── 4. tiếp tục các handler còn dở ──
       handler đã khai CHƯA XONG ở tick trước được gọi lại §8
       kết quả sinh ra đi vào hàng đợi gửi

       ── 5. scheduler rót sang net ──
       xoay vòng giữa các call, hạn mức theo byte §7.2
       lặp: net.send(...) cho tới khi hết hàng đợi hoặc net báo tắc nghẽn

       ── 6. trả kết quả cho ứng dụng ──
```

Hai ràng buộc thứ tự:

Bước 3 sau bước 2. Một RESPONSE đến trong cùng tick với thời điểm deadline được ưu tiên. Thứ tự ngược lại làm một số call bị báo hết hạn dù câu trả lời đã có trong bộ đệm.

Bước 5 sau bước 4. Một REQUEST đến ở tick này được trả lời trong cùng tick. Thứ tự ngược lại làm mọi câu trả lời trễ một tick, và độ trễ cộng dồn qua mỗi chặng của chuỗi gọi.

---

## 7. Hàng đợi gửi, scheduler và tắc nghẽn

Net giữ một ô chờ và từ chối xếp hàng (§2 C5). Net cũng cấm ghi đè ô chờ vì ô đó có thể đang giữ một verdict handshake. Hàng đợi của fRPC nằm trên net, không thay thế cơ chế của net.

### 7.1 Một hàng đợi cho mỗi call

Một FIFO duy nhất cho cả kết nối tạo ra tắc đầu hàng:

```
   call 7 đẩy 1 000 frame ITEM vào hàng đợi
   call 8 là một unary nhỏ, đẩy vào sau
        → call 8 phải đợi hết 1 000 frame của call 7
```

Bố cục bắt buộc:

```
       Call A        Call B        Call C
       hàng đợi      hàng đợi      hàng đợi
          │             │             │
          └─────────────┼─────────────┘
                        ▼
                   SCHEDULER            xoay vòng, hạn mức theo byte
                        │
                        ▼
                    net.send            một frame mỗi lần
```

Ba bất biến:

```
   1. Thứ tự được bảo đảm TRONG từng call.
   2. Scheduler được phép xen kẽ GIỮA các call.
   3. Byte trên dây không chen giữa một frame  (B7 của net).
```

Điều cấm đảo thứ tự áp dụng trong phạm vi một call: một ERROR đi trước ITEM cuối của chính call đó sẽ đến nơi trước dữ liệu mà nó kết thúc. Nó không áp dụng giữa hai call khác nhau.

### 7.2 Hạn mức theo byte

Scheduler nhường lượt sau khi một call đã gửi đủ số byte định trước, không phải đủ số frame. Kích thước frame biến thiên lớn, nên đếm frame không phản ánh phần đường truyền đã sử dụng.

Hạn mức không cắt nhỏ được một frame. Nó xác định bao nhiêu frame của một call được gửi trước khi nhường lượt. Một frame 16 MiB chiếm ô chờ của net cho tới khi đẩy xong, và B7 cấm frame khác chen vào.

Vì vậy scheduler đi kèm một khuyến nghị:

```
   Thân của một message RPC NÊN giữ dưới ~64 KiB.
   Dữ liệu lớn hơn NÊN cắt thành stream.
   Trần 16 MiB là trần của net, không phải kích thước làm việc thông thường.
```

### 7.3 Kết thúc call thì vứt hàng đợi của call đó

Khi một call kết thúc, các frame còn xếp trong hàng đợi của call đó PHẢI bị vứt, sau đó frame kết thúc được đẩy vào.

Vứt hàng đợi không phải đảo thứ tự: call đã kết thúc. Không có quy tắc này thì một CANCEL chỉ có tác dụng sau khi toàn bộ hàng đợi của call được đẩy hết.

Frame kết thúc (RESPONSE, END của bên bị gọi, ERROR) và CANCEL KHÔNG ĐƯỢC bị từ chối vì trần cả kết nối ở §4.8. Mỗi call có tối đa một frame như vậy, nên bộ nhớ vẫn có giới hạn. Nếu frame kết thúc bị bỏ vì tắc nghẽn, bên kia chờ tới deadline cho một call đã xong. ITEM thì ngược lại: chạm trần cả kết nối thì bên sản xuất chờ tick sau, không mất item.

Trường hợp biên: REQUEST chưa rời hàng đợi thì hủy là vứt im lặng, không gửi gì. Bên kia chưa nhận được call đó, nên một CANCEL gửi đi sẽ bị bỏ qua ở đầu kia (§5.9). Call kết thúc cục bộ với `CANCELLED` và không có byte nào lên dây.

### 7.4 Tắc nghẽn

```
   ứng dụng gọi        → fRPC dựng frame, đẩy vào hàng đợi của call, trả về ngay
   chạm trần           → trả lỗi tắc nghẽn, KHÔNG xếp hàng thêm
   bước 5 của tick     → scheduler rót sang net cho tới khi net báo tắc nghẽn
```

Hai quy tắc:

1. Hai trần (§4.8): một trần cho mỗi call, một trần cho tổng số frame đang xếp trên cả kết nối. Chỉ có trần mỗi call thì tổng số frame vẫn tăng theo số call.
2. Chạm trần là lỗi trả về cho bên gọi, không phải lỗi kết thúc session. Ứng dụng quyết định bỏ, giảm nhịp hay ngắt, theo `01-overview` §5.

Phạm vi của scheduler là công bằng cục bộ ở phía gửi. Scheduler không biết bên nhận tiêu thụ nhanh hay chậm; thông tin đó đến qua frame CREDIT (§5.6). Khi một call hết hạn mức, bên sản xuất của call đó dừng, còn scheduler vẫn rót các call khác như thường.

### 7.5 CREDIT là frame điều khiển

Nếu CREDIT đi chung luật với frame dữ liệu, nó có thể bị mất: hàng đợi của call đầy vì net đang tắc thì lệnh gửi CREDIT trả lỗi tắc nghẽn theo §7.4, CREDIT không lên dây, và bên sản xuất ở đầu kia dừng vô thời hạn. Vì vậy CREDIT theo ba luật riêng:

```
   1. Gộp            mỗi call, mỗi chiều có tối đa MỘT CREDIT chờ gửi.
                     CREDIT mới cộng vào CREDIT đang chờ, bão hòa ở trần UInt32.

   2. Không tính trần  CREDIT không tính vào trần mỗi call và trần cả kết nối
                     (§4.8). Nhờ luật 1, số CREDIT chờ gửi không vượt số call
                     đang mở, nên bộ nhớ vẫn có giới hạn.

   3. Đi trước       CREDIT được gửi trước các ITEM đang xếp của cùng call.
                     CREDIT nói về chiều ngược lại, không có quan hệ nhân quả
                     với ITEM đang xếp. Ngoại lệ: CREDIT KHÔNG ĐƯỢC đi trước
                     REQUEST mở call của nó.
```

Ngoại lệ ở luật 3 là bắt buộc. Bên gọi có thể cấp CREDIT cho chiều trả về khi REQUEST còn nằm trong hàng đợi. Nếu CREDIT lên dây trước, bên kia nhận CREDIT cho một call chưa tồn tại, bỏ qua nó theo mức một của §5.9, và hạn mức đó mất.

Lệnh cấp CREDIT vì vậy không bao giờ trả lỗi tắc nghẽn. Luật thứ tự của §7.1 vẫn giữ nguyên cho frame dữ liệu và frame kết thúc. Khi call kết thúc, CREDIT đang chờ bị vứt cùng hàng đợi (§7.3).

CREDIT không tạo ra vòng chờ giữa hai chiều của một bidi: CREDIT không tiêu hạn mức, và item bị dừng vì hết hạn mức không nằm trong hàng đợi vì bên sản xuất ngừng sinh item (§5.6).

---

## 8. Handler trên vòng tick

Không có luồng ngầm (§2 C6). Handler chạy bên trong bước 2 của tick. Handler chạy lâu làm vòng lặp dừng; heartbeat nằm ở bước 3 của tick net, nên khi tick dừng đủ lâu, peer gửi probe, không nhận được trả lời và kết thúc session (`01-overview` §6).

Handler có hai dạng kết quả:

```
   XONG        → giá trị trả về, hoặc lỗi. fRPC dựng RESPONSE/ERROR ngay.

   CHƯA XONG   → một mốc để fRPC gọi lại ở bước 4 của các tick sau.
                 fRPC giữ call trong pending, không gửi gì.
```

Dạng thứ hai áp dụng cho việc dài chia nhỏ qua nhiều tick và cho việc chờ một hệ thống khác. Điều kiện: trạng thái chờ đó PHẢI kiểm tra được bằng một thao tác không chặn ở mỗi tick. Không thỏa điều kiện này thì công việc đó không thuộc vòng tick; ứng dụng chạy nó ở nơi khác và nộp kết quả vào fRPC như một sự kiện.

Handler KHÔNG ĐƯỢC gọi lại vào `frpc_tick`, theo cùng lý do với điều cấm transport gọi ngược vào core ở `01-overview` §12: đệ quy vào một trạng thái đang xử lý dở.

Mục này mô tả lõi thuần. §19 định nghĩa một chế độ lái thứ hai, ở đó fRPC sở hữu luồng của mình và handler được phép chặn. Hai chế độ sinh ra byte giống hệt nhau.

---

## 9. Interceptor

Interceptor tạo thành một chuỗi chạy theo hai chiều.

```
   PHÍA BỊ GỌI                        PHÍA GỌI
   REQUEST                            call()
     ↓                                  ↓
   [1] log / metric                   [1] log / metric
     ↓                                  ↓
   [2] authentication §10             [2] gắn token
     ↓                                  ↓
   [3] authorization §10              [3] deadline
     ↓                                  ↓
   handler                            đẩy vào hàng đợi
     ↓                                  ↓
   ngược chuỗi 3 → 2 → 1              ngược chuỗi khi hoàn tất
```

Context truyền dọc chuỗi mang:

```
   call id · method (tham chiếu descriptor) · kind · danh tính session
   now (mốc của tick hiện tại) · deadline nếu có
   identity §10 · tín hiệu hủy
```

Context không chứa bảng metadata (§3.4). Dữ liệu ghi vào context chỉ tồn tại trong tiến trình và không đi qua mạng.

Năm ràng buộc:

- Interceptor chạy trong tick và chịu ràng buộc của §8: không chặn, không gọi lại vào fRPC. Khác với handler, interceptor KHÔNG có dạng CHƯA XONG; chuỗi PHẢI chạy hết trong một tick. Hệ quả: xác thực PHẢI là phép tính cục bộ, xem §10.1.
- Interceptor CÓ THỂ kết thúc call sớm bằng một lỗi. Khi đó handler không chạy và chuỗi đi ngược từ mắt xích đã chặn.
- Mọi mắt xích đã chạy bước `outgoing` hoặc `inbound` PHẢI nhận `outbound` đúng một lần, bất kể call kết thúc vì đâu: có response, có lỗi, REQUEST không vào được hàng đợi phía gọi, hết deadline, hay session kết thúc.
- Thứ tự chuỗi là cấu hình lúc khởi động và cố định trong đời session. Thay đổi giữa chừng làm hai call cùng method đi qua hai đường khác nhau.
- Phía gọi, chuỗi interceptor PHẢI chạy xong trước khi mã hóa khối ctx, vì chuỗi này là nơi gắn token, deadline và trace id.

---

## 10. Authentication và Authorization

Authentication và authorization trả lời hai câu hỏi khác nhau, và `02-flows` §3.3 quy định handshake không trả lời câu nào trong hai câu đó:

```
   Authentication   "bên kia là ai"                 → một lần mỗi session
   Authorization    "bên kia được gọi method này"   → mỗi call
```

### 10.1 Xác thực một lần mỗi session

fRPC không định nghĩa cơ chế xác thực. Nó định nghĩa vị trí để cắm cơ chế đó vào:

```
   session READY
     │
     │ bên gọi gọi một method xác thực do ứng dụng khai báo
     │   (token nằm trong request model hoặc trong FRpcCtx - §3.4)
     ▼
   bên bị gọi xác minh, gắn identity vào session
     │
     ▼
   mọi call sau đó đọc identity trong context §9
```

Trước khi có identity, interceptor authentication từ chối mọi method không nằm trong danh sách miễn trừ. Danh sách đó PHẢI khai báo tường minh.

Xác thực dùng một call thông thường thay vì một kind riêng vì cách đó dùng lại schema, fingerprint, interceptor, deadline và mã lỗi đã có, và không thêm nhánh nào vào phần đầu fRPC. Thay đổi thuật toán xác thực không thay đổi định dạng của tầng RPC.

Phép kiểm xác thực PHẢI là phép tính cục bộ. Interceptor không có dạng CHƯA XONG (§9), nên một lời gọi ra ngoài tiến trình để kiểm token sẽ chặn vòng tick; khi tick dừng đủ lâu, heartbeat ngừng và peer kết thúc session (§8).

```
   ĐƯỢC                                  KHÔNG ĐƯỢC
   ────                                  ──────────
   verify chữ ký tại chỗ                 gọi introspect token qua mạng
   tra cache danh tính trong bộ nhớ      truy vấn cơ sở dữ liệu
   so khớp một tập khóa đã nạp sẵn       chờ bất kỳ thao tác nào
```

Dữ liệu từ xa (khóa công khai xoay vòng, danh sách thu hồi, hạn mức) PHẢI được nạp nền và cập nhật vào cache, không nằm trên đường xử lý của một call. Cache chưa sẵn sàng thì từ chối bằng `UNAVAILABLE`.

Ràng buộc này giới hạn lựa chọn thuật toán: credential xác minh được bằng phép tính cục bộ (chữ ký bất đối xứng, MAC với khóa đã nạp) là phù hợp. Credential chỉ xác minh được bằng cách truy vấn hệ thống khác là không phù hợp, trừ khi kết quả truy vấn đã có trong cache.

### 10.2 Phân quyền theo method

Quyền là dữ liệu của descriptor (§12), không phải dữ liệu trên dây: mỗi method khai yêu cầu của mình, interceptor authorization đối chiếu với identity. Không đạt thì trả `PERMISSION_DENIED`; chưa có identity thì trả `UNAUTHENTICATED`. Hai mã tách biệt vì bên gọi xử lý khác nhau: một trường hợp cần xác thực lại, một trường hợp xác thực lại không thay đổi kết quả.

### 10.3 Ranh giới với TLS

```
   TLS        bảo vệ đường truyền      → transport, ngoài phạm vi Fomoxa
   auth       xác thực bên gọi         → fRPC
```

Hai cơ chế này độc lập. Một đường nội bộ giữa hai service có thể không dùng TLS mà vẫn xác thực. TLS không xác định bên kia được gọi method nào.

### 10.4 Một kết nối, một danh tính

```
   Một kết nối có ĐÚNG MỘT danh tính đã xác thực.
   Credential theo từng call, nếu có, KHÔNG ĐƯỢC thay thế danh tính đó.
```

Bốn trường hợp:

| Trạng thái session | `ctx.token` | Xử lý |
|---|---|---|
| Chưa xác thực | Có | Chỉ dùng cho method trong danh sách miễn trừ (§10.1). Method khác: `UNAUTHENTICATED` |
| Đã xác thực | Rỗng | Dùng danh tính của session |
| Đã xác thực | Cùng principal với session | Chấp nhận, không thay đổi trạng thái |
| Đã xác thực | Principal khác | `PERMISSION_DENIED`. Không đổi danh tính, không chạy handler |

Trường hợp thứ tư ngăn hai hệ quả. Thứ nhất, một kết nối xác thực bằng danh tính ít quyền có thể tự nâng quyền theo từng call. Thứ hai, bản ghi nhật ký ghi danh tính của session trong khi call chạy dưới một principal khác, làm mất khả năng đối chiếu khi truy vết.

Ủy quyền, tức một service gọi thay người dùng, vẫn thực hiện được theo một chiều:

```
   danh tính session   = bên mở kết nối này      ← nguồn quyền
   credential mỗi call = bên được gọi thay       ← chỉ THU HẸP
```

Quyền hiệu lực của một call là giao của hai tập quyền, KHÔNG BAO GIỜ là hợp. Cả hai PHẢI có mặt trong bản ghi audit.

---

## 11. Nén

Nén đặt sau codec và trước net, tức là bên trong fRPC, tại ranh giới giữa hai lời gọi mà fRPC thực hiện.

```
   model ──codec──> byte thân ──nén──> byte thân đã nén
                                            │
                    phần đầu fRPC (không nén) + thân
                                            │
                                          net.send
```

Quy tắc:

- Cờ nén là bit 7 của byte `kind` (§4.2).
- Nén chỉ áp cho model ở phần thân: request, response, item hoặc `FRpcError`. Phần đầu 5 byte và toàn bộ `[độ dài][FRpcCtx]` luôn ở dạng không nén.

```
   ┌──────────────┬──────────────┬─────────────────────┐
   │ độ dài ctx   │ FRpcCtx      │ model               │
   │ không nén    │ không nén    │ nén nếu bit 7 bật   │
   └──────────────┴──────────────┴─────────────────────┘
```

- Ranh giới này xuất phát từ thứ tự xử lý bên nhận:

```
   đọc phần đầu → đọc ctx → interceptor / auth → GIẢI NÉN → giải mã model → handler
```

  Interceptor xác thực đọc token trước khi quyết định xử lý call. Nén ctx buộc bên nhận giải nén trước khi đọc được token, tức là thực hiện phần tính toán nặng nhất cho cả những call sẽ bị từ chối.

- Nén không thay đổi mã định danh message. Message id mô tả kiểu của thân sau khi giải nén (§4.4).
- Trần sau giải nén bằng trần thân (§4.8). Bộ giải nén PHẢI kiểm tăng dần, KHÔNG ĐƯỢC cấp phát theo kích thước khai báo rồi kiểm sau. Đây là chốt chặn phân bổ cùng loại với trần net đặt cho frame handshake.
- Thân nhỏ KHÔNG NÊN nén. Ngưỡng là cấu hình, không phải giao thức.

Thuật toán nén chốt bằng cấu hình ở hai phía. Handshake của net có bố cục cố định (`02-flows` §3.2), và thêm trường vào đó là thay đổi phiên bản giao thức. Cách còn lại là thương lượng bằng một method fRPC gọi sau READY; cách này để dành ở §13.2.

Nén thay đổi tính tất định ở cấp frame, không ở cấp model. Thân sau giải nén PHẢI là đúng chuỗi byte canonical mà codec sinh. Chữ ký và hash PHẢI tính trên thân đã giải nén, KHÔNG ĐƯỢC tính trên byte trên dây.

---

## 12. Đăng ký service và descriptor

Descriptor là dữ liệu. Mỗi method khai:

```
   tên đầy đủ           "Player.Get"          (dùng để đọc và ghi nhật ký)
   loại call            unary | server stream | client stream | bidi
   kiểu request         message id + kiểu      ← định danh của method §3.3
   kiểu item chiều gọi  message id + kiểu      ← chỉ client stream và bidi §5.5
   kiểu response/item   message id + kiểu
   yêu cầu quyền        §10.2
   an toàn khi thử lại  unsafe | idempotent | keyed     §20.3
   hạn rảnh             chỉ stream, tùy chọn            §5.4
```

Tên đầy đủ không lên dây và không tham gia dispatch. Đổi tên method không thay đổi byte nào; đổi tên request model thì có, vì id sinh từ tên message.

Từ descriptor dựng được cả hai chiều: bảng dispatch của bên bị gọi và stub của bên gọi. Ba phép kiểm lúc đăng ký ở §4.10 chạy trên dữ liệu này.

Stub có kiểu là tiện lợi của API từng ngôn ngữ, thuộc hàng thứ hai của §15.1: một bản triển khai chỉ có đường gọi bằng byte vẫn liên thông từng byte với một bản có stub. Nó không nằm trong điều kiện hoàn thành của phase nào.

`fomoxac` CÓ THỂ sinh descriptor. Nó KHÔNG ĐƯỢC là điều kiện để fRPC chạy: theo vị trí của dự án, fomoxac là công cụ hỗ trợ, còn giao thức phải dựng lại được từ tài liệu. Descriptor viết tay được, và bản đặc tả phải đủ cho việc đó.

---

## 13. Ranh giới

### 13.1 Không làm, theo nguyên tắc

Các mục sau không nằm trong lộ trình. Cài đặt chúng mâu thuẫn với một quyết định ở tầng dưới.

| Không làm | Lý do |
|---|---|
| Truyền lại, sắp xếp lại, lọc trùng | Fomoxa không làm (§2 C9). Thêm vào đây là dựng lại tầng tin cậy ở sai tầng |
| Bảng metadata key/value tự do | Mâu thuẫn RFC-0001 §4.1. Nhu cầu tương ứng được đáp bằng khối ctx khai báo (§3.4, §4.11) |
| Nhiều session trên một call | Call thuộc đúng một session và kết thúc cùng session (§5.8) |
| Tự kết nối lại | `01-overview` §12 cấm ở tầng transport. Quyết định kết nối lại thuộc ứng dụng |
| Chống peer không hợp tác | Cổng ③ của handshake tin danh sách do bên kia khai; fRPC cũng vậy. Đây là giao thức giữa hai bên hợp tác |
| Sửa fomoxa-net để phục vụ fRPC | §0 |

### 13.2 Chưa làm, đã dự trù chỗ

Không mục nào dưới đây yêu cầu thay đổi bố cục phần đầu; các bit để dành ở §4.2 đã dự trù cho chúng.

| Chưa làm | Chỗ dự trù |
|---|---|
| `details` trong `FRpcError` | Nối field ở cuối, chênh lệch phiên bản hợp lệ (§4.5) |
| Health check | Là method fRPC thông thường, không cần thay đổi định dạng. Liệt kê method đã chuyển thành reflection ở §21 |
| Thương lượng thuật toán nén | Bit 7 đã có; phần thương lượng dùng một method gọi sau READY (§11) |
| ctx trên kind khác REQUEST | Bit 6 đã có; chỉ cần mở rộng luật ở §4.11 |
| Mở chiều trả về với hạn mức bằng 0 | Nối một field cờ vào cuối `FRpcCtx` để phân biệt "0" với "chưa khai" (§3.4, §5.6). Bên cũ bỏ qua cờ và gửi không giới hạn, an toàn nhờ luật item vượt hạn mức |

Kind 7 là giá trị duy nhất còn trống trong ba bit `kind`. Một kind thứ tám cần một trong ba bit để dành ở §4.2 làm bit mở rộng, không phải một thay đổi bố cục.

---

## 14. Bất biến

| # | Bất biến |
|---|---|
| R1 | Net không biết fRPC tồn tại. Không thêm loại frame, không thêm byte vào frame, không thay đổi handshake |
| R2 | Phần đầu fRPC đúng 5 byte, cố định cho mọi kind. Không có định danh method trên dây |
| R3 | Mã định danh message của frame luôn mô tả phần thân §4.4 |
| R3b | Hai method không dùng chung kiểu request, và tập id của fRPC rời khỏi tập id message thường §4.10 |
| R4 | Lỗi ở tầng fRPC kết thúc một call, không kết thúc session. Chỉ tầng frame của net được đóng session |
| R5 | Một call kết thúc bởi đúng một frame kết thúc, và đúng một lần hoàn tất ở phía gọi |
| R6 | Session kết thúc thì mọi call đang chờ hoàn tất đúng một lần với mã `UNAVAILABLE` |
| R7 | Mọi mốc thời gian của lõi đến từ `now` truyền vào tick. Lõi không đọc đồng hồ; vỏ đọc đồng hồ và bơm vào §19 |
| R8 | Lõi không chặn; handler chưa xong thì khai CHƯA XONG. Vỏ chỉ chặn ở bước chờ giữa hai tick. Handler chỉ được chặn trên worker của chế độ self-driven §19 |
| R9 | Thân được giải mã hoặc sao chép ngay trong tick nhận §2 C7 |
| R10 | Thứ tự được bảo đảm trong từng call; scheduler được xen kẽ giữa các call; kết thúc call thì vứt hàng đợi của nó §7 |
| R11 | Khối ctx luôn có tiền tố độ dài §4.11 |
| R12 | fRPC không đòi hỏi thay đổi nào trong fomoxa-net §0 |
| R13 | Một kết nối có đúng một danh tính đã xác thực. Credential theo call chỉ thu hẹp quyền §10.4 |
| R14 | Không phép kiểm nào trên đường xử lý của một call được ra khỏi tiến trình. Xác thực là phép tính cục bộ §10.1 |
| R15 | Ứng dụng chỉ nhận những message thường nó đã khai. Frame không thuộc tập nào không bao giờ tới ứng dụng §4.10 |

---

## 15. Lộ trình

Hai nguyên tắc xếp thứ tự, cả hai rút ra từ ràng buộc của Fomoxa:

- Deadline và dọn pending nằm ở phase 1. Theo §2 C9, không có tầng truyền lại bên dưới, nên deadline là cơ chế duy nhất đưa một call ra khỏi trạng thái chờ. Một bản unary không có deadline giữ lại các bản ghi pending vô thời hạn.
- Model của method lên sớm. `fomoxac` đã sinh mã định danh, fingerprint và codec cho mọi model, và từ chối sinh mã khi hai mã định danh đụng nhau (`SPEC-FINGERPRINT.md` §4). fRPC dùng lại nguyên cơ chế đó thay vì định nghĩa một tầng định danh riêng ở trên. Phần còn lại (hình thái lời gọi, tên method, quyền) là chính sách chứ không phải hình dạng byte, nên nó khai tại chỗ đăng ký chứ không sinh ra từ schema.

| Phase | Nội dung | Điều kiện hoàn thành |
|---|---|---|
| 1 | Phần đầu, call id, dispatch, unary, `FRpcError`, deadline, hủy pending khi session kết thúc | Hai SDK khác ngôn ngữ gọi unary qua lại; một call không có câu trả lời vẫn kết thúc và giải phóng tài nguyên |
| 2 | Hàng đợi gửi, scheduler, tắc nghẽn, hai chế độ lái §19 | Gửi dồn không làm tăng bộ nhớ không giới hạn; handler dài không làm mất kết nối |
| 3 | CANCEL, server streaming, END | Stream hủy giữa chừng không để lại call trong pending ở hai phía |
| 4 | Model của method và các model hệ thống khai trong schema; id, fingerprint và codec lấy từ `fomoxac` | Thêm một method chỉ cần thêm model và một dòng đăng ký; không mã định danh nào viết tay |
| 5 | Khối ctx, deadline lan truyền, interceptor | Trace id đi hết một chuỗi ba chặng; chặng cuối nhận được ngân sách còn lại |
| 6 | Identity, authentication, authorization | Method không miễn trừ bị chặn khi chưa xác thực |
| 7 | `RetryPolicy` + `idempotency_key`, nén, client/bidi stream, flow control bằng credit | §20, §5.5, §5.6 |
| 8 | Reflection | Một công cụ bên ngoài, không có mã nguồn service, liệt kê được method và gọi được một method §21 |

Tiêu chí kiểm tra của cả lộ trình: một thay đổi schema không tương thích bị chặn tại thời điểm kết nối, và byte sinh ra giống nhau giữa hai SDK.

### 15.1 Phải chốt trước bản triển khai thứ hai

| Loại | Gồm | Lý do |
|---|---|---|
| Quy phạm, ràng buộc liên thông | Ba mức bất thường §5.9 · luật call id · ranh giới nén/ctx §11 · bố cục phần đầu và khối ctx · trạng thái hạn mức và luật item vượt hạn mức §5.6 · CREDIT không bị mất vì trần hàng đợi §7.5 | Xác định cách hai bên diễn giải cùng một chuỗi byte. Mỗi SDK tự quyết định là mất liên thông |
| Chất lượng cài đặt, không ràng buộc liên thông | Scheduler §7.1 tới §7.4 · hai chế độ lái §19 · `RetryPolicy` §20 · hạn rảnh của stream §5.4 | Một bản cài đặt FIFO đơn giản liên thông được với một bản có scheduler; byte trên dây giống nhau. Kiểm bằng test hành vi |

Hệ quả: §5.9, §5.6 và §11 là quy phạm, kể cả với bản triển khai không bật nén hay flow control, vì bên kia vẫn có thể gửi frame nén hoặc CREDIT. §7 triển khai dần được.

---

## 16. Danh sách kiểm tra tối thiểu

Theo mô hình của RFC-0003: một bản triển khai được coi là đúng khi qua hết các phép thử sau. Các phép thử không phụ thuộc ngôn ngữ.

### Định dạng
- Mã hóa rồi giải mã lại từng kind, khớp từng byte với §4.9
- Phần đầu đúng 5 byte, kể cả với END và CANCEL
- `kind` = 7 → ERROR cho call, session vẫn hoạt động
- Bit để dành khác 0 → ERROR cho call, session vẫn hoạt động
- Message id sai kiểu cho cặp `(method, kind)` → `INVALID_ARGUMENT`, session vẫn hoạt động

### Đăng ký
- Hai method khai cùng một kiểu request → lỗi lúc khởi động
- Một kiểu trong tập id fRPC trùng id một message thường → lỗi lúc khởi động
- Message id thuộc tập message thường đã khai → giao cho ứng dụng, không đọc 5 byte đầu như phần đầu fRPC
- REQUEST mang id không thuộc tập nào → `UNIMPLEMENTED` ngay, ứng dụng không nhận frame nào
- Frame mang id không thuộc tập nào và phần đầu hỏng → bỏ, không trả lời, ứng dụng không nhận frame nào

### Khối ctx
- Bit 6 tắt → request model bắt đầu tại byte 5, không có tiền tố độ dài
- Bit 6 bật, bên gửi có `FRpcCtx` nhiều field hơn bên nhận biết → request model vẫn giải mã đúng
- Độ dài ctx vượt 64 KiB → lỗi trước khi phân bổ
- Bit 6 bật trên kind khác REQUEST → `INTERNAL` cho call, session vẫn hoạt động
- `deadline_ms` qua ba chặng A→B→C: giá trị C nhận được nhỏ hơn giá trị A gửi
- `deadline_ms = 0` → không đặt deadline §3.4

### Xác thực
- Session chưa xác thực, gọi method không miễn trừ → `UNAUTHENTICATED`
- Session đã xác thực, `ctx.token` mang principal khác → `PERMISSION_DENIED`, handler không chạy, danh tính session không đổi §10.4
- Quyền hiệu lực của call có ủy quyền là giao của hai tập quyền
- Cắt đường mạng tới hệ thống cấp token: call trả lời được từ cache hoặc trả `UNAVAILABLE`; vòng tick không dừng §10.1

### Call
- Call id chẵn/lẻ đúng vai
- Tái dùng call id đang pending → lỗi tại chỗ, không lên dây
- RESPONSE cho call id không còn pending → bỏ qua
- REQUEST mang kiểu không có handler → `UNIMPLEMENTED`
- Chạm trần pending → `RESOURCE_EXHAUSTED`, các call khác không bị ảnh hưởng
- REQUEST trùng call id đang chạy → bỏ qua, call gốc tiếp tục
- Call bị từ chối cục bộ sau khi chuỗi phía gọi đã chạy → chuỗi đi ngược đúng một lần
- Call id sai chẵn/lẻ → `INVALID_ARGUMENT`, session vẫn hoạt động
- `FRpcError` giải mã hỏng → call kết thúc với `INTERNAL`

### Ba mức bất thường
- RESPONSE cho call id không còn pending → bỏ qua, không sinh lỗi
- ITEM đến sau END → bỏ qua
- CANCEL cho call đã xong → bỏ qua
- Mọi vi phạm giao thức fRPC → session vẫn hoạt động, các call khác vẫn chạy

### Hàng đợi và scheduler
- Call A xếp 1 000 ITEM, call B đẩy một REQUEST sau đó → B không đợi hết A
- Call gửi frame lớn và call gửi frame nhỏ chia đường truyền theo byte, không theo số frame
- Call bị CANCEL → các frame còn xếp bị vứt, chỉ frame kết thúc đi ra
- Hủy khi REQUEST còn trong hàng đợi → không byte nào lên dây, call kết thúc `CANCELLED`
- Hàng đợi một call đầy: dưới ngưỡng → bên sản xuất chờ tick sau; quá ngưỡng → stream kết thúc `RESOURCE_EXHAUSTED`
- Chạm trần mỗi call → chỉ call đó tắc nghẽn
- Chạm trần cả kết nối → lệnh gửi trả tắc nghẽn, session vẫn hoạt động

### Streaming hai chiều và credit
- Client stream: ITEM của bên gọi vào handler theo thứ tự, RESPONSE chỉ ra sau END của bên gọi
- ITEM của bên gọi mang mã định danh sai → `INVALID_ARGUMENT`, call kết thúc
- Gửi thêm item sau khi đã đóng chiều gọi → lỗi tại chỗ, không byte nào lên dây
- Bidi: hai chiều chạy xen kẽ, END mỗi chiều độc lập
- `initial_credit = 0` → chiều trả về chưa lập, bên sản xuất không bị giới hạn
- `initial_credit = n` → đúng n item ra rồi dừng, call vẫn mở
- CREDIT tới → đúng số item được cấp chạy tiếp, đánh số liên tục với đoạn trước
- `initial_credit = 0`, rồi CREDIT n → chiều trả về chuyển sang đã lập, bên sản xuất dừng sau n item
- Client stream, bên bị gọi gửi CREDIT n trong tick nhận REQUEST → bên gọi dừng sau n item của chiều gọi
- Item tới vượt hạn mức → giao như item bình thường, không sinh lỗi
- CREDIT cho call đã đóng → bỏ qua
- Hạn mức bằng 0 kéo dài → không kích hoạt ngưỡng `RESOURCE_EXHAUSTED` của §5.4
- CREDIT gửi tới bên không khai `FRpcCredit` → bị bỏ qua, call vẫn chạy không giới hạn, ứng dụng không nhận frame nào
- CREDIT mang mã định danh không phải `FRpcCredit` → `INVALID_ARGUMENT` cho call, session vẫn hoạt động
- Hàng đợi của call đầy, cấp thêm CREDIT → không lỗi tắc nghẽn; hai lần cấp liên tiếp lên dây thành một CREDIT mang tổng

### Nén
- Bit 7 bật kèm bit 6: `[độ dài][FRpcCtx]` đọc được không cần giải nén
- Thân khai giãn vượt trần → lỗi trước khi cấp phát

### Thời gian
- Chu kỳ deadline chạy hết bằng cách bơm `now`
- RESPONSE và deadline rơi cùng một tick → RESPONSE được ưu tiên (§6)
- Unary, CANCEL không tới nơi → bên bị gọi vẫn kết thúc handler bằng hạn riêng
- Stream với `deadline_ms = 0`, im lặng ở cả hai chiều lâu hơn hạn handler của unary → call vẫn mở
- Bước chờ khi net tắc → không quay vòng; vòng lặp tỉnh dậy khi transport ghi được hoặc có dữ liệu đến

### Retry
- Method `unsafe` nhận `UNAVAILABLE` → không thử lại
- Method `keyed`: các lần thử lại mang cùng `idempotency_key`; hai thao tác khác định danh mang hai khóa khác nhau
- Thử lại trừ vào ngân sách deadline của lần gọi đầu

### Kết thúc
- DISCONNECT khi có N call chờ → đúng N lần hoàn tất, mỗi call một lần
- HANDSHAKE FAILED → như trên, không có lần dọn thứ hai
- Handler đang chạy khi session kết thúc → nhận tín hiệu hủy, không frame nào được gửi

### Reflection
- Phản hồi liệt kê đúng mọi method bên bị gọi phục vụ, kèm hình thái và các message id
- `schema_json` khớp từng byte với tệp đã nhúng
- Session chưa xác thực gọi `FRpc.Reflect` → `UNAUTHENTICATED`, trừ khi nhà vận hành tự đưa method này vào danh sách miễn trừ
- Greeting chỉ gồm các model cần cho reflection → handshake chấp nhận, reflection trả lời
- Server không bật reflection, dù schema của nó có hay không có model reflection → `UNIMPLEMENTED` ngay, ứng dụng không nhận frame nào

### Liên thông
- Client ngôn ngữ A với server ngôn ngữ B, unary và stream, khớp từng byte
- Một bên nối thêm field vào cuối request model → handshake chấp nhận, call vẫn chạy (RFC-0002 §9.1)

---

## 17. Phương án đã cân nhắc và bị loại

| Phương án | Lý do loại |
|---|---|
| Envelope model: một message id duy nhất cho mọi RPC, thân là field `Bytes` bên trong | Handshake chỉ còn xác minh envelope, không xác minh request/response model. Envelope có `n` cố định, nên nối field vào body làm đổi fingerprint mà `n` không đổi, rơi vào nhánh ⓑ của `02-flows` §3.3 và bị từ chối session. Chênh lệch phiên bản không còn áp dụng được |
| Phần đầu RPC là field dẫn đầu của mọi model | Giữ được thân là một model canonical trọn vẹn, nhưng buộc codegen sinh một model gói cho mỗi method và nhân bản kiểu khi hai method dùng chung request. Phần đầu là khung của tầng, không phải dữ liệu nghiệp vụ; net đã có tiền lệ đặt khung ngoài schema (`'F' 'O'`, độ dài) |
| Thêm `method id` 4 byte vào phần đầu | Chỉ cho phép hai method dùng chung một kiểu request, với chi phí 4 byte trên mọi frame để lặp lại thông tin đã có trong frame. Dùng chung kiểu request làm hai method phụ thuộc nhau khi tiến hóa (§3.3) |
| Deadline trong phần đầu | 4 byte trên mọi frame, kể cả các kind không có khái niệm deadline. Bên bị gọi vẫn cần hạn riêng (§5.3) |
| Bảng metadata key/value tự do | Dữ liệu không khai báo, không có trong schema, handshake không xác minh được, nên mất tính chất ở dòng đầu bảng §1.1. Nhu cầu tương ứng được đáp bằng khối ctx khai báo (§3.4) |
| `FRpcCtx` làm field đầu của mọi request model | Buộc mọi method lệch phiên bản cùng lúc mỗi khi ctx đổi (§4.11) |
| Deadline là mốc thời gian tuyệt đối | Chỉ có nghĩa khi hai máy đồng bộ giờ. Fomoxa cấm đồng hồ treo tường ở mọi tầng (B9) |
| Kind riêng cho handshake của fRPC | Trùng chức năng với một method thông thường gọi sau READY, vốn đã có schema, interceptor, deadline và mã lỗi |
| Ưu tiên ERROR/CANCEL trong cùng một call | Phá quan hệ nhân quả với dữ liệu mà nó kết thúc. Vấn đề cancel chậm được giải bằng cách vứt hàng đợi của call đã kết thúc (§7.3) |
| Một FIFO duy nhất cho cả kết nối | Một stream lớn chặn mọi call khác (§7.1). Điều cấm đảo thứ tự chỉ áp trong một call |
| CREDIT chỉ cộng; hạn mức chưa lập thì không giới hạn mãi | Chiều gọi của client stream và bidi không bao giờ giới hạn được, nên upload lớn không có áp lực ngược (§5.6) |
| Chiều gọi bắt đầu với hạn mức 0, chờ CREDIT | Bên bị gọi không cài flow control không bao giờ gửi CREDIT, nên call treo tới deadline. Mất tương thích (§5.6) |
| `initial_credit = 0` là hạn mức bằng 0 | Mọi bên gọi dùng ctx chỉ để đặt deadline hay trace, và mọi bên gửi `FRpcCtx` cũ, đều mã hóa 0; stream của các bên đó đứng ngay từ đầu. Trái quy ước "chưa khai" của §3.4 |
| Hạn tổng bắt buộc cho stream | Stream theo dõi khỏe mạnh bị cắt định kỳ. Lý do của hạn bắt buộc là CANCEL có thể mất, điều chỉ xảy ra trên transport kiểu gói, nơi stream không được định nghĩa (§5.4, §5.7) |
| Khóa idempotency ngẫu nhiên mỗi lần gọi | Mất tính chất mọi SDK sinh cùng một khóa. Định danh thao tác trong request model giữ được tính tất định (§20.4) |

---

## 18. Còn phải chốt

Các điểm bản thiết kế chọn một hướng nhưng chưa có đủ dữ kiện:

1. Call id 32 bit (§4.7). Đủ cho các kịch bản đã xét; quy tắc quay vòng phải cài đúng ở cả hai phía.
2. Trần pending 65 536 mỗi chiều (§4.8). Cần số liệu từ tải thực tế.
3. Tập field của `FRpcCtx` (§3.4). Bảy field hiện tại là phỏng đoán. Nối field ở cuối là nâng cấp hợp lệ, bỏ field thì không, nên khai thiếu an toàn hơn khai thừa. `idempotency_key` và `initial_credit` là hai field nối thêm sau bản đầu, theo đúng đường đó.
4. Method không tham số phải khai model rỗng riêng (§3.3). Phase 4 xử lý bằng codegen. Nếu vẫn còn chi phí đáng kể sau đó, chỗ cần sửa là codegen, không phải định dạng trên dây.
5. Unary trên transport kiểu gói (§5.7). Bản thiết kế chấp nhận khả năng lặp ở phía bên bị gọi và giao việc chống lặp cho ứng dụng. Nếu mọi ứng dụng đều phải tự cài phần đó, nó thuộc về fRPC.
6. Nén chốt bằng cấu hình (§11). Áp dụng được khi hai phía do cùng một bên triển khai; không áp dụng được khi có bên thứ ba.

---

## 19. Hai chế độ lái

§8 quy định handler không được chặn vòng tick. Ràng buộc đó chỉ lộ ra API khi vòng tick là API.

Net đã tách máy trạng thái thuần (nhận frame và `now`, trả ra ý định) khỏi phần nối với transport. fRPC áp dụng cùng đường cắt, cao hơn một tầng:

```
   LÕI fRPC THUẦN          nhận sự kiện + now, trả ra frame và kết quả call
                           không luồng, không đồng hồ, không I/O
        │
   ─────┼──────────────────────────────────────────────
        │
   VỎ    ├─ app-driven   ứng dụng gọi tick(now) trong vòng lặp của mình
         └─ self-driven  fRPC sở hữu một luồng, tự chạy vòng lặp
```

| | App-driven | Self-driven |
|---|---|---|
| Ai gọi `tick(now)` | Ứng dụng | Luồng của fRPC |
| Handler | Chạy tuần tự trên luồng lái | Được phép chặn, trên worker riêng |
| API của bên gọi | Lấy kết quả ở tick sau | Chặn luồng gọi cho tới khi xong |
| Phù hợp với | Test xác định, C và Rust không runtime, ứng dụng đã có vòng lặp riêng | Service thông thường |

Điều cấm tự sinh luồng ngầm ở `01-overview` §12 nằm trong bảng dành cho transport và không áp cho tầng trên core. Ràng buộc áp dụng là câu đi kèm: một session do một luồng lái. Chế độ self-driven thỏa ràng buộc đó khi giữ đúng năm quy tắc:

```
1. Đúng một luồng chạm vào session. Mọi thứ đi vào là message
   trên hàng đợi CÓ TRẦN (§4.8), không phải lời gọi trực tiếp.

2. Lõi thuần vẫn PHẢI gọi được với `now` bơm vào. Vỏ self-driven
   dùng đồng hồ thật; bỏ B8 là mất khả năng chạy hết một chu kỳ
   deadline trong kiểm thử.

3. Giải mã thân trên luồng lái, chỉ trao giá trị đã sở hữu qua
   hàng đợi. Truyền một lát byte mượn sang luồng khác là đọc bộ
   nhớ đang bị ghi đè (§2 C7).

4. Handler chạy song song PHẢI khai báo theo từng method, không
   mặc định. Ở chế độ tuần tự, handler không cần khóa khi chạm
   trạng thái dùng chung; bật song song mặc định đưa tranh chấp
   vào mã không được viết cho nó.

5. Mỗi session mang một số thế hệ. Kết quả của handler trả về sau
   khi session kết thúc PHẢI bị vứt, không được gửi (§5.8).
```

### Bước chờ giữa hai tick

Vòng tick quy định thứ tự bên trong một tick; nó không quy định khoảng cách giữa hai tick. Vỏ tự chọn khoảng cách đó, và lựa chọn này quyết định hai đại lượng cùng lúc: độ trễ của một call và chi phí CPU của một kết nối rảnh.

Lấy nhịp cố định làm cả hai đại lượng xấu đi cùng nhau: nhịp ngắn tốn CPU khi rảnh, nhịp dài cộng thêm nửa chu kỳ vào mỗi chặng của chuỗi gọi. Vỏ KHÔNG NÊN dùng nhịp cố định khi transport có tín hiệu sẵn sàng.

Thay vào đó, vỏ chặn cho tới khi xảy ra một trong các sự kiện: dữ liệu đến, transport ghi được trở lại, kết nối đóng, hoặc mốc thời gian gần nhất mà lõi đang chờ đã tới. Bước 5 của tick rót hàng đợi gửi cho tới khi hết hoặc net báo tắc nghẽn (§6), nên sau một tick, frame còn lại trong hàng đợi luôn có nghĩa là net đang tắc. Điều kiện chờ suy ra từ lõi:

```
   còn frame chờ gửi (net đang tắc)  → chờ đọc được HOẶC ghi được
   còn handler chưa xong             → chờ tối đa một khoảng ngắn, để gọi lại §8
   còn deadline hoặc hạn rảnh        → chờ tối đa tới mốc gần nhất §5.3, §5.4
   không còn gì                      → chờ tối đa tới trần rảnh
```

Các dòng cộng dồn: hạn chờ là mốc sớm nhất trong các dòng áp dụng. Chờ đọc được luôn có mặt, kể cả khi đang chờ ghi được; một bên chỉ chờ ghi mà không đọc có thể cùng bên kia đợi bộ đệm của nhau vơi đi. Transport không báo được trạng thái ghi được thì vỏ thay điều kiện đó bằng một khoảng chờ ngắn cố định. Không dòng nào dẫn tới vòng lặp không chờ: vòng lặp như vậy chiếm trọn một nhân CPU trong suốt thời gian net tắc.

Trần rảnh PHẢI nhỏ hơn chu kỳ heartbeat của net, vì probe được gửi ở bước 3 của tick net (`01-overview` §6).

Điều cấm chặn ở `01-overview` §12 vẫn giữ nguyên: bốn hàm của transport (gửi, nhận, đóng mềm, đóng hẳn) KHÔNG ĐƯỢC chặn. Bước chờ là một hàm thứ năm, do vỏ gọi giữa hai tick, không do net gọi. Transport cài thêm hàm này vẫn thỏa §12.

Số luồng là chính sách của vỏ, không phải của giao thức. Một luồng cho mỗi kết nối cho phép chặn trực tiếp trên socket của kết nối đó và không cần cơ chế ghép kênh; trần thực tế của cách này nằm ở khoảng vài nghìn kết nối cho mỗi tiến trình. Vượt trần đó thì vỏ dùng một reactor ghép kênh. Cả hai cách sinh ra byte giống hệt nhau và không đổi lõi.

Thứ tự tuyệt đối trên dây (`01-overview` B7) áp cho từng kết nối: mỗi kết nối PHẢI có đúng một luồng ghi. Song song nằm giữa các kết nối, không nằm bên trong một kết nối.

Hai chế độ sinh ra byte giống hệt nhau trên dây: không kind mới, không bit mới, không thay đổi handshake. Vì vậy chúng không thuộc đặc tả fRPC mà thuộc tài liệu ràng buộc API của từng ngôn ngữ, theo phân định ở `01-overview`: những tài liệu đó mô tả cách gọi, không mô tả hành vi.

Chế độ app-driven PHẢI được hỗ trợ đầy đủ. Yêu cầu một runtime để chạy fRPC giới hạn số ngôn ngữ dựng lại được nó, mâu thuẫn với RFC-0001 §6.5.

Giới hạn cần ghi nhận: vòng tick chỉ báo hủy, không dừng được handler đang chạy. Hạn xử lý của bên bị gọi (§5.3) thông báo cho handler rằng đã quá hạn; nó không dừng một vòng lặp đang chạy trên worker. Handler không đọc tín hiệu hủy sẽ chạy tới khi tự kết thúc.

---

## 20. Retry

### 20.1 Lõi không thử lại

`01-overview` §7 quy định Fomoxa không truyền lại. fRPC giữ nguyên quy định đó ở tầng RPC:

```
   client ── Purchase() ──> backend
                             giao dịch thành công
                             gửi response
                        X    kết nối kết thúc
   client nhận UNAVAILABLE
        → không xác định được backend đã thực thi hay chưa
```

Lõi không có thông tin để quyết định thử lại có an toàn hay không, nên nó không quyết định. Kết nối kết thúc thì mọi call đang chờ hoàn tất với `UNAVAILABLE` (§5.8).

### 20.2 Stub phía client cung cấp retry

Kết nối sống lâu (§1.2) vẫn kết thúc khi deploy hoặc khi mạng lỗi, và mỗi lần như vậy mọi call đang chờ hoàn tất với `UNAVAILABLE`. Retry thuộc stub phía client, nằm trên lõi:

```
   ứng dụng   ── cung cấp kết nối, quyết định kết nối lại
      │
   RpcClient  ── RetryPolicy: số lần · backoff · mã được thử lại · ngân sách
      │
   lõi fRPC   ── không cài đặt retry
```

Stub thử lại trên kết nối do ứng dụng cung cấp. Khi kết nối cũ đã kết thúc, stub xin ứng dụng một kết nối, và ứng dụng quyết định mở kết nối mới hay trả lỗi. Stub không tự kết nối lại, theo §13.1.

### 20.3 Bốn quy tắc

Thử lại PHẢI trừ vào ngân sách deadline. Thử lại với deadline mới làm `deadline_ms` lan truyền ở §5.3 mất ý nghĩa ở mọi chặng phía sau. Ngân sách thuộc cả chuỗi thử lại, không thuộc từng lần.

Mỗi method khai mức an toàn khi thử lại trong descriptor (§12):

| Mức | Nghĩa | Ví dụ |
|---|---|---|
| `unsafe` | Mặc định. Chạy hai lần cho kết quả khác chạy một lần | `Purchase` không có định danh thao tác |
| `idempotent` | Chạy hai lần cho cùng trạng thái như chạy một lần, tự bản chất của thao tác | Đọc; `SetStatus(user, status)` |
| `keyed` | Thao tác có tác dụng phụ, bên bị gọi lọc trùng bằng `idempotency_key` (§20.4) | `Purchase(order_id, item)` |

Mức mặc định là `unsafe` vì khai sai theo hướng này chỉ làm mất một lần thử lại, còn khai sai theo hướng kia làm một giao dịch chạy hai lần.

Mã lỗi quyết định có nên thử lại hay không:

| Mã | Thử lại | Lý do |
|---|---|---|
| `UNAVAILABLE` | Chỉ với method `idempotent` hoặc `keyed` | Kết nối kết thúc, không xác định được bên bị gọi đã thực thi hay chưa §20.1 |
| `RESOURCE_EXHAUSTED` | Có, kèm backoff | Quá tải tạm thời |
| `DEADLINE_EXCEEDED` | Không | Ngân sách đã hết |
| `INVALID_ARGUMENT`, `PERMISSION_DENIED`, `UNAUTHENTICATED`, `UNIMPLEMENTED`, `FAILED_PRECONDITION` | Không | Gửi lại cùng dữ liệu cho cùng kết quả |
| `INTERNAL` | Tùy chính sách | Không đủ thông tin để kết luận |
| `CANCELLED` | Không | Bên gọi đã hủy |

Stream KHÔNG ĐƯỢC tự phát lại. Nếu một stream đứt sau khi bên gọi đã tiêu thụ N item, phát lại từ đầu sẽ lặp lại N item đó. Nối tiếp từ điểm đứt cần một con trỏ tiếp tục, và con trỏ đó là khái niệm của ứng dụng.

### 20.4 Idempotency

`UNAVAILABLE` không xác định được giao dịch đã thực thi hay chưa. Cơ chế xử lý là một khóa idempotency:

```
   khóa = băm của thân đã mã hóa
```

Vì byte tất định (RFC-0001 §6.1), mọi SDK ở mọi ngôn ngữ sinh ra cùng một khóa mà không cần thỏa thuận bổ sung. Một hệ thống RPC dựa trên format không tất định không có tính chất này.

Khóa băm thân chỉ đúng khi cùng thân nghĩa là cùng một thao tác. Hai lần mua cùng một món một cách cố ý có thân giống nhau, và lần thứ hai sẽ bị lọc như một lần thử lại. Vì vậy request model của method `keyed` PHẢI chứa một định danh thao tác do bên gọi chọn, ví dụ `order_id`. Các lần thử lại của cùng một thao tác mang cùng định danh nên cùng khóa; hai thao tác khác nhau mang hai định danh nên hai khóa. Sự phân biệt nằm trong dữ liệu có khai báo, không nằm trong một giá trị ngầm ngoài schema (§3.4).

Method `idempotent` không cần khóa. Stub chỉ đặt `idempotency_key` cho method `keyed`.

`idempotency_key` là field thứ sáu của `FRpcCtx` (§3.4), nối vào sau bản đầu cùng `RetryPolicy`. Bên nhận không biết field này dừng ở field năm và coi phần dư là byte thừa hợp lệ theo RFC-0002 §9.1; call vẫn chạy, chỉ mất khả năng lọc trùng.

---

## 21. Reflection

Reflection cho một công cụ bên ngoài, ví dụ một client dòng lệnh, học được method và model của một server mà không cần mã nguồn của service đó.

Handshake không cung cấp được thông tin này. Client không bao giờ thấy schema của server (`02-flows` §3.1), và kể cả thấy thì greeting chỉ mang message id, số field và một fingerprint 8 byte cho mỗi message. Fingerprint là hàm băm một chiều, không suy ngược ra tên hay kiểu field. Muốn mã hóa một request và giải mã một response, công cụ cần chính định nghĩa của model.

### 21.1 Method và model

```
   FRpc.Reflect      unary      FRpcReflectRequest → FRpcReflectResponse

   FRpcReflectRequest
     (không field)

   FRpcReflectResponse
     methods          Array<FRpcMethodInfo>
     schema_json      Bytes

   FRpcMethodInfo
     name             String     "Player.Get"
     shape            UInt32     0 unary · 1 server stream · 2 client stream · 3 bidi
     request_id       UInt32
     caller_item_id   UInt32
     reply_id         UInt32     response, hoặc item của chiều trả về
     retry_safety     UInt32     0 unsafe · 1 idempotent · 2 keyed (§20.3)
```

Ba model này là model hệ thống: khai với codec `rpc` (§4.5), và tên model, tên field, kiểu, thứ tự PHẢI đúng như trên. Fingerprint băm cả tên field, nên chỉ cần lệch một tên là công cụ không tính ra đúng fingerprint.

### 21.2 Quy tắc

- Reflection là tùy chọn. Server bật reflection thì PHẢI phục vụ đúng bố cục ở §21.1. Server không bật thì trả `UNIMPLEMENTED`, như với mọi REQUEST mà nó không phục vụ (§4.10, §5.2), dù schema của nó có khai model reflection hay không. Frame không bao giờ tới tầng ứng dụng.
- `methods` liệt kê mọi method bên bị gọi phục vụ, kể cả `FRpc.Reflect`. Method chỉ khai để gọi đi mà không phục vụ thì không liệt kê.
- `shape` và `retry_safety` khai `UInt32`, không khai Enum, cùng lý do với mã lỗi (§2 C13). Công cụ gặp giá trị lạ thì hiển thị nguyên số.
- `caller_item_id` bằng `request_id` khi method không khai kiểu item riêng cho chiều gọi (§5.5).
- `schema_json` là nội dung tệp schema mà công cụ sinh mã đã ghi cho chính binary đang chạy (với `fomoxac` là `.fomoxa/schema.json`), mã hóa UTF-8, nguyên văn. fRPC không diễn giải nội dung này. Tệp NÊN được nhúng vào binary lúc build để luôn khớp với mã đang chạy.
- Phản hồi không đổi trong suốt đời tiến trình. Bản triển khai NÊN mã hóa nó một lần rồi trả lại cùng một chuỗi byte.
- `FRpc.Reflect` đi qua chuỗi interceptor như mọi method, và KHÔNG ĐƯỢC mặc định nằm trong danh sách miễn trừ xác thực (§10.1). Phản hồi làm lộ tên và kiểu của mọi field; đưa nó ra ngoài phạm vi xác thực là quyết định của nhà vận hành.
- `schema_json` in có thụt dòng, khoảng 860 byte mỗi model, nên một schema lớn vượt mức 64 KiB khuyến nghị ở §7.2. Chấp nhận được vì reflection được gọi hiếm. Bản triển khai NÊN bật nén cho phản hồi này (§11).

### 21.3 Luồng của một công cụ

```
   kết nối 1   greeting: FRpcVoid, FRpcError, FRpcReflectRequest, FRpcReflectResponse
               công cụ tự tính id và fingerprint từ §21.1
               → gọi FRpc.Reflect → nhận methods + schema_json

   kết nối 2   greeting: toàn bộ schema học được
               → handshake xác minh từng message chung
               → gọi method thật
```

Kết nối 1 được chấp nhận dù greeting chỉ là một phần nhỏ schema của server: cổng ③ chỉ xét các message cả hai bên cùng có (`02-flows` §3.3). Fingerprint schema của hai bên khác nhau không phải lý do từ chối; nó chỉ là đường tắt khi hai bên khớp hoàn toàn.

Công cụ CÓ THỂ lưu schema học được và dùng lại ở các lần chạy sau, bỏ qua kết nối 1. Nếu server đổi schema sau đó, handshake của lần kết nối kế tiếp phát hiện: phần thay đổi còn tương thích thì vẫn được chấp nhận theo RFC-0002 §9.1, phần lệch thật thì bị từ chối với lý do 2. Công cụ khi đó phải tải lại schema.

Reflection không thêm kind, bit hay trường nào vào phần đầu. Nó là một method fRPC thông thường, nên R1 và R2 giữ nguyên.
