# Tuần 8 (23/11–29/11): Kiểm thử, đánh giá, báo cáo

| | |
|---|---|
| Phụ trách chính | **Hải Nam** (đánh giá, tổng hợp báo cáo) |
| Cùng làm | **Hà Nam** (kiểm thử toàn hệ thống), **Kiệt** (chốt dữ liệu, thử cài đặt từ đầu) |
| Sản phẩm cuối tuần | Hệ thống qua hết bảng kiểm thử; README đủ để người ngoài cài và chạy; mục 4 của báo cáo điền bằng số liệu thật |
| Cần có trước | Toàn bộ sản phẩm của tuần 1 đến tuần 7 |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

Tuần này không viết tính năng mới. Sau ngày 29/11 còn hai tuần dự phòng trước hạn nộp 12/12.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 8.1 | Chốt dữ liệu: chạy lại pipeline, ghi số liệu cuối | Kiệt | Hải Nam | T2 23/11 |
| 8.2 | Kiểm thử toàn hệ thống theo bảng | Hà Nam | Kiệt | T4 25/11 |
| 8.3 | Chạy đánh giá cuối cùng | Hải Nam | Kiệt | T4 25/11 |
| 8.4 | Sửa lỗi tìm thấy ở 8.2 và 8.3 | Người phụ trách tầng có lỗi | Hà Nam thử lại | T6 27/11 |
| 8.5 | Viết `README.md` ở thư mục gốc | Kiệt (dữ liệu, CSDL), Hải Nam (thuật toán), Hà Nam (backend, frontend) | | T6 27/11 |
| 8.6 | Thử cài đặt từ đầu theo README | Kiệt | Hà Nam | T7 28/11 |
| 8.7 | Cập nhật mục 4 của báo cáo | Hải Nam tổng hợp; mỗi người viết phần mình | Cả nhóm đọc soát | T7 28/11 |
| 8.8 | Dọn repo | Hà Nam | Hải Nam | T7 28/11 |
| 8.9 | Mỗi người vẽ lại sơ đồ kiến trúc từ trí nhớ | Cả ba | | CN 29/11 |
| 8.10 | Chuẩn bị trình bày | Cả ba | | CN 29/11 |

## Hướng dẫn chi tiết

### 8.1 Chốt dữ liệu (Kiệt)

1. Dùng đúng file zip feed của nhóm. Chạy lại `data_cleaner.ipynb` và `database_builder.ipynb` từ đầu.
2. Chép `metrolink.db` vào `backend/`, chuyển `database.sql` ra thư mục gốc, import lại MySQL.
3. Lập bảng số liệu cuối cho báo cáo, mỗi số lấy từ kết quả in của notebook:

   | Đại lượng | Số theo feed 03/10 | Số của nhóm |
   |---|---|---|
   | Tuyến, chuyến, dòng `stop_times`, điểm dừng (toàn feed) | 656; 69.680; 2.902.195; 15.693 | |
   | Tuyến tram, chuyến tram, dòng `stop_times` tram | 7; 14.745; 273.290 | |
   | Sân ga, ga, nút | 199; 99; 192 | |
   | Cạnh chạy tàu, cạnh đổi tuyến, ga trung chuyển | 378; 390; 44 | |
   | Thời gian chạy nhỏ nhất, lớn nhất, trung bình (giây) | 60; 360; 158,1 | |
   | Dòng giờ từ 24:00 trở lên; cặp ga liên tiếp vắt qua nửa đêm | 10.324; 442 | |
   | Vận tốc đường thẳng lớn nhất | 14,12 m/s (Whitefield đi Radcliffe) | |

4. Báo cho Hải Nam và Hà Nam khi xong, vì hai nhiệm vụ 8.2 và 8.3 phải chạy trên bộ dữ liệu này.

**Lưu ý:** import lại `database.sql` không xóa bảng `users`, nên tài khoản admin vẫn còn.

### 8.2 Kiểm thử toàn hệ thống (Hà Nam)

Khởi động theo thứ tự: MySQL, `node server.js`, `npm run dev`. Kiểm tra trước khi thử: 192 nút đều mở, còn đúng 2 khung giờ mặc định, `node check_sync.js` in "ĐỒNG BỘ".

Ghi "Đạt" hoặc mô tả lỗi vào cột cuối. Mỗi lỗi giao cho người phụ trách tầng tương ứng ở nhiệm vụ 8.4.

**A. Tìm đường**

| # | Ga đi | Ga đến | Chế độ | Giờ | Kết quả đúng | Kết quả thật |
|---|---|---|---|---|---|---|
| A1 | Altrincham | Bury | Nhanh nhất | 12:00 | 62 phút, không đổi tuyến | |
| A2 | Manchester Airport | Rochdale Town Centre | Nhanh nhất | 12:00 | 117 phút, đổi tuyến tại Victoria | |
| A3 | St Peter's Square | Manchester Airport | Nhanh nhất | 12:00 | 51,5 phút, không đổi tuyến | |
| A4 | The Trafford Centre | East Didsbury | Nhanh nhất | 12:00 | 42 phút, đổi tuyến tại Trafford Bar | |
| A5 | Eccles | Ashton-under-Lyne | Nhanh nhất | 12:00 | 68 phút, không đổi tuyến | |
| A6 | Altrincham | Bury | Nhanh nhất | 08:00 | 93 phút | |
| A7 | Manchester Airport | Rochdale Town Centre | Nhanh nhất | 08:00 | 150,5 phút, đổi tuyến tại St Peter's Square | |
| A8 | The Trafford Centre | East Didsbury | Nhanh nhất | 08:00 | 68 phút | |
| A9 | Manchester Airport | Rochdale Town Centre | Ít chuyển | 12:00 | 117 phút, 1 lần đổi tuyến | |
| A10 | Victoria | Victoria | Nhanh nhất | 12:00 | 0 phút, hoặc giao diện chặn và báo hai ga trùng nhau | |

**B. Giao diện người dùng**

| # | Phép thử | Kết quả đúng | Kết quả thật |
|---|---|---|---|
| B1 | Gõ vài chữ vào ô ga | Gợi ý tên ga, không tên nào lặp | |
| B2 | Nhập tên ga không tồn tại rồi tìm | Có thông báo, không gửi yêu cầu | |
| B3 | Tìm một hành trình có đổi tuyến | Danh sách ghi "Chờ đổi tuyến"; bản đồ có biểu tượng đổi tuyến | |
| B4 | Tìm hai hành trình liên tiếp | Bản đồ xóa lộ trình cũ và thu phóng theo lộ trình mới | |
| B5 | Tắt `server.js` rồi tìm | Có thông báo lỗi; nút tìm không bị kẹt | |
| B6 | Xem khung chú thích | Có dòng "Contains Transport for Greater Manchester data" | |

**C. Đăng nhập và phân quyền:** chạy lại năm phép thử của nhiệm vụ 7.8.

**D. Quản trị:** chạy lại chín dòng của nhiệm vụ 7.9.

**E. API trực tiếp:** chạy `backend/thu_api.ps1` của nhiệm vụ 5.9. Mọi phép thử phải đạt.

Sau khi thử xong: mở lại mọi ga, xóa các khung giờ thử, chạy `check_sync.js`.

### 8.3 Đánh giá cuối cùng (Hải Nam)

Chạy trên bộ dữ liệu đã chốt ở 8.1.

1. `python evaluate.py` và `python evaluate.py 15`. Ghi lại cho cả 12:00 và 08:00: số cặp A* trùng Dijkstra, số nút mở rộng trung bình của hai thuật toán, thời gian hành trình trung bình, trung vị, dài nhất, phân bố số lần đổi tuyến.
2. `python compare_cpp.py`: số cặp mà C++ khớp Python ở 12:00 và 08:00.
3. Đo lại thời gian một lần gọi `metro.exe` như nhiệm vụ 5.10.
4. Chạy lại phân tích chế độ 2 của nhiệm vụ 5.10.
5. So từng con số với bản nháp mục 4 của báo cáo. Số nào khác thì thay bằng số mới đo.

**Kết quả đúng** (feed 03/10, 20 m/s): A* trùng Dijkstra ở 9.702/9.702 cặp; số nút mở rộng 91,6 so với 108,4 lúc 12:00 và 95,3 so với 107,2 lúc 08:00; thời gian trung bình 41,1 phút lúc 12:00 và 63,2 phút lúc 08:00; 3.876 cặp đi thẳng và 5.826 cặp đổi tuyến một lần; C++ khớp Python ở mọi cặp.

### 8.4 Sửa lỗi

| Lỗi nằm ở | Người sửa |
|---|---|
| Notebook, `metrolink.db`, `database.sql` | Kiệt |
| `main.cpp`, `evaluate.py` | Hải Nam |
| `server.js`, `frontend/` phần tìm đường và đăng nhập | Hà Nam |
| `StationTable`, `Toast` | Kiệt |
| `PeakHoursAdmin` | Hải Nam |

Mỗi lỗi sửa xong, Hà Nam chạy lại đúng phép thử đã phát hiện ra nó. Sửa `main.cpp` thì chạy lại `compare_cpp.py`.

### 8.5 `README.md` (cả ba)

README viết cho một người chưa từng thấy project. Viết bằng lời của nhóm, không chép từ file hướng dẫn.

| Phần | Người viết | Nội dung |
|---|---|---|
| Giới thiệu, sơ đồ kiến trúc | Hải Nam | Bài toán, sơ đồ các tầng, bảng công nghệ |
| Dữ liệu | Kiệt | Nguồn feed, giấy phép, cách chạy hai notebook, ba việc phải làm sau mỗi lần chạy lại |
| Cơ sở dữ liệu | Kiệt | Các bảng của SQLite và MySQL, cách import, cách tạo tài khoản admin |
| Thuật toán | Hải Nam | Cách biên dịch `metro.exe`, hợp đồng dòng lệnh và JSON, cách chạy `evaluate.py` |
| Backend | Hà Nam | Các biến trong `.env`, cách chạy `server.js`, bảng 8 endpoint |
| Frontend | Hà Nam | Cách cài và chạy, các đường dẫn `/auth`, `/user`, `/admin` |
| Lỗi thường gặp | Cả ba | Gom từ mục "Lỗi thường gặp" của tám file kế hoạch, giữ những lỗi nhóm đã thật sự gặp |

### 8.6 Thử cài đặt từ đầu (Kiệt)

1. Clone repo vào một thư mục mới. Không chép gì từ thư mục đang làm việc.
2. Làm đúng từng bước trong README, không dựa vào trí nhớ.
3. Mỗi chỗ README thiếu, sai hoặc khó hiểu: ghi lại và báo người viết phần đó sửa.
4. Kết thúc khi tìm được Altrincham đi Bury trên web ở bản cài mới.

**Kết quả đúng:** từ lúc clone tới lúc tìm được đường chỉ dùng README.

### 8.7 Báo cáo cuối kỳ (Hải Nam tổng hợp)

Sửa mục "4. Cập nhật kết quả cuối kỳ (W15)" trong `report/Project report N12.ipynb`.

| Phần của mục 4 | Người viết | Việc cần làm |
|---|---|---|
| Chi tiết phương pháp, dữ liệu | Kiệt (dữ liệu, đồ thị); Hải Nam (chi phí, A*, tiêu chí ít đổi tuyến) | Thay số bằng bảng ở 8.1; ghi `MAX_SPEED` đã chốt ở 6.8 |
| Chương trình | Hà Nam | Sơ đồ kiến trúc, bảng chức năng, bảng API đúng với bản nhóm đã viết |
| Phân tích, đánh giá kết quả | Hải Nam | Thay mọi con số bằng kết quả ở 8.3 |
| Hạn chế, hướng phát triển | Hải Nam | Sửa theo những gì nhóm đã thật sự làm (xem bảng dưới) |
| Phân công, tỷ lệ đóng góp | Cả ba thống nhất | Ghi công việc thật của từng người và điền tỷ lệ |

Các câu trong bản nháp phải soát lại vì nhóm đã làm khác bản Madrid:

| Câu trong bản nháp | Sửa thế nào |
|---|---|
| "API quản trị chưa kiểm tra token ở phía server" | Bỏ khỏi "Hạn chế" nếu đã làm nhiệm vụ 5.7; mô tả middleware ở phần "Chương trình" |
| "Thêm kiểm tra token cho API quản trị" ở "Hướng phát triển" | Bỏ nếu đã làm |
| Hạn chế về tiêu chí ít đổi tuyến (605 cặp) | Giữ và thay số nếu chưa sửa; viết lại nếu đã làm nâng cấp "chế độ 2 tối ưu số chặng" |
| "Thao tác đóng ga áp dụng cho từng nút" | Giữ nếu chưa làm nút "Đóng cả ga" |
| Thời gian chạy 17–20 ms | Thay bằng số đo ở 8.3 |

Thêm vào "Hạn chế": feed không ghi thời gian tàu dừng ở ga (kết quả của nhiệm vụ 2.2).

Xóa mọi ghi chú nháp trong các ô trước khi nộp.

### 8.8 Dọn repo (Hà Nam)

1. Xóa `backend/thu_nghiem/` sau khi chuyển những gì còn dùng ra ngoài.
2. Kiểm tra `.env` không nằm trong lịch sử commit. Nếu đã lỡ commit, đổi mật khẩu MySQL và khóa JWT.
3. Tìm trong toàn bộ mã nguồn các chữ "Madrid", "metro_madrid", "localhost:3000". Không được còn, trừ `vite.config.js` và file `.env.example`.
4. Khôi phục `backend/metrolink.db` về trạng thái sau nhiệm vụ 8.1 (192 nút mở, 2 khung giờ) rồi mới commit.
5. `git status` sạch trên `main`; cả ba máy kéo về và chạy được.

### 8.9 Vẽ lại sơ đồ từ trí nhớ (cả ba)

Mỗi người, không mở tài liệu, tự vẽ ra giấy:

1. Sơ đồ các tầng từ feed GTFS tới bản đồ, ghi tên file ở mỗi tầng.
2. Đường đi của một lần bấm "Tìm đường": từng hàm, từng file, từng định dạng dữ liệu.
3. Đường đi của một lần gạt công tắc đóng ga, cho tới khi kết quả tìm đường thay đổi.
4. Vòng đời của token.

So ba bản vẽ với nhau và với sơ đồ trong [tuan1.md](tuan1.md). Chỗ nào ai vẽ thiếu thì người phụ trách tầng đó giảng lại.

### 8.10 Chuẩn bị trình bày (cả ba)

1. Kịch bản chạy thử, khoảng 5 phút:
   - Tìm Altrincham đi Bury lúc 12:00 (62 phút), rồi lúc 08:00 (93 phút), để cho thấy chi phí phụ thuộc thời gian.
   - Tìm Manchester Airport đi Rochdale Town Centre lúc 12:00 và 08:00, để cho thấy điểm đổi tuyến thay đổi.
   - Đăng nhập admin, đóng Market Street, tìm Bury đi Piccadilly (38 phút thành 47 phút).
   - Chạy `evaluate.py` và chỉ vào dòng 9.702/9.702.
2. Mỗi người chuẩn bị trả lời các "Câu hỏi cả nhóm phải trả lời được" của mọi tuần, không chỉ tuần mình phụ trách.
3. Tập chạy thử một lần trên máy sẽ dùng để trình bày. Kiểm tra trước: MySQL đang chạy, cổng 3000 trống, hai CSDL đồng bộ.

## Nghiệm thu cuối tuần

- [ ] Bảng số liệu ở 8.1 đã điền cột "Số của nhóm".
- [ ] Các nhóm phép thử A, B, C, D, E ở 8.2 đều đạt, hoặc lỗi còn lại đã được ghi vào mục "Hạn chế".
- [ ] A* trùng Dijkstra và C++ khớp Python ở mọi cặp ga.
- [ ] Kiệt cài được từ đầu chỉ bằng README.
- [ ] Mục 4 của báo cáo không còn số nào lấy từ bản nháp mà chưa kiểm tra lại; không còn ghi chú nháp.
- [ ] Bảng phân công trong báo cáo đã điền tỷ lệ đóng góp, cả ba đồng ý.
- [ ] Repo sạch: không `.env`, không `thu_nghiem/`, không còn chữ "Madrid".
- [ ] Cả ba vẽ được bốn sơ đồ ở 8.9.

## Hai tuần dự phòng (30/11–12/12)

| Thời gian | Việc |
|---|---|
| 30/11–06/12 | Sửa các lỗi còn lại; làm nâng cấp tùy chọn nếu cả nhóm đồng ý; chạy lại 8.2 và 8.3 sau mỗi thay đổi |
| 07/12–10/12 | Đọc soát báo cáo lần cuối; không sửa code trừ lỗi nghiêm trọng |
| 11/12 | Chạy lại toàn bộ notebook báo cáo, xuất bản nộp |
| 12/12 | Nộp |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 9 và Bước 10.
- `report/Project report N12.ipynb`, mục 4: bản nháp viết ngày 03/10/2026 từ lần chạy thử, dùng làm khung và để so số liệu.
