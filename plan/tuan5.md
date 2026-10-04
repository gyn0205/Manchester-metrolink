# Tuần 5 (02/11–08/11): API `server.js`

| | |
|---|---|
| Phụ trách chính | **Hà Nam** |
| Hỗ trợ và review | Kiệt (câu SQL, kiểm tra đồng bộ hai CSDL, bộ lệnh thử), Hải Nam (review `/api/routing`, đo thời gian chạy) |
| Sản phẩm cuối tuần | `backend/server.js` với 8 endpoint, đã thử bằng dòng lệnh; script kiểm tra đồng bộ |
| Cần có trước | `metro.exe` (tuần 4), database `metrolink` trong MySQL (tuần 2), `backend/README.md` (nhiệm vụ 4.8) |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

## Hợp đồng của API

Frontend ở tuần 6 và 7 chỉ dựa vào bảng này. Chốt trước khi viết; đổi thì báo cả nhóm.

| Phương thức | Đường dẫn | Đầu vào | Trả về khi thành công | Cần token admin |
|---|---|---|---|---|
| GET | `/api/stations` | Không | Mảng `{node_id, stop_name, stop_lat, stop_lon, status}` | Không |
| GET | `/api/routing` | Query `start`, `end`, `mode`, `time` | Nguyên JSON của `metro.exe` | Không |
| POST | `/api/register` | Body `{username, password}` | `{success, message}` | Không |
| POST | `/api/login` | Body `{username, password}` | `{success, token, role, username}` | Không |
| POST | `/api/admin/station/status` | Body `{stationId, status}` | `{message}` | Có |
| GET | `/api/admin/peak-hours` | Không | Mảng khung giờ | Có |
| POST | `/api/admin/peak-hours` | Body `{gio_bat_dau, gio_ket_thuc, he_so_luu_luong, thoi_gian_cho_tau}` | `{success, message}` | Có |
| DELETE | `/api/admin/peak-hours/:id` | Tham số `id` | `{success, message}` | Có |

Mã trạng thái dùng chung: 400 khi đầu vào sai, 401 khi thiếu hoặc sai token, 403 khi không phải admin, 404 khi không tìm thấy bản ghi, 500 khi lỗi phía máy chủ. Mọi phản hồi lỗi có trường `message`.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 5.1 | Khởi tạo backend: thư viện, `.env`, kết nối MySQL và SQLite | Hà Nam | Kiệt | T2 02/11 |
| 5.2 | `GET /api/stations` | Hà Nam | Kiệt | T2 02/11 |
| 5.3 | `GET /api/routing` gọi `metro.exe` | Hà Nam | Hải Nam | T4 04/11 |
| 5.4 | `POST /api/admin/station/status` ghi hai CSDL | Hà Nam | Kiệt | T5 05/11 |
| 5.5 | Ba endpoint `/api/admin/peak-hours` | Hà Nam | Kiệt | T6 06/11 |
| 5.6 | `POST /api/register` và `POST /api/login` | Hà Nam | Hải Nam | T7 07/11 |
| 5.7 | Middleware kiểm tra token cho `/api/admin` | Hà Nam | Hải Nam | T7 07/11 |
| 5.8 | Viết và thử trước các câu SQL trong Workbench | Kiệt | Hà Nam | T3 03/11 |
| 5.9 | Bộ lệnh thử API và script kiểm tra đồng bộ hai CSDL | Kiệt | Hà Nam | CN 08/11 |
| 5.10 | Đo thời gian chạy `metro.exe`; phân tích chế độ 2 | Hải Nam | Kiệt | CN 08/11 |
| 5.11 | Buổi giảng lại tuần 5 | Hà Nam trình bày | Cả nhóm | CN 08/11 |

## Hướng dẫn chi tiết

Viết từng endpoint, chạy `node server.js` trong `backend/`, thử ngay rồi mới viết endpoint kế.

### 5.1 Khởi tạo backend (Hà Nam)

1. Trong `backend/`: `npm init -y`, rồi cài `express`, `cors`, `mysql2`, `sqlite3`, `bcrypt`, `jsonwebtoken`, `dotenv`.
2. Tạo `backend/.env` với các biến `DB_HOST`, `DB_USER`, `DB_PASSWORD`, `DB_NAME`, `JWT_SECRET`, `PORT`. Kiểm tra `.env` đã nằm trong `.gitignore`.
3. Tạo `backend/.env.example` có cùng tên biến nhưng giá trị để trống, và commit file này.
4. Trong `server.js`: nạp `dotenv`, tạo app Express, bật `cors()` và `express.json()`.
5. Kết nối MySQL bằng các biến trong `.env`. Gọi thử một truy vấn ngay lúc khởi động và in lỗi nếu kết nối hỏng.
6. Mở SQLite bằng đường dẫn tuyệt đối ghép từ `__dirname` và `metrolink.db`. Đếm số dòng `KhungGioCaoDiem` và in ra để biết file mở đúng.
7. Lắng nghe ở cổng `PORT`.

**Vì sao không viết mật khẩu vào code:** bản Madrid để mật khẩu MySQL và khóa JWT ngay trong `server.js`, nên ai có repo cũng có mật khẩu.

**Kiểm tra:** `node server.js` in ra cổng đang chạy và "2 khung giờ". Sửa sai mật khẩu trong `.env` rồi chạy lại: phải thấy thông báo lỗi kết nối rõ ràng.

### 5.2 `GET /api/stations` (Hà Nam)

1. Truy vấn MySQL lấy `node_id`, `stop_name`, `stop_lat`, `stop_lon`, `status` từ bảng `tram`, sắp theo `node_id`.
2. Lỗi thì trả 500 kèm `message`; thành công thì trả mảng JSON.

**Lưu ý:** viết tên bảng bằng chữ thường, đúng như trong `database.sql`. Bản Madrid viết `Tram`; cách đó chỉ chạy được trên Windows, sang Linux là lỗi.

**Kiểm tra:**

```powershell
(Invoke-RestMethod http://localhost:3000/api/stations).Count
```

**Kết quả đúng:** 192.

### 5.3 `GET /api/routing` (Hà Nam)

1. Đọc `start`, `end`, `mode`, `time` từ `req.query`.
2. Kiểm tra từng tham số trước khi dùng. Sai thì trả 400:

   | Tham số | Hợp lệ khi |
   |---|---|
   | `start`, `end` | Chỉ gồm chữ số |
   | `mode` | Là `1` hoặc `2` |
   | `time` | Đúng dạng `HH:MM` |

3. Gọi `metro.exe` bằng `execFile` của module `child_process`: đường dẫn tuyệt đối tới file exe, bốn tham số truyền dưới dạng mảng, tùy chọn `cwd` là `__dirname`.
4. Trong hàm gọi lại: có lỗi thì trả 500; không lỗi thì `JSON.parse` stdout trong `try/catch` và trả kết quả.

**Vì sao dùng `execFile` thay vì `exec`:** bản Madrid ghép bốn tham số vào một chuỗi lệnh rồi đưa cho `exec`, tức là cho shell chạy. Một ký tự `&` trong tham số `time` đủ để nối thêm một lệnh bất kỳ. `execFile` không đi qua shell, và bước 2 chặn tham số lạ.

**Vì sao cần `cwd`:** `metro.exe` mở `metrolink.db` theo đường dẫn tương đối (xem câu hỏi 2 của tuần 4).

**Kiểm tra:**

```powershell
curl.exe "http://localhost:3000/api/routing?start=3&end=27&mode=1&time=12:00"
curl.exe -i "http://localhost:3000/api/routing?start=abc&end=27&mode=1&time=12:00"
```

**Kết quả đúng:** lệnh đầu trả đúng JSON mà `.\metro.exe 3 27 1 12:00` in ra, `total_time` bằng 3720. Lệnh thứ hai trả mã 400.

### 5.4 `POST /api/admin/station/status` (Hà Nam)

1. Đọc `stationId` và `status` từ body. `status` chỉ được là 0 hoặc 1; sai thì trả 400.
2. Cập nhật `status` của nút trong MySQL. Không dòng nào bị ảnh hưởng thì trả 404.
3. Cập nhật tiếp cùng nút đó trong SQLite.
4. Cả hai thành công mới trả thành công.

**Điểm cần hiểu:** danh sách ga trên web đọc từ MySQL, còn `metro.exe` đọc SQLite. Chỉ cập nhật MySQL thì trang quản trị báo "đã đóng" nhưng tàu vẫn chạy qua ga đó. Hãy quyết định và ghi vào code: nếu MySQL thành công mà SQLite thất bại thì làm gì?

**Kiểm tra:** gọi endpoint đóng nút 178, rồi chạy script của nhiệm vụ 5.9 để thấy hai CSDL khớp. Mở lại nút 178 sau khi thử.

### 5.5 Ba endpoint `/api/admin/peak-hours` (Hà Nam)

1. `GET`: trả mọi dòng của bảng `khunggiocaodiem` trong MySQL.
2. `POST`: kiểm tra đầu vào rồi chèn vào MySQL, sau đó chèn vào SQLite.

   | Trường | Hợp lệ khi |
   |---|---|
   | `gio_bat_dau`, `gio_ket_thuc` | Đúng dạng `HH:MM`, giờ bắt đầu nhỏ hơn giờ kết thúc |
   | `he_so_luu_luong` | Số, lớn hơn hoặc bằng 1 |
   | `thoi_gian_cho_tau` | Số, lớn hơn hoặc bằng 0 |

3. `DELETE`: xóa ở MySQL rồi xóa ở SQLite. Không tìm thấy thì trả 404.

**Vì sao hệ số phải từ 1 trở lên:** lập luận "A* tối ưu" ở tuần 3 dựa vào việc giờ cao điểm chỉ làm chi phí tăng. Nếu quản trị viên nhập hệ số nhỏ hơn 1, tàu "chạy nhanh hơn" vận tốc mà heuristic giả định, và A* có thể trả kết quả không tối ưu.

**Vấn đề `id` lệch:** MySQL và SQLite mỗi bên tự tăng `id` riêng. Chọn một trong hai cách và ghi lý do vào code:

| Cách | Làm thế nào | Nhược điểm |
|---|---|---|
| Xóa theo khung giờ (bản Madrid) | Đọc `gio_bat_dau`, `gio_ket_thuc` từ MySQL theo `id`, rồi xóa ở SQLite theo hai giá trị đó | Hai khung giờ trùng giờ sẽ bị xóa cùng lúc ở SQLite |
| Giữ `id` giống nhau | Khi chèn, lấy `insertId` của MySQL và chèn vào SQLite với đúng `id` đó; xóa theo `id` ở cả hai bên | Phải bảo đảm hai bên chưa lệch từ trước |

**Kiểm tra:** thêm khung 12:00–13:00, hệ số 2, chờ 600. Gọi `/api/routing?start=3&end=27&mode=1&time=12:00`: phải ra 5.670 giây (94,5 phút) thay vì 3.720. Xóa khung giờ vừa thêm: kết quả trở lại 3.720.

### 5.6 Đăng ký và đăng nhập (Hà Nam)

**`POST /api/register`:**

1. `username` hoặc `password` rỗng thì trả 400.
2. Băm mật khẩu bằng `bcrypt.hash` với 10 vòng.
3. Chèn vào bảng `users` với `role` là `user`.
4. Lỗi `ER_DUP_ENTRY` thì trả 400 với thông báo tên đăng nhập đã tồn tại.

**`POST /api/login`:**

1. Tìm người dùng theo `username`.
2. So mật khẩu bằng `bcrypt.compare`. Không khớp thì trả 401.
3. Tạo token bằng `jwt.sign`, payload `{ id, role }`, khóa lấy từ `.env`, hạn 1 ngày.
4. Trả `{ success, token, role, username }`.

**Tạo tài khoản admin:** đăng ký tài khoản `admin` qua API, rồi chạy trong Workbench:

```sql
UPDATE metrolink.users SET role = 'admin' WHERE username = 'admin';
```

**Kiểm tra:**

```powershell
$body = @{ username = 'admin'; password = '<mật khẩu>' } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri http://localhost:3000/api/register -ContentType 'application/json' -Body $body
$login = Invoke-RestMethod -Method Post -Uri http://localhost:3000/api/login -ContentType 'application/json' -Body $body
$login.role
```

**Kết quả đúng:** đăng ký lần hai với cùng tên bị từ chối; sau lệnh `UPDATE` và đăng nhập lại, `$login.role` là `admin`; cột `password` trong MySQL là chuỗi băm bắt đầu bằng `$2b$`.

### 5.7 Middleware kiểm tra token (Hà Nam)

Bản Madrid cấp token nhưng không endpoint nào kiểm tra, nên ai cũng gọi được API quản trị. Bản của nhóm phải kiểm tra.

1. Viết hàm middleware: đọc header `Authorization`, tách phần sau chữ `Bearer`.
2. Thiếu header hoặc `jwt.verify` ném lỗi thì trả 401.
3. Token hợp lệ nhưng `role` khác `admin` thì trả 403.
4. Hợp lệ thì gọi `next()`.
5. Gắn middleware cho mọi đường dẫn bắt đầu bằng `/api/admin`, đặt trước phần khai báo các route đó.

**Kiểm tra:**

```powershell
$h = @{ Authorization = "Bearer $($login.token)" }
Invoke-RestMethod http://localhost:3000/api/admin/peak-hours -Headers $h
curl.exe -i http://localhost:3000/api/admin/peak-hours
```

**Kết quả đúng:** lệnh đầu trả 2 khung giờ; lệnh thứ hai trả 401. Đăng nhập bằng một tài khoản thường rồi gọi lại với token đó: trả 403.

### 5.8 Viết trước các câu SQL (Kiệt)

Trước ngày Hà Nam viết từng endpoint, Kiệt viết và chạy thử câu SQL tương ứng trong Workbench, lưu vào `backend/thu_nghiem/cau_sql_api.sql`:

| Endpoint | Câu SQL cần có |
|---|---|
| `/api/stations` | Lấy 5 cột của `tram`, sắp theo `node_id` |
| `/api/admin/station/status` | Cập nhật `status` theo `node_id`, cho cả MySQL lẫn SQLite |
| `/api/admin/peak-hours` | Lấy tất cả; chèn một khung giờ; xóa theo `id` |
| `/api/register`, `/api/login` | Chèn người dùng; tìm theo `username` |

Mọi câu có tham số đều viết với dấu `?` giữ chỗ, không ghép chuỗi.

### 5.9 Bộ lệnh thử và script kiểm tra đồng bộ (Kiệt)

1. Viết `backend/thu_api.ps1`: một loạt lệnh `Invoke-RestMethod` và `curl.exe -i` gọi lần lượt cả 8 endpoint, gồm cả các trường hợp sai (tham số sai, thiếu token, token của người dùng thường). In ra tên phép thử và kết quả đạt hay không.
2. Viết `backend/check_sync.js` bằng `mysql2` và `sqlite3`:
   - So `status` của cả 192 nút giữa hai CSDL.
   - So danh sách khung giờ (bốn trường) giữa hai CSDL.
   - In "ĐỒNG BỘ", hoặc liệt kê từng chỗ lệch.
3. Chạy `check_sync.js` sau mỗi phép thử ghi dữ liệu.

**Kết quả đúng:** sau khi chạy hết `thu_api.ps1` và hoàn tác các thay đổi, `check_sync.js` in "ĐỒNG BỘ", MySQL và SQLite đều còn đúng 2 khung giờ và 192 nút đang mở.

### 5.10 Đo thời gian chạy và phân tích chế độ 2 (Hải Nam)

1. Đo thời gian một lần gọi `metro.exe` bằng `Measure-Command`, 20 lần cho mỗi truy vấn trong 4 truy vấn khác nhau, lấy trung bình. Bản nháp báo cáo ghi 17–20 ms; thay bằng số đo thật.
2. Thêm chế độ 2 vào `search` của `evaluate.py`, đúng công thức chi phí của `main.cpp`.
3. Trên toàn bộ 9.702 cặp, so chế độ 2 có heuristic với chế độ 2 không heuristic: bao nhiêu cặp có cùng số lần đổi tuyến, bao nhiêu cặp có số chặng nhiều hơn mức tối thiểu, nhiều hơn tối đa mấy chặng. Bản nháp báo cáo ghi 605 cặp, nhiều nhất 4 chặng; kiểm tra lại.
4. Ghi kết quả vào một file tạm để tuần 8 đưa vào báo cáo.

## Nghiệm thu cuối tuần

- [ ] `node server.js` khởi động không lỗi; mật khẩu và khóa JWT nằm trong `.env`, không nằm trong code.
- [ ] `/api/stations` trả 192 phần tử.
- [ ] `/api/routing?start=3&end=27&mode=1&time=12:00` trả `total_time` bằng 3720; tham số sai trả 400.
- [ ] Đóng rồi mở một nút qua API: `check_sync.js` in "ĐỒNG BỘ" ở cả hai lần.
- [ ] Thêm khung 12:00–13:00 hệ số 2 chờ 600: Altrincham đi Bury lúc 12:00 ra 94,5 phút; xóa đi thì trở lại 62 phút.
- [ ] Đăng ký, đăng nhập chạy; mật khẩu trong MySQL là chuỗi băm.
- [ ] `/api/admin/*` trả 401 khi thiếu token, 403 với token người dùng thường, 200 với token admin.
- [ ] `thu_api.ps1` chạy hết, mọi phép thử đạt.

## Câu hỏi cả nhóm phải trả lời được

1. Vì sao danh sách ga đọc từ MySQL còn tìm đường lại đọc SQLite? Chỉ cập nhật một bên thì chuyện gì xảy ra?
2. `exec` và `execFile` khác nhau thế nào? Vì sao vẫn phải kiểm tra tham số khi đã dùng `execFile`?
3. Cột `password` chứa gì? Vì sao băm hai lần cùng một mật khẩu lại ra hai chuỗi khác nhau mà `compare` vẫn đúng?
4. Ai cũng đọc được payload của JWT. Vậy điều gì ngăn người dùng tự sửa `role` thành `admin`?
5. Vì sao hệ số lưu lượng phải lớn hơn hoặc bằng 1?
6. Thêm một khung giờ qua API thì lần tìm đường kế tiếp đã đổi kết quả, không cần khởi động lại gì. Vì sao?

<details>
<summary>Gợi ý đáp án</summary>

1. MySQL phục vụ web (nhiều kết nối, có tài khoản người dùng); SQLite là file để `metro.exe` đọc trực tiếp. Chỉ cập nhật MySQL thì giao diện đổi nhưng thuật toán không đổi; chỉ cập nhật SQLite thì ngược lại.
2. `exec` đưa cả chuỗi cho shell diễn giải; `execFile` chạy thẳng file với danh sách tham số. Vẫn phải kiểm tra vì `metro.exe` có thể xử lý sai tham số lạ, và để trả lỗi 400 rõ ràng thay vì 500.
3. Chuỗi băm của bcrypt, trong đó có sẵn muối (salt) ngẫu nhiên. `compare` đọc muối từ chuỗi băm, băm lại mật khẩu nhập vào với muối đó rồi so.
4. Chữ ký. Sửa payload thì chữ ký không còn khớp, và `jwt.verify` với khóa bí mật trên máy chủ sẽ từ chối.
5. Để heuristic vẫn chấp nhận được: `h` giả định tàu không nhanh hơn 20 m/s. Hệ số nhỏ hơn 1 rút ngắn thời gian chạy, tức là tăng vận tốc thật.
6. `metro.exe` khởi động lại và đọc lại bảng `KhungGioCaoDiem` ở mỗi lần gọi.

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| `EADDRINUSE: address already in use :::3000` | Cổng 3000 đang bị chiếm (trên máy Hải Nam là Docker Desktop). Dừng chương trình đó, hoặc đổi `PORT` trong `.env` |
| `ER_ACCESS_DENIED_ERROR` | Sai `DB_PASSWORD` trong `.env`, hoặc chưa nạp `dotenv` trước khi tạo kết nối |
| `ER_BAD_DB_ERROR: Unknown database 'metrolink'` | Chưa import `database.sql` |
| `/api/routing` trả "Not found" với mọi cặp | Thiếu `cwd` khi gọi `execFile`, hoặc `metrolink.db` không nằm trong `backend` |
| `/api/routing` trả 500 "Lỗi thực thi" | Chưa biên dịch `metro.exe`, hoặc đường dẫn tới file exe sai |
| `req.body` là `undefined` | Chưa bật `express.json()`, hoặc lệnh thử thiếu header `Content-Type: application/json` |
| `curl` trong PowerShell báo lỗi tham số | `curl` là bí danh của `Invoke-WebRequest`. Gõ `curl.exe` |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 7.3 và 7.4.
- Bản Madrid: `Project-AI-2025.2-main/backend/server.js`. Bản này dùng `exec` với chuỗi ghép, để mật khẩu trong code và không kiểm tra token; đừng chép ba chỗ đó.
