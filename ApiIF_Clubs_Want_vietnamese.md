# api/clubs/want.json (GET)

## Khái quát tính năng

Lấy danh sách các câu lạc bộ "muốn" (want) dựa trên trạng thái `listable` của câu lạc bộ, có phân trang.

## Chi tiết xử lý

### Chứng thực:

Không có

### Validation: (nếu có)

Không có

### Cài đặt parameter:

Set [limit] = [parameter].[limit]. Nếu trong [parameter].[limit] không có giá trị, hoặc [parameter].[limit] khác kiểu số thì set [limit] = `REQUEST_LIST_LIMIT`.

Set [lessThanClubId] = [parameter].[less_than_club_id]. Nếu không có giá trị, set [lessThanClubId] = `null`.

### Lấy danh sách ID câu lạc bộ muốn

Tìm kiếm `club_status_id` có `listable` là `true` từ bảng `ClubStatus`. Xem kết quả là [clubStatusIds].

Xây dựng các truy vấn `SELECT id FROM clubs` cho từng `club_status_id`.
Trong mỗi truy vấn, nếu [lessThanClubId] tồn tại, thêm điều kiện `AND id < :lessThanClubId`.
Mỗi truy vấn được sắp xếp `ORDER BY id DESC` và giới hạn `LIMIT :limit`.

Gộp các truy vấn con bằng `UNION`.

Thực hiện truy vấn SQL chính để lấy `id` từ các truy vấn `UNION` đã gộp, sắp xếp `ORDER BY want.id DESC` và giới hạn `LIMIT :limit`. Xem kết quả là [clubIds].

### Lấy thông tin chi tiết câu lạc bộ

Tìm kiếm các câu lạc bộ từ bảng `clubs` với điều kiện `id` nằm trong danh sách [clubIds].

Xem kết quả là [clubs].

### Cài đặt next_url

Nếu số lượng bản ghi trong [clubs] lớn hơn [limit]:

Xóa bản ghi cuối cùng của [clubs].

Set [nextUrl] = `API_SSL_URL/api/clubs/want.json?less_than_club_id=[clubs].[last_element].[id]&limit=[limit]`.

### Tạo response:

Thực hiện xử lý từng record 1 của toàn bộ [clubs], xem 1 record sẽ là 1 [club].

Thực hiện xử lý theo tài liệu "共通設計書-model-club" để tạo response cho mỗi [club].

Thêm kết quả vào [responseClubs].

Tạo object response cuối cùng bao gồm:

* `clubs`: [responseClubs].
* `next_url`: [nextUrl].

Xem kết quả là [response].

### Trả response

Trả về [response] cùng với HTTP STATUS CODE 'OK'. Kết thúc xử lý.

## Chi tiết request

Request nhận vào các tham số để tìm kiếm câu lạc bộ.

```json
{
    "limit": {{LIMIT_PER_PAGE}},
    "less_than_club_id": {{LAST_CLUB_ID}}
}

Chi tiết response
Response trả về danh sách các câu lạc bộ đã tìm kiếm được cùng với thông tin phân trang.

{
  "clubs": [response_clubs],
  "next_url": "[next_url]"
}

Control DB
Không có