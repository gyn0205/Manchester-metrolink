# Tuần 1 (05/10–11/10): Dữ liệu GTFS và `data_cleaner.ipynb`

| | |
|---|---|
| Phụ trách chính | **Kiệt** |
| Hỗ trợ và review | Hải Nam, Hà Nam |
| Sản phẩm cuối tuần | `data_pipeline/data_cleaner.ipynb` sinh ra `cleaned_stops.csv` (192 dòng) và `cleaned_stop_times.csv` |
| Cần có trước | Không |

## Quy ước chung cho cả 8 tuần

Phần này chỉ viết một lần ở đây; các file `tuan2.md` đến `tuan8.md` dùng lại.

### Thành viên

| Tên trong kế hoạch | Thành viên | Phụ trách chính (theo mục "Phân công" của báo cáo) |
|---|---|---|
| Hải Nam | Bùi Hải Nam (20235384) | Thuật toán A*, lõi C++, heuristic, thực nghiệm đánh giá, tổng hợp báo cáo |
| Kiệt | Nguyễn Tuấn Kiệt (20235357) | Dữ liệu GTFS, dựng đồ thị, SQLite và MySQL |
| Hà Nam | Nguyễn Hà Nam (20235174) | API Node.js, giao diện React, bản đồ Leaflet, kiểm thử toàn hệ thống |

Mỗi tuần có một người phụ trách chính viết tầng của tuần đó. Hai người còn lại nhận việc review, việc hỗ trợ và việc chuẩn bị cho tầng của mình.

### Lịch

| Tuần | Ngày | Trọng tâm | Phụ trách chính |
|---|---|---|---|
| 1 | 05/10–11/10 | Dữ liệu GTFS, `data_cleaner.ipynb` | Kiệt |
| 2 | 12/10–18/10 | Đồ thị, SQLite, MySQL | Kiệt |
| 3 | 19/10–25/10 | Thuật toán bằng Python; nộp báo cáo giữa kỳ 24/10 | Hải Nam |
| 4 | 26/10–01/11 | Lõi C++ `main.cpp` | Hải Nam |
| 5 | 02/11–08/11 | API `server.js` | Hà Nam |
| 6 | 09/11–15/11 | Giao diện người dùng | Hà Nam |
| 7 | 16/11–22/11 | Đăng nhập và trang quản trị | Hà Nam (chia ba phần) |
| 8 | 23/11–29/11 | Kiểm thử, đánh giá, báo cáo | Hải Nam |
| Dự phòng | 30/11–12/12 | Sửa lỗi, hoàn thiện báo cáo; nộp ngày 12/12 | Cả nhóm |

Trong cột "Hạn" của các bảng phân công, T2 là thứ Hai, T7 là thứ Bảy, CN là Chủ nhật.

### Cách làm việc

1. **Tự viết trước, đối chiếu sau.** [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md) và mã nguồn bản Madrid (`Project-AI-2025.2-main`) là đáp án. Chỉ mở khi bản của bạn đã chạy, hoặc khi bạn kẹt quá 30 phút.
2. **Mỗi người một nhánh theo tuần**, ví dụ `tuan1-kiet`. Thông điệp commit bắt đầu bằng mã nhiệm vụ, ví dụ `1.4 loc mang tram`. Xong thì tạo pull request; người review gộp vào `main`.
3. **Review nghĩa là chạy lại.** Người review phải chạy được trên máy mình và ra đúng số ở mục "Nghiệm thu".
4. **Buổi giảng lại vào Chủ nhật (30–45 phút).** Người phụ trách giải thích code từng đoạn. Hai người còn lại phải trả lời được mục "Câu hỏi cả nhóm phải trả lời được".
5. **Cả nhóm dùng chung một bản feed.** TfGM cập nhật feed hằng đêm; hai bản tải khác ngày có thể cho `node_id` khác nhau.
6. **Số "kết quả đúng" lấy từ feed ngày 03/10/2026.** Nếu bản feed của nhóm cho số khác, ghi số mới vào notebook và báo cả nhóm.
7. **Mã thử nghiệm để ở `backend/thu_nghiem/`.** Thư mục này bị xóa ở tuần 8.

### Các tầng nối với nhau thế nào

```text
GTFS (routes, trips, stop_times, stops)
  │ data_cleaner.ipynb      lọc tram, gộp sân ga thành nút (ga, tuyến)    tuần 1
  ▼
cleaned_stops.csv, cleaned_stop_times.csv
  │ database_builder.ipynb  tạo cạnh, gán node_id                         tuần 2
  ├─► metrolink.db (SQLite) ─► evaluate.py (tuần 3), metro.exe (tuần 4)
  └─► database.sql ─► MySQL ─► server.js (tuần 5)
                                 │ gọi metro.exe, nhận JSON qua stdout
                                 ▼
                               frontend React (tuần 6, 7)
```

Ba "hợp đồng" giữ các tầng khớp nhau. Đổi một trong ba là phải sửa mọi tầng dùng nó:

- **`node_id`:** số nguyên gán ở tuần 2, dùng chung cho SQLite, MySQL, tham số của `metro.exe` và `id` ở frontend.
- **Giao diện của `metro.exe`:** nhận `<nút đi> <nút đến> <chế độ> <HH:MM>`, in một dòng JSON ra stdout.
- **8 endpoint của `server.js`:** frontend chỉ nói chuyện với hệ thống qua các endpoint này.

## Bảng phân công tuần 1

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 1.1 | Cài công cụ, kiểm tra phiên bản | Cả ba, mỗi người trên máy mình | | T2 05/10 |
| 1.2 | Tạo repo git, khung thư mục, `.gitignore` | Hải Nam | Kiệt | T3 06/10 |
| 1.3 | Tải feed, chia sẻ cho nhóm, khảo sát 4 file GTFS | Kiệt | Hải Nam | T4 07/10 |
| 1.4 | Notebook ô 1–2: đọc dữ liệu, lọc mạng tram | Kiệt | Hà Nam | T5 08/10 |
| 1.5 | Notebook ô 3: gộp sân ga thành ga, tạo nút (ga, tuyến) | Kiệt | Hải Nam | T6 09/10 |
| 1.6 | Notebook ô 4: lưu CSV và tự kiểm tra | Kiệt | Hải Nam | T7 10/10 |
| 1.7 | Chạy lại notebook trên máy mình, so số liệu | Hải Nam, Hà Nam | | CN 11/10 |
| 1.8 | Chuẩn bị cho tuần 3: Dijkstra trên đồ thị mẫu | Hải Nam | | CN 11/10 |
| 1.9 | Chuẩn bị cho tuần 5: máy chủ Express tối thiểu | Hà Nam | | CN 11/10 |
| 1.10 | Buổi giảng lại tuần 1 | Kiệt trình bày | Cả nhóm | CN 11/10 |

## Hướng dẫn chi tiết

### 1.1 Cài công cụ (cả ba)

1. Cài và kiểm tra từng công cụ:

   | Công cụ | Lệnh kiểm tra | Dùng từ tuần |
   |---|---|---|
   | Python 3.11 trở lên | `python --version` | 1 |
   | pandas, ipykernel | `python -m pip install pandas ipykernel` | 1 |
   | Git | `git --version` | 1 |
   | VS Code và extension Jupyter | Mở được một file `.ipynb` | 1 |
   | MySQL Server 8 và Workbench | `Get-Service MySQL80` | 2 |
   | gcc, g++ (MinGW) | `g++ --version` | 4 |
   | Node.js 22 | `node --version` | 5 |

2. Gửi kết quả các lệnh trên vào nhóm chat để biết máy ai thiếu gì.
3. Riêng máy Hải Nam: Docker Desktop đang chiếm cổng 3000, và máy có cả MySQL80 lẫn MySQL của XAMPP. Chỉ bật một MySQL tại một thời điểm.

### 1.2 Tạo repo và khung thư mục (Hải Nam)

1. Chạy `git init` trong thư mục `Manchester metrolink`.
2. Tạo `backend/`, `data_pipeline/raw_data/`, `data_pipeline/processed_data/`. Thư mục `frontend/` để tuần 6 tạo bằng Vite.
3. Tự viết `.gitignore`. Với mỗi dòng, bạn phải nói được vì sao bỏ qua:

   | Cần bỏ qua | Lý do |
   |---|---|
   | `node_modules/`, `dist/` | Sinh lại được bằng `npm install` và `npm run build` |
   | `*.exe`, `*.o` | Mỗi máy tự biên dịch |
   | `.env` | Chứa mật khẩu MySQL và khóa JWT |
   | `data_pipeline/raw_data/` | Feed nặng khoảng 250 MB sau khi giải nén |
   | `data_pipeline/processed_data/cleaned_stop_times.csv` | Khoảng 14 MB, sinh lại được |

4. Commit đầu tiên gồm `HUONG_DAN_BUILD.md`, `plan/`, `report/` và `.gitignore`.
5. Tạo repo riêng tư trên GitHub, đẩy lên, mời Kiệt và Hà Nam.
6. Kiệt và Hà Nam clone về, mỗi người tạo nhánh của mình.

**Kiểm tra:** `git status` sạch; cả ba người nhìn thấy cùng một commit trên `main`.

### 1.3 Tải feed và khảo sát GTFS (Kiệt)

1. Tải và giải nén:

   ```powershell
   cd "C:\Project Ai\Manchester metrolink\data_pipeline"
   curl.exe -L -o tfgm.zip "https://odata.tfgm.com/opendata/downloads/TfGMgtfsnew.zip"
   Expand-Archive tfgm.zip -DestinationPath raw_data -Force
   ```

2. Đổi tên file zip kèm ngày tải (ví dụ `tfgm_2026-10-05.zip`) và đưa lên Drive của nhóm. Hải Nam và Hà Nam dùng đúng file này, không tự tải.
3. Kiểm tra `raw_data` có 9 file `.txt`, trong đó `stop_times.txt` khoảng 160 MB.
4. Tạo notebook nháp `data_pipeline/khao_sat.ipynb`. Với từng file trong 4 file `routes.txt`, `trips.txt`, `stop_times.txt`, `stops.txt`: đọc 5 dòng đầu, in tên cột, đếm số dòng.
5. Trả lời các câu hỏi sau bằng code trong notebook:

   | Câu hỏi | Kết quả đúng |
   |---|---|
   | `routes.txt` có bao nhiêu dòng? Bao nhiêu dòng có `route_type` bằng 0? | 656 dòng; 12 dòng type 0, trong đó 4 dòng là "Replacement bus" |
   | 8 `route_id` còn lại ứng với bao nhiêu tên tuyến (`route_short_name`)? | 7 tuyến: Blue, Green, Navy, Pink, Purple, Red, Yellow |
   | `trips.txt`, `stop_times.txt`, `stops.txt` có bao nhiêu dòng? | 69.680; 2.902.195; 15.693 |
   | Giờ lớn nhất trong `arrival_time` của tram là bao nhiêu? | Lớn hơn 24:00:00 (tới `26:12:30`) |
   | Tên các điểm dừng tram có gì chung? | Đuôi " (Manchester Metrolink)" |
   | Hai sân ga của cùng một ga có `stop_code` giống nhau ở đâu? | 11 ký tự đầu giống nhau; ký tự cuối là số sân ga |

6. Vẽ ra giấy sơ đồ khóa: `routes` nối `trips` qua `route_id`; `trips` nối `stop_times` qua `trip_id`; `stop_times` nối `stops` qua `stop_id`.
7. Chọn một `trip_id` của tram, in các điểm dừng của nó theo `stop_sequence` kèm giờ đến. Đây chính là một chuyến tàu thật; cạnh của đồ thị ở tuần 2 sinh ra từ các cặp dòng liên tiếp này.

**Gợi ý:** `stop_times.txt` rất lớn, nên chỉ đọc các cột cần bằng `usecols`, và ép kiểu chuỗi cho cột mã và cột giờ bằng `dtype`.

### 1.4 Ô 1 và ô 2: đọc dữ liệu, lọc mạng tram (Kiệt)

Tạo `data_pipeline/data_cleaner.ipynb`, chọn kernel Python 3.11.

**Ô 1, đọc dữ liệu:**

1. Đọc `routes.txt` và `stops.txt` (cột `stop_code` ép kiểu chuỗi).
2. Đọc `trips.txt`, chỉ lấy `route_id` và `trip_id`, kiểu chuỗi.
3. Đọc `stop_times.txt`, chỉ lấy `trip_id`, `arrival_time`, `departure_time`, `stop_id`, `stop_sequence`.
4. In số dòng của cả bốn bảng.

**Ô 2, lọc tram:**

1. Lọc `routes` theo hai điều kiện cùng lúc: `route_type` bằng 0, và `route_short_name` không chứa chữ "replacement" (không phân biệt hoa thường).
2. Nối `trips` với các tuyến tram để mỗi chuyến biết tên tuyến; đặt tên cột là `line`.
3. Nối `stop_times` với các chuyến tram để chỉ giữ dòng của tram và gắn `line` vào từng dòng.
4. Bỏ các dòng thiếu `arrival_time` hoặc `departure_time`.
5. In danh sách tuyến, số chuyến và số dòng `stop_times` còn lại.

**Gợi ý:** mặt nạ boolean kết hợp bằng `&` và `~`; `Series.str.contains(..., case=False)`; `DataFrame.merge(..., on=...)` mặc định là inner join, nên bản thân phép nối đã lọc dữ liệu.

**Kết quả đúng:**

```text
routes: 656 | trips: 69680 | stop_times: 2902195 | stops: 15693
Tuyến tram: ['Blue Line', 'Green Line', 'Navy Line', 'Pink Line', 'Purple Line', 'Red Line', 'Yellow Line']
Chuyến tram: 14745 | dòng stop_times tram: 273290
```

### 1.5 Ô 3: gộp sân ga thành ga, tạo nút (ga, tuyến) (Kiệt)

Đây là ô quan trọng nhất của tuần. Ở Manchester, mỗi `stop_id` là một sân ga theo chiều, và nhiều tuyến dùng chung sân ga. Vì vậy `stop_id` không cho biết hành khách đang ở tuyến nào.

1. Lấy các điểm dừng có xuất hiện trong `stop_times` của tram.
2. Tạo cột `station_code` bằng 11 ký tự đầu của `stop_code`.
3. Cắt đuôi " (Manchester Metrolink)" khỏi `stop_name` rồi bỏ khoảng trắng thừa.
4. Gộp theo `station_code` để được bảng ga: tên lấy giá trị đầu, vĩ độ và kinh độ lấy trung bình các sân ga.
5. Gắn `station_code` vào từng dòng `stop_times` của tram.
6. Thay `stop_id` bằng khóa nút dạng `<mã ga>|<tuyến>`.
7. Lập bảng nút: lấy các bộ (`stop_id`, `station_code`, `line`) không trùng, nối với bảng ga để có tên và tọa độ, sắp theo `stop_name` rồi `line`, đánh lại chỉ số.

**Gợi ý:** `Series.str[:11]`; `str.replace(..., regex=False)`; `groupby(...).agg(...)`; `drop_duplicates()`; `sort_values([...]).reset_index(drop=True)`.

**Vì sao bước 7 phải sắp xếp:** tuần 2 gán `node_id` theo thứ tự dòng của bảng này. Thứ tự ổn định thì `node_id` giống nhau trên máy của cả ba người.

**Kết quả đúng:**

```text
Sân ga: 199 | Ga: 99
Nút (ga, tuyến): 192
```

### 1.6 Ô 4: lưu CSV và tự kiểm tra (Kiệt)

1. Tạo thư mục `processed_data` nếu chưa có.
2. Lưu bảng nút thành `cleaned_stops.csv` với các cột `stop_id`, `stop_name`, `stop_lat`, `stop_lon`, `line`, `station_code`.
3. Lưu `stop_times` của tram thành `cleaned_stop_times.csv` với các cột `trip_id`, `arrival_time`, `departure_time`, `stop_id`, `stop_sequence`.
4. Không trừ 24 giờ ở các giờ lớn hơn 24:00. Bản Madrid có trừ, và sinh ra thời gian chạy âm.
5. Thêm một ô kiểm tra, mỗi ý là một lệnh `assert`:
   - `stop_id` của bảng nút không trùng nhau.
   - Không có ô nào bị thiếu trong bảng nút.
   - Có đúng 7 tuyến.
   - Mỗi ga có một tên riêng (99 tên cho 99 ga).
   - Vĩ độ nằm trong 53,3–53,7 và kinh độ nằm trong −2,5 đến −2,0.

**Kết quả đúng:** `processed_data` có `cleaned_stops.csv` (192 dòng dữ liệu) và `cleaned_stop_times.csv` (khoảng 14 MB).

### 1.7 Chạy lại trên máy mình (Hải Nam, Hà Nam)

1. Lấy file zip của nhóm trên Drive, giải nén vào `data_pipeline/raw_data`.
2. Kéo nhánh của Kiệt về, bấm Run All.
3. So bốn dòng số liệu ở 1.4 và 1.5 với máy Kiệt. Lệch ở đâu thì báo ngay.
4. Mở `cleaned_stops.csv`, tìm ga St Peter's Square. Ga này phải xuất hiện 7 dòng, mỗi dòng một tuyến.

### 1.8 Chuẩn bị: Dijkstra trên đồ thị mẫu (Hải Nam)

Mục đích là nắm thuật toán trước khi có dữ liệu thật ở tuần 3.

1. Tạo `backend/thu_nghiem/toy_search.py`.
2. Khai báo đồ thị có hướng bằng `dict`: `A→B 4`, `A→C 2`, `C→B 1`, `B→D 5`, `C→D 8`, `C→E 10`, `D→E 2`, `D→F 6`, `E→F 3`.
3. Viết `dijkstra(graph, start, goal)` dùng `heapq`: hàng đợi chứa cặp (chi phí, nút); lấy nút chi phí nhỏ nhất; bỏ qua phần tử đã lỗi thời; nới lỏng các cạnh đi ra; ghi `came_from` để truy vết.
4. Trả về tổng chi phí và danh sách nút trên đường đi.

**Kết quả đúng:** từ A đến F chi phí 13, đường đi A, C, B, D, E, F.

### 1.9 Chuẩn bị: máy chủ Express tối thiểu (Hà Nam)

1. Trong `backend/thu_nghiem/`, chạy `npm init -y` rồi `npm install express`.
2. Viết `hello.js`: tạo app Express, một route `GET /api/ping` trả về JSON `{ "ok": true }`, lắng nghe ở một cổng đọc từ biến môi trường `PORT` (mặc định 3000).
3. Chạy `node hello.js`, mở `http://localhost:3000/api/ping` trên trình duyệt.

**Kết quả đúng:** trình duyệt hiện `{"ok":true}`.

### 1.10 Buổi giảng lại (Kiệt trình bày)

Kiệt mở notebook, chạy từng ô và giải thích. Hải Nam và Hà Nam trả lời các câu hỏi dưới đây.

## Nghiệm thu cuối tuần

- [ ] Repo có trên GitHub, cả ba người clone và đẩy được.
- [ ] Cả ba máy dùng cùng một file zip feed.
- [ ] `data_cleaner.ipynb` chạy Run All không lỗi trên cả ba máy.
- [ ] Số liệu khớp: 7 tuyến, 14.745 chuyến, 273.290 dòng, 199 sân ga, 99 ga, 192 nút.
- [ ] `toy_search.py` trả về chi phí 13.
- [ ] `hello.js` trả về `{"ok":true}`.

## Câu hỏi cả nhóm phải trả lời được

1. Một chuyến (trip) nối với tuyến và với các điểm dừng qua những khóa nào?
2. Vì sao lọc `route_type == 0` là chưa đủ?
3. Vì sao nút của đồ thị là cặp (ga, tuyến), không phải ga, cũng không phải sân ga?
4. Vì sao không quy giờ lớn hơn 24:00 về khoảng 0–24?
5. Thứ tự dòng trong `cleaned_stops.csv` ảnh hưởng tới điều gì ở tuần 2?

<details>
<summary>Gợi ý đáp án</summary>

1. `trips.route_id` trỏ tới `routes`; `stop_times.trip_id` trỏ tới `trips`; `stop_times.stop_id` trỏ tới `stops`.
2. Bốn tuyến xe buýt thay thế cũng mang `route_type` 0.
3. Đổi tuyến phải tốn thời gian, nên trạng thái phải ghi cả tuyến đang đi. Sân ga không dùng được vì nhiều tuyến chung một sân ga. Ga không dùng được vì không phân biệt được "ở lại tuyến cũ" với "đổi tuyến".
4. Chuyến qua nửa đêm có giờ đến nhỏ hơn giờ đi nếu quy về 0–24, làm thời gian chạy bị âm.
5. `node_id` được gán theo thứ tự dòng. Thứ tự khác thì `node_id` khác, và SQLite, MySQL, frontend sẽ không còn nói về cùng một nút.

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| `No module named 'pandas'` | Notebook đang dùng kernel khác. Chọn lại kernel Python đã cài pandas ở góc trên phải |
| Đọc `stop_times.txt` rất chậm hoặc hết bộ nhớ | Thiếu `usecols`; chỉ đọc 5 cột cần dùng |
| `FileNotFoundError: raw_data/...` | Notebook không nằm trong `data_pipeline`, hoặc chưa giải nén feed |
| Số ga khác 99 | Cắt `stop_code` sai số ký tự, hoặc `stop_code` bị đọc thành số; kiểm tra `dtype` |
| Số liệu lệch giữa các máy | Dùng bản feed khác nhau. Lấy lại file zip trên Drive của nhóm |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 1 đến Bước 4.
- Bản Madrid: `Project-AI-2025.2-main/data_pipeline/data_cleaner.ipynb`. Bản này lọc `route_type == 1` và có trừ 24 giờ, nên chỉ dùng để so cách tổ chức.
