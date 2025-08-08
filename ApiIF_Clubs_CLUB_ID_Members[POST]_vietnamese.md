# api/clubs/{{CLUB_ID}}/members/{{USER_ID}}.json (DELETE)

## Khái quát tính năng

Xóa thành viên khỏi câu lạc bộ.

## Chi tiết xử lý

### Chứng thực:

Nếu có [parameter].[auth_token] thì tiến hành chứng thực từ auth_token (Tham chiếu: "ミイル共通設計書-共通処理-その他：auth_token 認証" (Miil Tài liệu thiết kế chung-Xử lý chung-Khác, phần "auth_token 認証" (Chứng thực auth_token))).

Get [auth_user].

### Validation:

#### Get thông tin câu lạc bộ:

Bảng: clubs

Điều kiện: clubs.id = [CLUB_ID]

Xem kết quả là [club]

Trường hợp không tồn tại [club] thì trả về HTTP STATUS CODE 'NOT FOUND' và kết thúc xử lý.

#### Get thông tin thành viên:

Bảng: club_members

Điều kiện: club_members.id = [CLUB_MEMBER_ID]

Xem kết quả là [club_member]

Trường hợp không tồn tại [club_member] thì trả về HTTP STATUS CODE 'NOT FOUND' và kết thúc xử lý.

### Xác nhận thành viên trước khi xóa:

Kiểm tra thành viên dựa theo "ミイル共通設計書-共通処理-部活: メンバーを削除する前の確認 (Tài liệu xử lý chung-xử lý chung-Clubs - Kiểm tra thành viên trước khi xóa)", set kết quả vào [can_leave]. Parameter: [auth_user], [club], [club_member]

Nếu [can_leave] = false thì trả về HTTP STATUS 'Forbidden', kết thúc xử lý.

### Xóa thành viên:

**<u>[[Bắt đầu transaction DB]]</u>**

Xóa thành viên dựa theo "ミイル共通設計書-共通処理-部活: メンバー削除の処理 (Tài liệu xử lý chung-xử lý chung-Clubs - Xử lý xóa thành viên)". Parameter: [club], [club_member]

**<u>[[Kết thúc transaction DB]]</u>**

Nếu xử lý thất bại, trả về HTTP STATUS CODE 'FORBIDDEN' và kết thúc xử lý.

### Trả response

Trả về data response của kết quả và HTTP STATUS CODE 'OK' và kết thúc xử lý.

## Chi tiết request

```json
{
    "auth_token": "{{AUTH_TOKEN}}"
}

Chi tiết response

{}

Control DB
club_members/delete
Tên vật lý	NotNull	Value
club_id	TRUE	[club].[id]
user_id	TRUE	[club_member].[user_id]

clubs/update club_members_count
Tên vật lý	NotNull	Value
id	TRUE	[club].[id]
club_members_count	TRUE	[memberCount] (số lượng thành viên sau khi xóa)