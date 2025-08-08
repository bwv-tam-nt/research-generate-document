# update_hot_home_eatout_all_venues

## Khái quát tính năng

Batch này chịu trách nhiệm cập nhật các ảnh "hot" cho các danh mục "Home", "Eatout" và "All" liên quan đến địa điểm (venue). Batch này tính toán điểm số dựa trên lượt yêu thích (favorites) và bình luận (comments) trong vòng 1 ngày gần nhất, sau đó xác định các ảnh hot nhất cho từng danh mục và địa điểm cụ thể để nhập vào hệ thống.

## Giải thích

### Về xuất log

Tham chiếu '共通設計書-共通処理-バッチ処理: ログ出力について' (Tài liệu xử lý chung - xử lý batch - Về xuất log)

### Khi lỗi

Tham chiếu '共通設計書-共通処理-バッチ処理: エラー時' (Tài liệu xử lý chung - xử lý batch - Xử lý khi lỗi)

### Parameter khi khởi động

Không có

## Chi tiết xử lý

### Khởi động batch

[batch_name] = "update_hot_home_eatout_all_venues"

Set ngày giờ chạy (yyyy-mm-dd hh:mm:ss) batch vào [started_time]

**LOG:: [batch_name] + " start " + [started_time] + " : [args]=" + (xuất toàn bộ nội dung [args]. Trường hợp không có giá trị thì xuất "null")**

### Tạo bảng tạm và tính điểm số cho ảnh

Thực hiện tạo một bảng tạm `tt_venue_scores` trong cơ sở dữ liệu để lưu trữ điểm số của các bức ảnh.

```sql
CREATE TEMPORARY TABLE IF NOT EXISTS tt_venue_scores (
  id int(11) AUTO_INCREMENT PRIMARY KEY,
  photo_id int(11),
  score int(11),
  user_id int(11),
  venue_id int(11),
  category_id int(11),
  recipe_id int(11)
) ENGINE=MEMORY DEFAULT CHARSET=utf8;
Thực hiện chèn dữ liệu vào bảng tt_venue_scores bằng cách tính tổng điểm từ lượt yêu thích (favorites) và bình luận (comments) trong 1 ngày gần nhất. Điểm bình luận được nhân với COMMENT_WEIGHT.

SQL

INSERT INTO tt_venue_scores (photo_id, score, user_id, venue_id, category_id, recipe_id)
(SELECT t.photo_id, SUM(t.cnt) AS score, p.user_id, p.venue_id, p.category_id, p.recipe_id
FROM (
  SELECT photo_id, COUNT(*) AS cnt
  FROM favorites
  WHERE updated_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
  GROUP BY photo_id

  UNION ALL

  SELECT photo_id, (COUNT(DISTINCT user_id) * :commentWeight) AS cnt
  FROM comments
  WHERE updated_at >= DATE_SUB(NOW(), INTERVAL 1 DAY)
  GROUP BY photo_id
) AS t
LEFT JOIN photos p ON p.id = t.photo_id
WHERE p.private = 0 AND p.category_id IS NOT NULL
GROUP BY t.photo_id
ORDER BY score DESC);

Lấy danh mục "Home" và "Eatout"
Thực hiện truy vấn để lấy tất cả các danh mục có CATEGORY_TYPE_NUMBER là HOME ([homeCategories]) và EATOUT ([eatoutCategories]).

Xử lý danh mục "Home"
Thực hiện xử lý các danh mục [homeCategories] với điều kiện venue_id = 1 để xác định các ảnh hot cho các địa điểm thuộc danh mục "Home". Kết quả được nhập với tên venue_id=1.

Xử lý danh mục "Eatout"
Thực hiện xử lý các danh mục [eatoutCategories] với điều kiện venue_id > 1 để xác định các ảnh hot cho các địa điểm thuộc danh mục "Eatout". Kết quả được nhập với tên venue_id=eatout.

Xử lý tất cả danh mục
Nối hai mảng [homeCategories] và [eatoutCategories] lại thành [allCategories]. Sau đó, thực hiện xử lý [allCategories] mà không có điều kiện venue_id cụ thể để xác định các ảnh hot cho tất cả các địa điểm. Kết quả được nhập với tên venue_id=all.

Giải thích chi tiết hàm processCategoriesWithVenue:

Hàm này nhận vào một mảng các danh mục ([categories]), một tên cơ sở ([baseName]), một điều kiện địa điểm ([venueCondition]) và đối tượng transaction.

Thêm giá trị null vào đầu mảng [categories].

Lặp qua từng category trong mảng [categories]:

Khởi tạo name với giá trị của [baseName] và query với 'SELECT MIN(id) AS min_id FROM tt_venue_scores '.

Khởi tạo đối tượng replacements.

Nếu có [venueCondition], thêm điều kiện này vào query.

Nếu category không phải là null:

Nối &category_id=${category.id} vào name.

Tạo một mảng categoryIds ban đầu chứa category.id.

Nếu category không có parent_category_id, tìm tất cả các danh mục con của nó (với idx > -100) và thêm ID của chúng vào categoryIds.

Thêm điều kiện category_id IN (:categoryIds) vào query (sử dụng AND hoặc WHERE tùy thuộc vào sự tồn tại của venueCondition).

Gán categoryIds vào replacements.categoryIds.

Thêm GROUP BY user_id ORDER BY id LIMIT 500 vào query.

Thực hiện truy vấn để lấy min_id từ tt_venue_scores và gán kết quả vào [minIdsRows].

Nếu [minIdsRows] rỗng, bỏ qua category này và tiếp tục vòng lặp.

Thực hiện truy vấn để lấy photo_id từ tt_venue_scores với các id trong [minIdsRows] và gán kết quả vào [rows].

Trích xuất tất cả các photo_id từ [rows] vào mảng [photoIds].

Gọi venueMergeRepo.importHotPhotos để nhập các ảnh hot với name và [photoIds] vào hệ thống.

Sau khi hoàn tất toàn bộ xử lý thì xuất log.

LOG:: [batch_name] + " finishied successfully: " + [started_time]

Control DB
tt_venue_scores/Insert
Tên vật lý	NotNull	Value
id	TRUE	AUTO_INCREMENT
photo_id		ID của ảnh
score		Tổng điểm của ảnh (từ favorites và comments trong 1 ngày gần nhất)
user_id		ID của người dùng sở hữu ảnh
venue_id		ID của địa điểm liên quan đến ảnh
category_id		ID của danh mục của ảnh
recipe_id		ID của công thức liên quan đến ảnh

[<tên biến/cấu trúc>] (nếu có)
CATEGORY_TYPE_NUMBER
Tên vật lý	NotNull	Value
HOME	TRUE	Số đại diện cho loại danh mục "Home"
EATOUT	TRUE	Số đại diện cho loại danh mục "Eatout"

COMMENT_WEIGHT
Tên vật lý	NotNull	Value
COMMENT_WEIGHT	TRUE	Hệ số trọng số dùng để tính điểm cho bình luận (ví dụ: 10)
