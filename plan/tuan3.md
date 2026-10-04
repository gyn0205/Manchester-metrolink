# Tuần 3 (19/10–25/10): Thuật toán bằng Python và báo cáo giữa kỳ

| | |
|---|---|
| Phụ trách chính | **Hải Nam** |
| Hỗ trợ và review | Kiệt (số liệu dữ liệu, vận tốc đường thẳng), Hà Nam (review, đọc soát báo cáo) |
| Sản phẩm cuối tuần | `backend/evaluate.py` chạy Dijkstra và A* trên toàn bộ 9.702 cặp ga; báo cáo giữa kỳ nộp ngày 24/10 |
| Cần có trước | `backend/metrolink.db` của tuần 2 |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

## Vì sao viết bằng Python trước

Thuật toán viết bằng Python ngắn và dễ gỡ lỗi. Khi bản Python đã đúng, tuần 4 chỉ còn là chuyển sang C++, và bản Python trở thành thước đo để kiểm tra bản C++. File `evaluate.py` cũng là công cụ sinh số liệu cho báo cáo.

**Lưu ý về báo cáo giữa kỳ:** bản nháp mục 3 của báo cáo đang ghi `backend/main.cpp` là "Hoàn thành, chạy từ dòng lệnh". Theo lịch này, đến ngày 24/10 nhóm mới có bản Python; bản C++ viết ở tuần 4. Nhiệm vụ 3.8 sửa lại dòng đó cho đúng thực tế.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 3.1 | Nạp đồ thị từ SQLite vào Python | Hải Nam | Kiệt | T2 19/10 |
| 3.2 | Dijkstra nhiều nguồn, kiểm tra đích theo tên ga | Hải Nam | Kiệt | T3 20/10 |
| 3.3 | Cạnh đổi tuyến, đếm số lần đổi, truy vết đường đi | Hải Nam | Kiệt | T3 20/10 |
| 3.4 | Chi phí phụ thuộc thời gian | Hải Nam | Hà Nam | T4 21/10 |
| 3.5 | Heuristic Haversine và A* | Hải Nam | Hà Nam | T5 22/10 |
| 3.6 | Tính vận tốc đường thẳng lớn nhất trên mọi cạnh | Kiệt | Hải Nam | T4 21/10 |
| 3.7 | Chạy toàn bộ cặp ga, so A* với Dijkstra | Hải Nam | Kiệt | T6 23/10 |
| 3.8 | Cập nhật mục 3 của báo cáo và nộp | Hải Nam tổng hợp; Kiệt viết phần dữ liệu; Hà Nam đọc soát | | T6 23/10, nộp T7 24/10 |
| 3.9 | Chuẩn bị cho tuần 4: biên dịch được SQLite với g++ | Kiệt | | CN 25/10 |
| 3.10 | Chuẩn bị cho tuần 5: bcrypt và JWT | Hà Nam | | CN 25/10 |
| 3.11 | Buổi giảng lại tuần 3 | Hải Nam trình bày | Cả nhóm | CN 25/10 |

## Hướng dẫn chi tiết

Viết `backend/evaluate.py` theo từng bước. Sau mỗi bước, chạy và so với "Kết quả đúng" rồi mới làm bước kế.

Tổ chức file thành các hàm, phần chạy thử đặt trong khối `if __name__ == '__main__':`. Tuần 4 có một script khác `import` các hàm này.

### 3.1 Nạp đồ thị (Hải Nam)

1. Đọc bảng `Tram`, chỉ lấy nút có `status = 1`, vào một `dict`: `node_id` → (tên ga, vĩ độ, kinh độ).
2. Đọc bảng `Ket_Noi` bằng truy vấn gộp theo (`u`, `v`) và lấy `MIN(travel_time)`. Chỉ thêm cạnh khi cả hai đầu đều có trong `dict` nút.
3. Lưu cạnh dạng danh sách kề: `adj[u]` là danh sách bộ ba (nút đến, trọng số, có phải đổi tuyến không).
4. Lập `by_name`: tên ga → danh sách `node_id` của ga đó.
5. Đọc bảng `KhungGioCaoDiem`, đổi `HH:MM` thành giây tính từ nửa đêm.

**Vì sao dùng đúng hai câu truy vấn này:** tuần 4 lõi C++ dùng lại y hệt, nên hai bản luôn nhìn thấy cùng một đồ thị. Điều kiện `status = 1` là cách trang quản trị đóng ga ở tuần 7.

**Kết quả đúng:** 192 nút, 99 ga, 378 cạnh chạy tàu, 2 khung giờ.

### 3.2 Dijkstra nhiều nguồn (Hải Nam)

Viết `search(start_name, goal_name, ...)`, ở bước này chưa có heuristic và chưa có giờ cao điểm.

1. Khởi tạo: mọi nút của ga đi đều có `g = 0` và đều được đẩy vào hàng đợi.
2. Vòng lặp: lấy phần tử nhỏ nhất ra khỏi hàng đợi.
3. Kiểm tra đích ngay lúc lấy ra: tên ga của nút bằng `goal_name` thì trả kết quả.
4. Nếu chi phí của phần tử lớn hơn `g` hiện tại của nút thì bỏ qua (phần tử lỗi thời).
5. Nới lỏng từng cạnh đi ra: nếu đi qua nút hiện tại rẻ hơn thì cập nhật `g` và đẩy vào hàng đợi.
6. Hàng đợi rỗng mà chưa tới đích thì trả về "không tìm thấy".

**Thử:**

| Phép thử | Kết quả đúng |
|---|---|
| Altrincham đi Bury | 3.720 giây (62 phút) |
| Manchester Airport đi Rochdale Town Centre | Không tìm thấy, vì chưa có cạnh đổi tuyến và hai ga không nằm trên cùng tuyến |
| Altrincham đi Bury, chỉ khởi tạo nút Purple Line của Altrincham | Không tìm thấy, vì Purple Line không tới Bury |

### 3.3 Cạnh đổi tuyến và truy vết (Hải Nam)

1. Sau khi nạp đồ thị, với mỗi ga có từ hai nút, thêm cạnh giữa mọi cặp nút khác nhau của ga đó: trọng số 300 giây, đánh dấu là đổi tuyến.
2. Đếm tổng số cạnh đổi tuyến.
3. Trong `search`, ghi số lần đổi tuyến của đường đi tốt nhất tới mỗi nút.
4. Ghi `came_from` khi nới lỏng. Khi tới đích, lần ngược `came_from` để lấy danh sách nút. Nút xuất phát là nút không có `came_from`.
5. Thí nghiệm: tạm sửa bước khởi tạo để chỉ xuất phát từ một nút duy nhất của Altrincham, lần lượt thử từng nút, và so kết quả.

**Kết quả đúng** (chưa có giờ cao điểm):

| Hành trình | Thời gian | Đổi tuyến |
|---|---|---|
| Tổng số cạnh đổi tuyến | 390 | |
| Altrincham đi Bury | 62 phút | Không |
| St Peter's Square đi Manchester Airport | 51,5 phút | Không |
| The Trafford Centre đi East Didsbury | 42 phút | Tại Trafford Bar |
| Manchester Airport đi Rochdale Town Centre | 117 phút | Tại Victoria |
| Altrincham đi Bury, chỉ xuất phát từ nút Purple Line | 67 phút | Một lần, thừa |

Dòng cuối cho thấy vì sao phải xuất phát từ mọi nút của ga đi: hành khách được lên bất kỳ tuyến nào tại ga đi mà không mất 300 giây.

**Về số ga trên lộ trình:** Altrincham đi Bury có hai lộ trình cùng 62 phút: qua Exchange Square (24 ga) và qua Market Street, Shudehill (25 ga). Thuật toán trả về lộ trình nào là tùy thứ tự lấy phần tử bằng nhau ra khỏi hàng đợi. Cả hai đều đúng, nên chỉ so tổng thời gian, đừng so danh sách ga.

### 3.4 Chi phí phụ thuộc thời gian (Hải Nam)

1. Thêm tham số `start_sec` (giờ khởi hành, tính bằng giây) cho `search`.
2. Viết `peak_at(second_of_day)`: trả về (hệ số, giây chờ thêm) của khung giờ đầu tiên chứa thời điểm đó; ngoài mọi khung giờ thì trả về (1,0; 0).
3. Khi mở rộng nút `u`: thời điểm trong ngày bằng `(start_sec + phần nguyên của g[u]) % 86400`.
4. Chi phí của cạnh bằng trọng số nhân hệ số; nếu là cạnh đổi tuyến thì cộng thêm giây chờ.

**Kết quả đúng:**

| Hành trình | 12:00 | 08:00 |
|---|---|---|
| Altrincham đi Bury | 62 phút | 93 phút |
| Manchester Airport đi Rochdale Town Centre | 117 phút, đổi tại Victoria | 150,5 phút, đổi tại St Peter's Square |

**Điểm cần hiểu:** hệ số được tra theo thời điểm hành khách tới nút, không theo giờ khởi hành. Hành trình thứ hai kéo dài qua 09:30, nên phần sau không còn bị nhân 1,5. Vì thế tổng thời gian chỉ tăng 29% chứ không phải 50%.

### 3.5 Heuristic và A* (Hải Nam)

1. Viết `haversine_time(u, v)`: khoảng cách Haversine giữa hai nút (mét) chia cho `MAX_SPEED`. Mặc định 20 m/s; cho phép đổi bằng tham số dòng lệnh.
2. Thêm tham số `use_heuristic` cho `search`. Khi bật, `h(n)` là `haversine_time` từ `n` tới một nút của ga đích; khi tắt, `h(n) = 0`.
3. Phần tử hàng đợi là bộ ba (`g + h`, `g`, nút). Bước kiểm tra phần tử lỗi thời vẫn so theo `g`.
4. Thêm biến đếm số nút được mở rộng (số lần đi qua bước 4 của 3.2 mà không bị bỏ qua).
5. Trả về (tổng thời gian, số lần đổi tuyến, số nút mở rộng).

**Kết quả đúng:** với mọi phép thử ở 3.3 và 3.4, bật hay tắt heuristic đều cho cùng tổng thời gian; số nút mở rộng khi bật nhỏ hơn hoặc bằng khi tắt.

### 3.6 Vận tốc đường thẳng lớn nhất (Kiệt)

Heuristic giả định không chuyến tàu nào nhanh hơn `MAX_SPEED` theo đường thẳng. Kiệt kiểm tra giả định đó trên dữ liệu.

1. Với mỗi cạnh trong 378 cạnh chạy tàu, tính khoảng cách Haversine giữa hai ga đầu cạnh, chia cho `travel_time`.
2. Tìm giá trị lớn nhất và cạnh đạt giá trị đó.
3. Lập bảng 5 cạnh nhanh nhất để đưa vào báo cáo.

**Kết quả đúng:** lớn nhất là 14,12 m/s, trên cạnh Whitefield đi Radcliffe.

**Điều rút ra:** `MAX_SPEED` phải lớn hơn hoặc bằng 14,12 m/s. Giá trị 20 và 15 đều hợp lệ; giá trị càng gần 14,12 thì heuristic càng mạnh.

### 3.7 Chạy toàn bộ cặp ga (Hải Nam)

1. Lập danh sách mọi cặp (ga đi, ga đến) có thứ tự, ga đi khác ga đến.
2. Với hai giờ khởi hành 12:00 và 08:00, chạy `search` cho mọi cặp, một lần bật và một lần tắt heuristic.
3. In các đại lượng sau cho mỗi giờ khởi hành:
   - Số cặp mà A* và Dijkstra cho cùng tổng thời gian (sai khác dưới 10⁻⁶).
   - Số nút mở rộng trung bình của A* và của Dijkstra, cùng phần trăm giảm.
   - Thời gian hành trình trung bình, trung vị, dài nhất, và cặp ga dài nhất.
   - Phân bố số lần đổi tuyến.
4. Chạy lại với `MAX_SPEED` bằng 15.

**Kết quả đúng** (20 m/s, khởi hành 12:00):

```text
Đồ thị: 192 nút, 99 ga, 378 cạnh chạy tàu, 390 cạnh đổi tuyến | 9702 cặp ga
A* trùng chi phí với Dijkstra: 9702/9702 cặp
Số nút mở rộng trung bình: A* = 91.6, Dijkstra = 108.4 (A* giảm 15.6%)
Thời gian hành trình (phút): trung bình 41.1, trung vị 40.0, dài nhất 117.0
Hành trình dài nhất: Manchester Airport -> Rochdale Town Centre
Số lần đổi tuyến: {0: 3876, 1: 5826}
```

| Giờ khởi hành | `MAX_SPEED` | A* | Dijkstra | A* giảm |
|---|---|---|---|---|
| 12:00 | 20 m/s | 91,6 | 108,4 | 15,6% |
| 08:00 | 20 m/s | 95,3 | 107,2 | 11,1% |
| 12:00 | 15 m/s | 85,7 | 108,4 | 20,9% |
| 08:00 | 15 m/s | 91,2 | 107,2 | 14,9% |

### 3.8 Báo cáo giữa kỳ (Hải Nam tổng hợp, hạn nộp 24/10)

Sửa mục "3. Tiến độ giữa kỳ (W8)" trong `report/Project report N12.ipynb`.

| Việc | Người làm |
|---|---|
| Chạy lại hai notebook trên bản feed của nhóm; cập nhật bảng "Kết quả xử lý dữ liệu" và năm "Vấn đề gặp phải" | Kiệt |
| Sửa cột "Trạng thái" trong bảng "Chương trình" theo đúng thực tế ngày 23/10 | Hải Nam |
| Sửa dòng "Lõi tìm đường": bản Python (`backend/evaluate.py`) hoàn thành; bản C++ viết trong tuần 26/10–01/11 | Hải Nam |
| Sửa đoạn mô tả lệnh `metro.exe` thành mô tả cách chạy `evaluate.py` | Hải Nam |
| Cập nhật bảng "Kết quả chạy thử lõi tìm đường" bằng kết quả của `evaluate.py` | Hải Nam |
| Sửa dòng "API" và "Giao diện" theo thực tế (mới có mã chuẩn bị, chưa tích hợp) | Hà Nam |
| Đọc soát toàn bộ mục 3, xóa các ghi chú nháp trong ô | Hà Nam |
| Nộp | Hải Nam, thứ Bảy 24/10 |

Không ghi "Hoàn thành" cho phần chưa viết.

### 3.9 Chuẩn bị: biên dịch SQLite với g++ (Kiệt)

Mục đích là bảo đảm tuần 4 không mất thời gian vì công cụ.

1. Đặt `sqlite3.c` và `sqlite3.h` vào `backend/`. Đây là thư viện SQLite bản gộp một file, lấy từ thư mục `backend` của bản Madrid hoặc từ trang tải của sqlite.org. Hai file này không tự viết.
2. Biên dịch thư viện một lần: `gcc -c sqlite3.c -o sqlite3.o`. Lệnh này chạy khá lâu.
3. Viết `backend/thu_nghiem/kiem_tra_sqlite.cpp`: `#include "../sqlite3.h"`, in ra kết quả của `sqlite3_libversion()`.
4. Biên dịch và chạy: `g++ thu_nghiem/kiem_tra_sqlite.cpp sqlite3.o -o thu_nghiem/kiem_tra.exe`.

**Kết quả đúng:** chương trình in ra số phiên bản SQLite.

### 3.10 Chuẩn bị: bcrypt và JWT (Hà Nam)

1. Trong `backend/thu_nghiem/`, chạy `npm install bcrypt jsonwebtoken`.
2. Viết `thu_auth.js`:
   - Băm một mật khẩu bằng `bcrypt.hash` với 10 vòng, in chuỗi băm.
   - Băm lại cùng mật khẩu đó, so hai chuỗi băm.
   - Dùng `bcrypt.compare` kiểm tra mật khẩu đúng và mật khẩu sai.
   - Tạo token bằng `jwt.sign` chứa `{ id, role }`, hạn 1 ngày.
   - Kiểm tra token bằng `jwt.verify` với khóa đúng và với khóa sai.
3. Dán token vào một công cụ giải mã JWT để xem phần payload.

**Kết quả đúng:** hai chuỗi băm khác nhau dù cùng mật khẩu; `compare` trả `true` rồi `false`; `verify` với khóa sai ném lỗi; payload đọc được mà không cần khóa.

## Nghiệm thu cuối tuần

- [ ] `python evaluate.py` chạy không lỗi và in đủ các đại lượng ở 3.7.
- [ ] A* trùng Dijkstra ở 9.702/9.702 cặp, cả 12:00 lẫn 08:00.
- [ ] Bốn dòng của bảng số nút mở rộng khớp (hoặc đã ghi số mới nếu feed của nhóm khác).
- [ ] Kiệt: vận tốc đường thẳng lớn nhất 14,12 m/s.
- [ ] Báo cáo giữa kỳ đã nộp ngày 24/10, cột "Trạng thái" đúng thực tế.
- [ ] Kiệt: `kiem_tra.exe` in ra phiên bản SQLite.
- [ ] Hà Nam: `thu_auth.js` chạy đủ năm phép thử.

## Câu hỏi cả nhóm phải trả lời được

1. Vì sao kiểm tra đích lúc lấy nút ra khỏi hàng đợi, không phải lúc đẩy vào?
2. Vì sao phải bỏ qua phần tử lỗi thời trong hàng đợi?
3. Vì sao mọi nút của ga đi đều có `g = 0`?
4. Heuristic "chấp nhận được" nghĩa là gì? Vì sao `MAX_SPEED` không được nhỏ hơn 14,12 m/s?
5. Vì sao A* chỉ giảm khoảng 16% số nút mở rộng so với Dijkstra?
6. Vì sao cạnh đổi tuyến không làm heuristic sai?

<details>
<summary>Gợi ý đáp án</summary>

1. Lúc đẩy vào, chi phí tới nút đó chưa chắc là nhỏ nhất; có thể còn đường khác rẻ hơn chưa xét. Lúc lấy ra thì chi phí đã chốt.
2. `heapq` không sửa được ưu tiên của phần tử đã có, nên mỗi lần tìm được đường rẻ hơn ta đẩy thêm một phần tử mới. Phần tử cũ vẫn nằm trong hàng đợi và phải bị bỏ qua.
3. Hành khách được lên bất kỳ tuyến nào tại ga đi. Nếu chỉ xuất phát từ một nút, hành khách phải trả 300 giây để "đổi tuyến" ngay tại ga đi.
4. `h(n)` không bao giờ lớn hơn chi phí thật từ `n` tới đích. Nếu `MAX_SPEED` nhỏ hơn vận tốc thật của một cạnh, `h` sẽ ước lượng cao hơn chi phí thật qua cạnh đó.
5. 20 m/s cao hơn nhiều so với vận tốc thật nên `h` yếu; đồ thị nhỏ và gồm các nhánh hướng tâm nên phần lớn nút nằm gần đường tối ưu; giờ cao điểm làm chi phí thật tăng còn `h` giữ nguyên.
6. Hai nút của cùng một ga có cùng tọa độ, nên `h` bằng nhau. Cạnh đổi tuyến có chi phí dương, vẫn thỏa `h(u) − h(v) ≤ chi phí`.

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| Altrincham đi Bury ra 67 phút | Chỉ khởi tạo một nút của ga đi |
| A* cho kết quả khác Dijkstra | `MAX_SPEED` nhỏ hơn 14,12; hoặc bỏ qua phần tử lỗi thời bằng cách so `f` thay vì `g` |
| Kết quả 08:00 bằng đúng 1,5 lần kết quả 12:00 ở mọi cặp | Tra hệ số theo giờ khởi hành thay vì theo thời điểm tới nút |
| `KeyError` khi tra `g` | Chưa khởi tạo `g` cho nút chưa thăm; dùng giá trị mặc định rất lớn |
| Chạy toàn bộ cặp ga mất nhiều phút | Đang nạp lại đồ thị trong mỗi lần gọi `search`; chỉ nạp một lần |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 10 (toàn văn `evaluate.py`) và Bước 9 (bảng hành trình).
- Mục 4 của `report/Project report N12.ipynb`: công thức chi phí, lập luận về tính chấp nhận được, các bảng số liệu.
