# fRPC: lớp RPC của Fomoxa

[English](../README.md) · Tiếng Việt

fRPC là một lớp RPC đặt trên Fomoxa. Repo này mô tả nó ở mức khái niệm và mức byte, không kèm SDK, API hay mã nguồn.

Trạng thái: bản thiết kế, chưa phải đặc tả và chưa phải hướng dẫn triển khai. Mục 18 liệt kê các quyết định còn để ngỏ; mục 15.1 liệt kê những phần phải trở thành quy phạm trước khi có bản triển khai thứ hai.

## Bản đồ tài liệu

| Tài liệu | Nội dung | Độ dài |
|---|---|---|
| [01-design.md](01-design.md) | Vị trí trong kiến trúc, ràng buộc kế thừa, định dạng trên dây, các luồng, nhịp tick, hàng đợi, xác thực, bất biến, lộ trình, danh mục kiểm thử | ~1 250 dòng |

Thư mục `en/` là bản tiếng Anh. Các bản dịch khác, nếu có, nằm ở thư mục mã ngôn ngữ của riêng nó.

Hai tài liệu nền, đều nằm ở repo riêng:

| Repo | Phần fRPC sử dụng |
|---|---|
| `specification` (RFC-0001, RFC-0002, RFC-0003) | Cách mã hóa Model, chênh lệch phiên bản §9.1, tính tất định, quy tắc Enum |
| `implementation-guide` (`01-overview.md`, `02-flows.md`) | Ba tầng, frame DATA, handshake fingerprint, nhịp tick, ô chờ một frame, vòng đời dữ liệu sự kiện |

## Ranh giới với fomoxa-net

fRPC là repo riêng, nằm ngoài fomoxa-net, không phải nhánh mở rộng của fomoxa-net và không có trong lộ trình của nó. fRPC đứng trên specification và dùng một bản triển khai net theo đúng cách một ứng dụng dùng net.

Ranh giới này có một phép thử:

> Nếu fRPC đòi hỏi một thay đổi trong fomoxa-net thì logic đang được đặt sai tầng.

Mọi thông tin fRPC cần đều nằm trong phần dữ liệu của frame DATA, phần mà hướng dẫn triển khai định nghĩa là byte mờ đối với tầng frame. Net không thêm loại frame, byte tiêu đề hay trường handshake nào cho fRPC.

## Đọc theo công việc

| Việc cần làm | Các mục cần đọc |
|---|---|
| Hiểu fRPC đứng ở đâu và thừa hưởng những gì | §0–§2 |
| Chỉ cần byte trên dây | §4 |
| Cài đặt dispatch và đường unary | §3.3, §4.4, §4.10, §5.1, §5.2 |
| Cài đặt deadline, hủy và lan truyền ngân sách | §5.3, §6 bước 3 |
| Cài đặt streaming | §5.4, §7 |
| Hiểu hàng đợi, tắc nghẽn và scheduler | §7 |
| Viết handler | §8, §19 |
| Cài đặt authentication và authorization | §9, §10 |
| Cài đặt nén | §11, §4.11 |
| Kiểm chứng một bản triển khai | §16, sau đó §14 |
| Hiểu vì sao một lựa chọn RPC quen thuộc không được dùng | §17 |

## Tóm tắt một trang

Net xác định một chuỗi byte thuộc loại message nào. fRPC thêm hai câu hỏi:

```
   net    → chuỗi byte này thuộc loại message nào
   fRPC   → nó thuộc cuộc gọi nào, và thành phần nào xử lý nó
```

Mọi thứ đi trong phần dữ liệu của frame DATA, sau một phần đầu cố định 5 byte:

```
   ┌──────┬──────────────┐
   │ kind │ call id      │      bit 7   thân đã nén
   │ 1B   │ 4B u32 LE    │      bit 6   có khối ctx
   └──────┴──────────────┘      bit 2-0 kind (0..5)
```

Có sáu kind: REQUEST, RESPONSE, ERROR, ITEM, END, CANCEL. Chi phí cố định mỗi message là 16 byte, gồm 11 byte của net và 5 byte của fRPC.

Trên dây không có định danh method. Method được xác định bằng mã định danh message của request model, nên toàn bộ thiết kế phụ thuộc vào một quy tắc: hai method không được dùng chung một kiểu request.

Không có bảng metadata key/value. Thông tin xuyên suốt đi trong `FRpcCtx`, một model có khai báo, tùy chọn, có tiền tố độ dài, mang deadline còn lại, ngữ cảnh trace, token và tenant.

Khi đứng trên Fomoxa, fRPC có các tính chất sau và phải giữ nguyên chúng:

- Handshake của net từ chối schema lệch nhau ngay lúc kết nối, kèm mã lý do.
- Chênh lệch phiên bản (RFC-0002 §9.1) áp dụng theo từng kiểu request, nên nối một field là thay đổi hợp lệ riêng cho từng method.
- Byte tất định. Nhờ đó ký payload, sinh khóa idempotency từ nội dung và phát lại một chuỗi call đều thực hiện được mà không cần thỏa thuận thêm giữa các bản triển khai.

Các bất biến mọi bản triển khai phải giữ nằm ở §14. Hai bất biến thường bị vi phạm nhất: lỗi ở tầng fRPC kết thúc một call và không bao giờ kết thúc session; không phép kiểm nào trên đường xử lý của một call được ra khỏi tiến trình.

## Ngoài phạm vi

- Cách mã hóa message và cách tính fingerprint thuộc repo `specification`.
- Framing, handshake, heartbeat và hợp đồng với transport thuộc `implementation-guide` và net.
- Ràng buộc API theo từng ngôn ngữ, gồm cả việc chọn giữa chế độ app-driven và self-driven ở §19. Mỗi bản triển khai tự đặt tên và tự chọn hình dạng lời gọi. Hai chế độ sinh ra byte giống hệt nhau nên lựa chọn đó nằm ngoài giao thức.
- Chính sách retry, backoff và kết nối lại thuộc stub phía client nằm trên lõi (§20).

## Giấy phép

CC BY 4.0, xem [LICENSE](../LICENSE). Các bản triển khai phần mềm là dự án độc lập và chọn giấy phép riêng của chúng.
