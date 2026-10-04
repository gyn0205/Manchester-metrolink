# Tuần 2 (12/10–18/10): Dựng đồ thị, SQLite và MySQL

| | |
|---|---|
| Phụ trách chính | **Kiệt** |
| Hỗ trợ và review | Hải Nam (review đồ thị), Hà Nam (MySQL) |
| Sản phẩm cuối tuần | `data_pipeline/database_builder.ipynb`, `backend/metrolink.db`, `database.sql` đã import vào MySQL |
| Cần có trước | `cleaned_stops.csv` và `cleaned_stop_times.csv` của tuần 1 |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

## Tuần này tạo ra gì cho các tầng sau

- **`node_id`:** số nguyên 0 đến 191, từ tuần này trở đi mọi tầng dùng nó để gọi tên một nút.
- **`metrolink.db` (SQLite):** lõi tìm đường đọc file này mỗi lần chạy.
- **`database.sql` (MySQL):** API đọc danh sách ga, khung giờ cao điểm và tài khoản từ đây.

Hai cơ sở dữ liệu chứa cùng một đồ thị. Nếu chạy lại notebook mà chỉ cập nhật một bên, `node_id` hai bên lệch nhau và đường đi sẽ vô lý.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 2.1 | Ô 1a: mã hóa nút thành `node_id` | Kiệt | Hải Nam | T2 12/10 |
| 2.2 | Ô 1b: tìm ga kế tiếp, tính thời gian chạy, gộp cạnh bằng trung vị | Kiệt | Hải Nam | T4 14/10 |
| 2.3 | Ô 2: kiểm tra đồ thị | Kiệt | Hải Nam | T4 14/10 |
| 2.4 | Ô 3: ghi SQLite | Kiệt | Hải Nam | T5 15/10 |
| 2.5 | Viết tay lược đồ MySQL trong Workbench | Kiệt và Hà Nam | | T6 16/10 |
| 2.6 | Ô 4: sinh `database.sql`, import vào MySQL | Kiệt | Hà Nam | T7 17/10 |
| 2.7 | Đặt file đúng chỗ, commit | Kiệt | | T7 17/10 |
| 2.8 | Khảo sát đồ thị bằng SQL | Hải Nam | | CN 18/10 |
| 2.9 | Chạy lại notebook và import MySQL trên máy mình | Hải Nam, Hà Nam | | CN 18/10 |
| 2.10 | Chuẩn bị cho tuần 3: Haversine và A* trên đồ thị mẫu | Hải Nam | | CN 18/10 |
| 2.11 | Chuẩn bị cho tuần 5: Express đọc MySQL | Hà Nam | | CN 18/10 |
| 2.12 | Buổi giảng lại tuần 2 | Kiệt trình bày | Cả nhóm | CN 18/10 |

## Hướng dẫn chi tiết

### 2.1 Ô 1a: mã hóa nút (Kiệt)

Tạo `data_pipeline/database_builder.ipynb`.

1. Đọc hai file CSV của tuần 1; `trip_id` ép kiểu chuỗi.
2. Thêm cột `node_id` bằng số thứ tự dòng (0, 1, 2…) và cột `status` bằng 1 cho bảng nút.
3. Tạo một `dict` ánh xạ từ khóa chữ (`<mã ga>|<tuyến>`) sang `node_id`.

**Kiểm tra:** `dict` có 192 phần tử; `node_id` lớn nhất là 191.

### 2.2 Ô 1b: tính cạnh (Kiệt)

Một cạnh có hướng nối hai điểm dừng liên tiếp trong cùng một chuyến.

1. Sắp `stop_times` theo `trip_id` rồi `stop_sequence`.
2. Trong từng chuyến, lấy `stop_id` và `arrival_time` của dòng kế tiếp, đặt vào hai cột mới `next_stop_id` và `next_arrival`.
3. Bỏ dòng cuối của mỗi chuyến (không có ga kế tiếp).
4. Tính `travel_time` bằng giây: giờ đến ga sau trừ giờ rời ga trước.
5. Gộp theo cặp (`stop_id`, `next_stop_id`): `travel_time` lấy trung vị, thêm cột `so_chuyen` đếm số chuyến đi qua cạnh.
6. Đổi hai đầu cạnh sang `node_id`, đặt tên cột là `u` và `v`. Thêm cột `status` bằng 1.

**Gợi ý:** `groupby('trip_id')[...].shift(-1)`; `pd.to_timedelta` đọc được chuỗi giờ lớn hơn 24 như `25:10:00`; `.dt.total_seconds()`; `groupby([...]).agg(...)` với `'median'` và `'size'`; `Series.map(dict)`.

**Tự kiểm tra thêm:** trong feed, `arrival_time` và `departure_time` của cùng một dòng có khác nhau không? Kết quả đúng: hai cột bằng nhau ở mọi dòng tram. Feed không ghi thời gian tàu dừng ở ga, nên thời gian hành trình tính ra thấp hơn thực tế một chút. Ghi điều này vào mục "Hạn chế" của báo cáo.

### 2.3 Ô 2: kiểm tra đồ thị (Kiệt)

1. In số dòng cạnh thô, số dòng có thời gian âm, số dòng bằng 0.
2. In số nút, số ga, số cạnh có hướng.
3. In thời gian chạy nhỏ nhất, lớn nhất, trung bình của các cạnh sau khi gộp.
4. Thêm hai `assert`: mọi cạnh có thời gian lớn hơn 0; không cạnh nào trỏ tới nút không tồn tại.
5. Kiểm tra thêm: mỗi nút có ít nhất một cạnh đi ra hoặc đi vào.
6. Đếm số ga có từ hai nút trở lên (ga trung chuyển) và tính tổng `k × (k − 1)` trên các ga, với `k` là số nút của ga. Con số thứ hai chính là số cạnh đổi tuyến mà lõi tìm đường sẽ tự sinh ở tuần 3 và 4.

**Kết quả đúng:**

```text
Số dòng cạnh thô: 258545 | thời gian âm: 0 | bằng 0: 3
Số nút: 192 | Số ga: 99 | Số cạnh có hướng: 378
Thời gian chạy (giây): min 60.0 | max 360.0 | trung bình 158.1
Ga trung chuyển: 44 | Cạnh đổi tuyến: 390
```

### 2.4 Ô 3: ghi SQLite (Kiệt)

1. Mở kết nối tới `metrolink.db` bằng module `sqlite3`.
2. Ghi bảng `Tram` với các cột `node_id`, `stop_id`, `stop_name`, `stop_lat`, `stop_lon`, `line`, `status`.
3. Ghi bảng `Ket_Noi` với các cột `u`, `v`, `travel_time`, `so_chuyen`, `status`.
4. Tạo hai chỉ mục trên `Ket_Noi(u)` và `Ket_Noi(v)`.
5. Tạo bảng `KhungGioCaoDiem` gồm `id` (khóa chính tự tăng), `gio_bat_dau`, `gio_ket_thuc` (chuỗi `HH:MM`), `he_so_luu_luong`, `thoi_gian_cho_tau` (số thực).
6. Khai báo `PEAK_HOURS` ở đầu ô và chèn hai khung giờ mặc định:

   | Bắt đầu | Kết thúc | Hệ số lưu lượng | Giây chờ thêm khi đổi tuyến |
   |---|---|---|---|
   | 07:00 | 09:30 | 1,5 | 300 |
   | 16:00 | 18:30 | 1,3 | 180 |

7. `commit` rồi đóng kết nối.

**Gợi ý:** `DataFrame.to_sql(tên bảng, conn, if_exists='replace', index=False)`; `cursor.executemany(...)` với dấu `?` cho tham số.

**Lưu ý:** hai khung giờ này là giả định của nhóm, vì TfGM không công bố dữ liệu tương ứng. Báo cáo phải ghi rõ điều đó.

**Kiểm tra:** mở lại `metrolink.db` bằng Python, đếm ba bảng. Phải ra 192, 378 và 2.

### 2.5 Viết tay lược đồ MySQL (Kiệt và Hà Nam)

Trước khi để notebook sinh file SQL, hai bạn tự gõ lệnh `CREATE TABLE` trong Workbench để hiểu từng cột.

1. Tạo database `metrolink` với bảng mã `utf8mb4`.
2. Tạo bốn bảng:

   | Bảng | Cột |
   |---|---|
   | `tram` | `node_id` int, khóa chính; `stop_name` varchar(255); `stop_lat`, `stop_lon` double; `status` tinyint(1) mặc định 1 |
   | `ket_noi` | `u`, `v` int; `travel_time` double; `status` tinyint(1) mặc định 1 |
   | `khunggiocaodiem` | `id` int tự tăng, khóa chính; `gio_bat_dau`, `gio_ket_thuc` varchar(10); `he_so_luu_luong` float mặc định 1; `thoi_gian_cho_tau` float mặc định 0 |
   | `users` | `id` int tự tăng, khóa chính; `username` varchar(255), duy nhất; `password` varchar(255); `role` enum('user','admin') mặc định 'user'; `created_at` timestamp |

3. Chèn thử một dòng vào mỗi bảng, đọc lại, rồi xóa database để tuần này import bản chính thức.

**Điểm cần hiểu:** cột `password` dài 255 ký tự vì nó chứa chuỗi băm của bcrypt, không phải mật khẩu gốc.

### 2.6 Ô 4: sinh `database.sql` và import (Kiệt)

1. Viết hàm bọc chuỗi trong dấu nháy đơn cho SQL. Hàm phải xử lý được tên ga có dấu nháy, ví dụ St Peter's Square.
2. Ghép danh sách các lệnh theo thứ tự: tạo database, `USE`, rồi với từng bảng `tram`, `ket_noi`, `khunggiocaodiem` thì `DROP TABLE IF EXISTS` trước, `CREATE TABLE` sau.
3. Riêng bảng `users` dùng `CREATE TABLE IF NOT EXISTS` và không `DROP`.
4. Thêm ba lệnh `INSERT` nhiều dòng cho `tram`, `ket_noi` và `khunggiocaodiem`. Khung giờ lấy từ cùng biến `PEAK_HOURS` của ô 3.
5. Ghi ra `database.sql` với mã hóa UTF-8.
6. Import bằng Workbench (File → Open SQL Script → Execute) hoặc bằng dòng lệnh:

   ```powershell
   cmd /c '"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u root -p --default-character-set=utf8mb4 < database.sql'
   ```

**Kết quả đúng:** truy vấn sau trả về 192, 378, 2.

```sql
SELECT (SELECT COUNT(*) FROM metrolink.tram) AS so_nut,
       (SELECT COUNT(*) FROM metrolink.ket_noi) AS so_canh,
       (SELECT COUNT(*) FROM metrolink.khunggiocaodiem) AS so_khung_gio;
```

### 2.7 Đặt file đúng chỗ (Kiệt)

1. Chép `data_pipeline/metrolink.db` vào `backend/`.
2. Chuyển `data_pipeline/database.sql` ra thư mục gốc của project.
3. Commit notebook, `backend/metrolink.db` và `database.sql`.

**Quy tắc của nhóm:** mỗi lần chạy lại notebook phải làm lại cả ba việc: chép `metrolink.db`, chuyển `database.sql`, import lại MySQL.

### 2.8 Khảo sát đồ thị bằng SQL (Hải Nam)

Viết truy vấn trên `metrolink.db` (bằng Python `sqlite3` hoặc một công cụ xem SQLite) để trả lời:

| Câu hỏi | Kết quả đúng |
|---|---|
| St Peter's Square có bao nhiêu nút? | 7, mỗi tuyến một nút |
| `node_id` của các nút Altrincham và Bury là gì, thuộc tuyến nào? | Altrincham: 3 (Blue), 4 (Green), 5 (Purple). Bury: 27 (Blue), 28 (Green) |
| Manchester Airport và Rochdale Town Centre có mấy nút? | Mỗi ga một nút: 89 (Navy) và 136 (Pink) |
| Từ một nút Altrincham đi ra được những nút nào, mất bao nhiêu giây? | Ghi lại |
| Có bao nhiêu ga có từ hai nút trở lên? | 44 |
| Tổng `k × (k − 1)` trên các ga là bao nhiêu? | 390 |
| Cạnh nào dài nhất và ngắn nhất? | 360 giây và 60 giây |
| Mỗi cạnh (u, v) có cạnh ngược (v, u) không? | Có, ở cả 378 cạnh |

Ghi các truy vấn vào `backend/thu_nghiem/khao_sat_do_thi.sql` để tuần 3 dùng lại.

### 2.9 Chạy lại trên máy mình (Hải Nam, Hà Nam)

1. Kéo nhánh của Kiệt, chạy `database_builder.ipynb`, so bốn dòng số liệu ở 2.3.
2. Import `database.sql` vào MySQL của mình, chạy truy vấn đếm ở 2.6.
3. So `node_id` của Altrincham và Bury với máy Kiệt. Phải giống nhau.

### 2.10 Chuẩn bị: Haversine và A* trên đồ thị mẫu (Hải Nam)

1. Viết hàm `haversine(lat1, lon1, lat2, lon2)` trả về mét, bán kính Trái Đất 6.371.000 m.
2. Thử: khoảng cách giữa (53,0; −2,0) và (54,0; −2,0) phải xấp xỉ 111.195 m.
3. Mở rộng `toy_search.py` của tuần 1 thành A*: hàng đợi sắp theo `g + h`, với `h` đọc từ một `dict`.
4. Chạy với `h = {A: 10, B: 8, C: 9, D: 4, E: 2, F: 0}`. Kết quả phải vẫn là 13.
5. Đổi `h[C]` thành 20 rồi chạy lại. Kết quả phải là 14, tức là sai.

**Điều rút ra:** `h` không được lớn hơn chi phí thật còn lại. Ở bước 5, chi phí thật từ C tới F là 11, nhưng `h[C]` bằng 20.

### 2.11 Chuẩn bị: Express đọc MySQL (Hà Nam)

1. Trong `backend/thu_nghiem/`, chạy `npm install mysql2`.
2. Viết `doc_mysql.js`: kết nối tới database `metrolink`, thêm route `GET /api/dem` trả về số dòng của bảng `tram`.
3. Mật khẩu MySQL đọc từ biến môi trường, không viết thẳng vào file.

**Kết quả đúng:** `http://localhost:3000/api/dem` trả về 192.

## Nghiệm thu cuối tuần

- [ ] `database_builder.ipynb` chạy Run All không lỗi trên cả ba máy.
- [ ] 192 nút, 99 ga, 378 cạnh; thời gian chạy 60–360 giây, trung bình 158,1.
- [ ] `backend/metrolink.db` có ba bảng với 192, 378 và 2 dòng.
- [ ] MySQL có database `metrolink` với bốn bảng; truy vấn đếm ra 192, 378, 2.
- [ ] `node_id` của Altrincham và Bury giống nhau trên ba máy.
- [ ] Hải Nam: A* trên đồ thị mẫu ra 13 với `h` đúng và 14 với `h[C] = 20`.
- [ ] Hà Nam: `/api/dem` trả về 192.

## Câu hỏi cả nhóm phải trả lời được

1. Vì sao gộp mỗi cạnh bằng trung vị, không dùng trung bình hay giá trị nhỏ nhất?
2. Cạnh đổi tuyến có nằm trong bảng `Ket_Noi` không? Nếu không, chúng sinh ra ở đâu?
3. Vì sao cần cả SQLite lẫn MySQL? Mỗi bên phục vụ tầng nào?
4. Điều gì xảy ra nếu chạy lại notebook mà quên import lại MySQL?
5. Vì sao `users` dùng `IF NOT EXISTS` còn ba bảng kia thì `DROP` rồi tạo lại?
6. Bảng `tram` của MySQL thiếu những cột nào so với bảng `Tram` của SQLite? Trang quản trị mất gì vì thiếu cột đó?

<details>
<summary>Gợi ý đáp án</summary>

1. Cùng một cạnh có nhiều giá trị giữa các chuyến và có 3 dòng bằng 0. Trung vị không bị các giá trị bất thường kéo lệch; giá trị nhỏ nhất sẽ lấy đúng các dòng bằng 0.
2. Không. Lõi tìm đường tự sinh chúng lúc chạy: hai nút khác nhau có cùng tên ga thì nối với nhau, trọng số 300 giây.
3. SQLite là một file, lõi C++ đọc trực tiếp không cần máy chủ. MySQL phục vụ API: danh sách ga, khung giờ, tài khoản người dùng.
4. `node_id` hai bên có thể lệch. Frontend lấy `node_id` từ MySQL rồi gửi cho lõi C++ đọc SQLite, nên lõi sẽ tìm đường giữa hai nút khác với ý người dùng.
5. Ba bảng kia sinh lại được từ feed. `users` chứa tài khoản do người dùng tạo; `DROP` là mất hết.
6. Thiếu `stop_id` và `line`. Bảng quản trị hiện 192 dòng nhưng không cho biết mỗi dòng thuộc tuyến nào (xem nâng cấp tùy chọn ở tuần 7).

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| Thời gian chạy âm | Đã trừ 24 giờ ở tuần 1, hoặc chưa sắp theo `stop_sequence` trước khi `shift` |
| Số cạnh lớn hơn 378 rất nhiều | `shift` trên cả bảng thay vì trong từng chuyến, nên nối ga cuối chuyến này với ga đầu chuyến sau |
| `u` hoặc `v` bị thiếu | Khóa nút trong `stop_times` không khớp bảng nút; kiểm tra ô 3 của tuần 1 |
| MySQL báo lỗi cú pháp ở lệnh `INSERT` vào `tram` | Tên ga có dấu nháy đơn chưa được xử lý |
| `ER_ACCESS_DENIED_ERROR` | Sai mật khẩu `root` |
| Import xong nhưng tên ga bị lỗi ký tự | Thiếu `--default-character-set=utf8mb4` khi import bằng dòng lệnh |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 5 và Bước 6.
- Bản Madrid: `Project-AI-2025.2-main/data_pipeline/database_builder.ipynb` và `database.sql`. Bản Madrid giữ nguyên các dòng cạnh thô và không ghi khung giờ vào MySQL.
