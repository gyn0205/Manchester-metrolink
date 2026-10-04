# Tuần 4 (26/10–01/11): Lõi C++ `main.cpp`

| | |
|---|---|
| Phụ trách chính | **Hải Nam** |
| Hỗ trợ và review | Kiệt (script đối chiếu, thử trường hợp biên), Hà Nam (review hợp đồng JSON) |
| Sản phẩm cuối tuần | `backend/main.cpp` biên dịch thành `metro.exe`, cho kết quả khớp `evaluate.py`; tài liệu hợp đồng của `metro.exe` |
| Cần có trước | `backend/metrolink.db`, `backend/evaluate.py`, `backend/sqlite3.o` (nhiệm vụ 3.9) |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

## Hợp đồng của `metro.exe`

Tuần 5 `server.js` gọi chương trình này, tuần 6 frontend đọc JSON của nó. Chốt hợp đồng trước khi viết.

**Dòng lệnh:** `metro.exe <nút đi> <nút đến> <chế độ> <HH:MM>`

| Tham số | Ý nghĩa |
|---|---|
| Nút đi, nút đến | `node_id` của một nút bất kỳ thuộc ga đi và ga đến |
| Chế độ | 1 là nhanh nhất; 2 là ít đổi tuyến nhất |
| Giờ | Giờ khởi hành; thiếu thì coi là 00:00 |

**stdout khi tìm thấy:** đúng một đối tượng JSON.

```text
{"status": "success", "total_time": <giây>, "path": [
  {"name": <tên ga>, "lat": <số>, "lon": <số>,
   "step_time": <giây của chặng tới ga này>,
   "is_transfer": <true nếu chặng tới ga này là đổi tuyến>,
   "arrival_time": <giây tích lũy từ lúc khởi hành>}, ...]}
```

**stdout khi không tìm thấy:** `{"status": "error", "message": "Not found"}`

**stderr:** thông tin gỡ lỗi. Không in gì khác ngoài JSON ra stdout, vì `server.js` đưa toàn bộ stdout vào `JSON.parse`.

Một lần đổi tuyến hiện trong `path` thành hai phần tử liên tiếp cùng tên ga; phần tử thứ hai có `is_transfer` là `true`.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 4.1 | Khung chương trình: kiểu dữ liệu, tham số dòng lệnh, `parseTime` | Hải Nam | Hà Nam | T2 26/10 |
| 4.2 | Đọc ba bảng từ SQLite, dựng danh sách kề, thêm cạnh đổi tuyến | Hải Nam | Kiệt | T4 28/10 |
| 4.3 | Hàm `heuristic` (Haversine) | Hải Nam | Kiệt | T4 28/10 |
| 4.4 | A*: nhiều nguồn, chi phí theo giờ, hai chế độ | Hải Nam | Kiệt | T6 30/10 |
| 4.5 | Truy vết đường đi và in JSON | Hải Nam | Hà Nam | T7 31/10 |
| 4.6 | Script đối chiếu C++ với Python trên toàn bộ cặp ga | Kiệt | Hải Nam | CN 01/11 |
| 4.7 | Thử trường hợp biên | Kiệt | Hải Nam | CN 01/11 |
| 4.8 | Viết `backend/README.md`: hợp đồng của `metro.exe` | Hải Nam | Hà Nam | CN 01/11 |
| 4.9 | Chuẩn bị cho tuần 5: làm trước nhiệm vụ 5.1 | Hà Nam | | CN 01/11 |
| 4.10 | Buổi giảng lại tuần 4 | Hải Nam trình bày | Cả nhóm | CN 01/11 |

## Hướng dẫn chi tiết

Biên dịch sau mỗi nhiệm vụ. Lệnh biên dịch, chạy trong `backend/`:

```powershell
g++ main.cpp sqlite3.o -o metro.exe
```

### 4.1 Khung chương trình (Hải Nam)

1. Khai báo bốn kiểu dữ liệu:

   | Kiểu | Trường |
   |---|---|
   | `Node` | `id`, `name`, `lat`, `lon` |
   | `Edge` | `to`, `weight`, `is_transfer` |
   | `AStarNode` | `id`, `f_score`, `g_score`, và toán tử so sánh theo `f_score` |
   | `PeakHour` | `start_sec`, `end_sec`, `multiplier`, `extra_wait` |

2. Khai báo ba biến toàn cục: bảng nút (`unordered_map<int, Node>`), danh sách kề (`unordered_map<int, vector<Edge>>`), danh sách khung giờ (`vector<PeakHour>`).
3. Viết `parseTime`: nhận chuỗi `HH:MM`, trả về giây tính từ nửa đêm. Chuỗi sai định dạng thì trả về 0, không được làm chương trình dừng đột ngột.
4. Trong `main`: nếu thiếu tham số thì thoát với mã lỗi; đổi ba tham số đầu sang số nguyên; tham số giờ là tùy chọn.
5. Bọc phần đổi tham số trong `try/catch`. Tham số không phải số thì in JSON lỗi thay vì để chương trình sập.

**Kiểm tra:** in tạm bốn giá trị đã đọc ra `cerr`; chạy `.\metro.exe 3 27 1 12:00` và thấy `3 27 1 43200`.

### 4.2 Đọc SQLite và dựng đồ thị (Hải Nam)

API C của SQLite luôn theo một khuôn. Đây là khuôn chung, không phải lời giải; ba câu truy vấn và phần xử lý từng dòng do bạn viết:

```cpp
sqlite3* db;
sqlite3_open("metrolink.db", &db);
sqlite3_stmt* stmt = nullptr;
if (sqlite3_prepare_v2(db, "SELECT ...;", -1, &stmt, nullptr) == SQLITE_OK) {
    while (sqlite3_step(stmt) == SQLITE_ROW) {
        // sqlite3_column_int(stmt, 0)
        // sqlite3_column_double(stmt, 1)
        // (const char*)sqlite3_column_text(stmt, 2)
    }
    sqlite3_finalize(stmt);
}
sqlite3_close(db);
```

1. Đọc `KhungGioCaoDiem` vào danh sách khung giờ, dùng `parseTime` cho hai cột giờ.
2. Đọc `Tram` với điều kiện `status = 1` vào bảng nút.
3. Đọc `Ket_Noi` bằng đúng câu truy vấn của `evaluate.py` (gộp theo `u`, `v`, lấy `MIN(travel_time)`). Chỉ thêm cạnh khi cả hai đầu có trong bảng nút.
4. Thêm cạnh đổi tuyến: hai nút khác `id` mà cùng tên ga thì nối với nhau, trọng số 300, `is_transfer` là `true`.
5. In ra `cerr` số nút, số cạnh chạy tàu, số cạnh đổi tuyến, số khung giờ.

**Kết quả đúng:** 192 nút, 378 cạnh chạy tàu, 390 cạnh đổi tuyến, 2 khung giờ.

**Điểm cần hiểu:** `sqlite3_open` không báo lỗi khi file không tồn tại; nó tạo một file rỗng. Khi đó cả ba câu truy vấn đều thất bại, đồ thị rỗng, và mọi cặp ga đều "Not found". Đây là lý do `metro.exe` phải chạy trong thư mục chứa `metrolink.db`.

### 4.3 Hàm `heuristic` (Hải Nam)

1. Viết hàm đổi độ sang radian.
2. Viết `heuristic(u, target)`: khoảng cách Haversine giữa hai nút, bán kính 6.371.000 m, chia cho 20,0.
3. In tạm `heuristic` giữa nút 3 (Altrincham) và nút 27 (Bury) ra `cerr`, so với `haversine_time` của `evaluate.py` cho cùng cặp nút.

**Kết quả đúng:** cả hai bản cho khoảng 1.143,67 giây.

### 4.4 A* (Hải Nam)

Viết `findPathAStar(start_id, goal_id, mode, start_time_sec)`.

1. Nếu `start_id` hoặc `goal_id` không có trong bảng nút: in JSON lỗi và thoát hàm.
2. Khởi tạo `g_score` của mọi nút bằng một số rất lớn.
3. Với mọi nút cùng tên ga với `start_id`: đặt `g_score` bằng 0 và đẩy vào hàng đợi ưu tiên.
4. Hàng đợi là `priority_queue` với `greater<AStarNode>` để phần tử có `f_score` nhỏ nhất ra trước.
5. Vòng lặp chính, theo đúng thứ tự của bản Python:
   - Lấy phần tử đầu.
   - Tên ga của nút bằng tên ga của `goal_id` thì chuyển sang truy vết (4.5).
   - Phần tử lỗi thời thì bỏ qua.
   - Tính thời điểm trong ngày, tra hệ số và giây chờ.
   - Nới lỏng từng cạnh đi ra.
6. Chi phí của cạnh theo chế độ:

   | Chế độ | Cạnh chạy tàu | Cạnh đổi tuyến |
   |---|---|---|
   | 1 (nhanh nhất) | trọng số × hệ số | trọng số × hệ số + giây chờ |
   | 2 (ít đổi tuyến) | 1,0 × hệ số | 10.000 + giây chờ |

7. Hàng đợi rỗng thì in JSON lỗi.

**Gợi ý:** tách việc tra khung giờ thành một hàm riêng, vì 4.5 dùng lại. Bản Madrid chép đoạn này hai lần.

### 4.5 Truy vết và in JSON (Hải Nam)

1. Từ nút đích, lần ngược `came_from` cho tới khi gặp nút không có `came_from`. Nút đó là nút xuất phát thật sự; nó có thể khác `start_id`.
2. Ở mỗi bước lùi từ `curr` về `prev`: tìm cạnh nối `prev` tới `curr`, tính lại chi phí thật theo giây (trọng số × hệ số, cộng giây chờ nếu là đổi tuyến), với hệ số tra theo thời điểm tới `prev`.
3. Ghi (nút, thời gian chặng, có phải đổi tuyến không) vào danh sách; cộng dồn vào tổng thời gian.
4. Thêm nút xuất phát với thời gian chặng bằng 0, rồi đảo ngược danh sách.
5. In JSON đúng hợp đồng ở đầu file. `arrival_time` là tổng tích lũy của `step_time`.

**Kiểm tra:**

```powershell
.\metro.exe 3 27 1 12:00
.\metro.exe 3 27 1 12:00 | python -m json.tool
```

**Kết quả đúng:** lệnh đầu in một dòng bắt đầu bằng `{"status": "success","total_time": 3720,`. Lệnh thứ hai in JSON đã định dạng; nếu báo lỗi thì JSON sai cú pháp. `path` có 24 hoặc 25 phần tử, vì có hai lộ trình cùng 62 phút (xem ghi chú ở nhiệm vụ 3.3).

**Lỗi JSON hay gặp:** dấu phẩy thừa sau phần tử cuối; in `True`/`False` thay vì `true`/`false`; thiếu dấu nháy kép quanh tên ga.

### 4.6 Script đối chiếu C++ với Python (Kiệt)

1. Tạo `backend/compare_cpp.py`, `import` các hàm nạp đồ thị và `search` từ `evaluate.py`.
2. Với mỗi cặp ga có thứ tự: lấy `node_id` đầu tiên của ga đi và ga đến, gọi `metro.exe` bằng `subprocess.run` với đường dẫn tuyệt đối tới file exe, đọc stdout, phân tích JSON.
3. So `total_time` của C++ với kết quả `search` của Python (bật heuristic, cùng giờ khởi hành). Coi là khớp nếu sai khác dưới 0,5 giây.
4. Chạy cho 12:00 và 08:00, chế độ 1. In số cặp khớp và liệt kê tối đa 10 cặp lệch.

Mỗi lần gọi `metro.exe` mất khoảng 20 ms, nên 9.702 cặp chạy trong vài phút.

**Kết quả đúng:** khớp 9.702/9.702 cặp ở cả hai giờ khởi hành.

### 4.7 Thử trường hợp biên (Kiệt)

Chạy từng lệnh, ghi kết quả thật vào cột cuối, báo Hải Nam các dòng không đạt.

| Phép thử | Kết quả mong đợi | Kết quả thật |
|---|---|---|
| Nút không tồn tại: `.\metro.exe 999 27 1 12:00` | JSON lỗi "Not found" | |
| Thiếu giờ: `.\metro.exe 3 27 1` | Chạy như khởi hành 00:00 | |
| Giờ sai định dạng: `.\metro.exe 3 27 1 abc` | Chạy như khởi hành 00:00, không sập | |
| Tham số không phải số: `.\metro.exe a b 1 12:00` | JSON lỗi, không sập | |
| Ga đi trùng ga đến | `path` có 1 phần tử, `total_time` bằng 0 | |
| Chạy từ thư mục khác | JSON lỗi "Not found" | |
| Đóng riêng nút Navy Line của Victoria (`status = 0` trong SQLite), tìm Manchester Airport đi Rochdale Town Centre lúc 12:00 | Vẫn 117 phút, nhưng điểm đổi tuyến không còn là Victoria | |
| Đóng cả 6 nút của Victoria, tìm lại hành trình trên | JSON lỗi "Not found": mọi tàu đi nhánh Rochdale đều qua Victoria | |
| Mở lại Victoria (`status = 1`), tìm lại | Trở lại 117 phút, đổi tuyến tại Victoria | |
| Đóng cả 4 nút của Market Street, tìm Bury đi Piccadilly lúc 12:00 | 47 phút, 1 lần đổi tuyến (bình thường 38 phút, không đổi tuyến) | |
| Mở lại Market Street, tìm lại | Trở lại 38 phút | |
| Chế độ 2, Manchester Airport đi Rochdale Town Centre, 12:00 | 117 phút, 1 lần đổi tuyến | |

Sau khi thử xong, chạy lại `compare_cpp.py` để chắc chắn đã mở lại mọi ga.

### 4.8 Tài liệu hợp đồng (Hải Nam)

Viết `backend/README.md` gồm: cách biên dịch, bảng tham số, cấu trúc JSON, ví dụ một lần đổi tuyến trong `path`, và lưu ý phải chạy trong thư mục `backend`. Hà Nam đọc và xác nhận đủ thông tin để viết `/api/routing` ở tuần 5.

### 4.9 Chuẩn bị (Hà Nam)

Làm trước nhiệm vụ 5.1 trong [tuan5.md](tuan5.md): khởi tạo `backend/package.json`, cài thư viện, tạo `.env`.

## Nâng cấp tùy chọn

Chỉ làm sau khi 4.1–4.6 đã đạt. Mỗi nâng cấp kéo theo việc ở tầng khác.

| Nâng cấp | Việc ở tuần này | Việc kéo theo |
|---|---|---|
| In tên tuyến trong lộ trình | Thêm `line` vào `Node`, đọc cột `line` của bảng `Tram`, in `"line"` trong từng phần tử `path` | `RouteList.jsx` hiển thị tuyến (tuần 6) |
| Chế độ 2 tối ưu số chặng | Dùng `h = 0` khi `mode == 2` trong cả `main.cpp` lẫn `evaluate.py` | Sửa mục "Hạn chế" của báo cáo (tuần 8) |

## Nghiệm thu cuối tuần

- [ ] `g++ main.cpp sqlite3.o -o metro.exe` không báo lỗi.
- [ ] `.\metro.exe 3 27 1 12:00` in `"total_time": 3720`, `path` có 24 hoặc 25 phần tử.
- [ ] stdout là JSON hợp lệ (qua được `python -m json.tool`).
- [ ] `compare_cpp.py`: khớp 9.702/9.702 cặp ở 12:00 và 08:00.
- [ ] Bảng trường hợp biên ở 4.7 đã điền đủ cột "Kết quả thật".
- [ ] `backend/README.md` mô tả đủ hợp đồng; Hà Nam đã xác nhận.

## Câu hỏi cả nhóm phải trả lời được

1. Vì sao thông tin gỡ lỗi in ra `cerr` chứ không phải `cout`?
2. Vì sao chạy `metro.exe` từ thư mục khác thì mọi cặp ga đều "Not found"?
3. `priority_queue` mặc định đưa phần tử lớn nhất ra trước. Chương trình làm gì để phần tử nhỏ nhất ra trước?
4. Ở chế độ 2, `g` đo bằng gì và `h` đo bằng gì? Hệ quả là gì?
5. Vì sao vòng truy vết dừng khi nút không còn `came_from`, thay vì dừng khi gặp `start_id`?
6. Mỗi lần tìm đường, `metro.exe` khởi động lại và đọc lại cả ba bảng. Cách làm này được gì và mất gì?

<details>
<summary>Gợi ý đáp án</summary>

1. `server.js` đưa toàn bộ stdout vào `JSON.parse`. Chỉ một dòng chữ thừa là hỏng.
2. `sqlite3_open` tạo file rỗng ở thư mục hiện tại; các truy vấn thất bại; đồ thị rỗng.
3. Khai báo `priority_queue<AStarNode, vector<AStarNode>, greater<AStarNode>>` và định nghĩa `operator>` so theo `f_score`.
4. `g` đếm số chặng (và 10.000 cho mỗi lần đổi tuyến), còn `h` vẫn tính bằng giây. `h` có thể lớn hơn chi phí thật, nên không còn chấp nhận được. Số lần đổi tuyến vẫn tối thiểu vì mức phạt 10.000 lớn hơn mọi giá trị của `h`, nhưng số chặng có thể nhiều hơn mức tối thiểu.
5. Có nhiều nút xuất phát; đường tốt nhất có thể bắt đầu từ một nút khác `start_id`.
6. Được: thay đổi của quản trị viên (đóng ga, sửa khung giờ) có hiệu lực ngay ở lần tìm kế tiếp, và không cần quản lý tiến trình chạy thường trực. Mất: mỗi lần gọi tốn khoảng 17–20 ms cho khởi động và đọc dữ liệu, lớn hơn nhiều thời gian tìm kiếm trên 192 nút.

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| `undefined reference to sqlite3_open` | Quên `sqlite3.o` trong lệnh biên dịch, hoặc chưa biên dịch `sqlite3.c` |
| `'M_PI' was not declared` | Đang biên dịch với `-std=c++17`. Bỏ cờ đó, dùng `-std=gnu++17`, hoặc tự khai báo hằng số pi |
| Mọi cặp ga đều "Not found" | Chạy sai thư mục, hoặc `metrolink.db` chưa có trong `backend` |
| Altrincham đi Bury ra 67 phút | Chỉ khởi tạo `start_id`, chưa khởi tạo mọi nút cùng tên ga |
| Chương trình treo ở vòng truy vết | Vòng lặp so với `start_id` trong khi đường đi bắt đầu từ nút khác |
| `total_time` lệch bản Python vài giây ở 08:00 | Truy vết tra hệ số theo thời điểm tới `curr` thay vì tới `prev` |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 7.1 và 7.2.
- Bản Madrid: `Project-AI-2025.2-main/backend/main.cpp`. Bản này chỉ xuất phát từ một nút; bốn chỗ khác biệt được liệt kê ở Bước 7.1.
