# Tuần 6 (09/11–15/11): Giao diện người dùng

| | |
|---|---|
| Phụ trách chính | **Hà Nam** |
| Hỗ trợ và review | Kiệt (viết `RouteList.jsx`, kiểm thử trên web), Hải Nam (review phần đọc JSON, chốt số liệu đánh giá) |
| Sản phẩm cuối tuần | Trang tìm đường chạy ở `http://localhost:5173`: nhập ga, xem lộ trình từng ga và đường đi trên bản đồ |
| Cần có trước | `server.js` chạy được (tuần 5) |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

## Cấu trúc tuần này

```text
App
└── UserDashboard     giữ state: danh sách ga, kết quả tìm đường, trạng thái đang tải
    ├── Sidebar       form nhập; gọi onSearch(nút đi, nút đến, chế độ, giờ)
    │   └── RouteList hiển thị từng ga của lộ trình
    └── MapView       vẽ mọi ga và lộ trình trên bản đồ

services/api.js       nơi duy nhất gọi fetch tới /api/...
```

Đường đi của dữ liệu khi người dùng bấm "Tìm đường":

```text
Sidebar.handleSubmit → UserDashboard.handleSearch → api.getRoute
  → GET /api/routing (server.js) → metro.exe → JSON
  → state của UserDashboard → RouteList và MapView vẽ lại
```

Tuần này chưa có đăng nhập. `App.jsx` chỉ hiển thị `UserDashboard`; router thêm ở tuần 7.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 6.1 | Tạo khung Vite và React, Tailwind, proxy `/api` | Hà Nam | Kiệt | T2 09/11 |
| 6.2 | `services/api.js` | Hà Nam | Hải Nam | T3 10/11 |
| 6.3 | `UserDashboard.jsx` | Hà Nam | Hải Nam | T4 11/11 |
| 6.4 | `components/Sidebar.jsx` | Hà Nam | Kiệt | T5 12/11 |
| 6.5 | `components/RouteList.jsx` | Kiệt | Hải Nam | T6 13/11 |
| 6.6 | `components/MapView.jsx` | Hà Nam | Kiệt | T7 14/11 |
| 6.7 | Kiểm thử 7 hành trình trên web | Kiệt | Hà Nam | CN 15/11 |
| 6.8 | Chốt số liệu đánh giá và giá trị `MAX_SPEED` | Hải Nam | Kiệt | CN 15/11 |
| 6.9 | Buổi giảng lại tuần 6 | Hà Nam trình bày; Kiệt trình bày `RouteList` | Cả nhóm | CN 15/11 |

## Hướng dẫn chi tiết

Thứ tự khởi động mỗi lần làm việc: dịch vụ MySQL, rồi `node server.js` trong `backend/`, rồi `npm run dev` trong `frontend/`.

### 6.1 Khung dự án (Hà Nam)

1. Ở thư mục gốc của project, chạy `npm create vite@latest frontend`, chọn React và JavaScript. Phần khung do Vite sinh ra không cần tự viết.
2. Trong `frontend/`: `npm install`, rồi cài `leaflet`, `react-leaflet`, `react-router-dom`, `lucide-react`.
3. Cài Tailwind theo cách của bản Madrid: gói `tailwindcss`, `@tailwindcss/postcss`, `postcss`; file `postcss.config.js` khai báo plugin `@tailwindcss/postcss`; dòng `@import "tailwindcss";` ở đầu `src/index.css`.
4. Trong `vite.config.js`, thêm proxy: mọi đường dẫn bắt đầu bằng `/api` chuyển tới `http://localhost:3000`.
5. Đổi `<title>` trong `index.html` thành "Manchester Metrolink Routing".
6. Xóa nội dung mẫu trong `App.jsx` và `App.css`.

**Vì sao dùng proxy:** frontend chỉ gọi đường dẫn tương đối như `/api/stations`. Trình duyệt thấy mọi yêu cầu đều tới `localhost:5173`, nên không vướng CORS, và cổng của backend chỉ ghi ở một chỗ. Bản Madrid viết thẳng `http://localhost:3000` ở nhiều file; đổi cổng là phải sửa từng file.

**Kiểm tra:** `npm run dev`, mở `http://localhost:5173/api/stations`. Phải thấy mảng JSON 192 phần tử. Thêm một thẻ có class Tailwind (ví dụ chữ đỏ, in đậm) vào `App.jsx` và thấy nó có hiệu lực.

### 6.2 `services/api.js` (Hà Nam)

1. `getStations()`: gọi `/api/stations`, đổi tên trường về tên dùng trong frontend: `node_id` thành `id`, `stop_name` thành `name`, `stop_lat` thành `lat`, `stop_lon` thành `lon`, giữ `status`. Lỗi thì trả mảng rỗng.
2. `getRoute(start, end, mode, time)`: gọi `/api/routing` với bốn tham số trên query. Mã hóa `time` bằng `encodeURIComponent`. Lỗi mạng thì trả `{ status: 'error', message }`.

**Điểm cần hiểu:** đây là lớp "phiên dịch" giữa tên cột của CSDL và tên biến của giao diện. Mọi component khác không gọi `fetch` trực tiếp, nên đổi API chỉ phải sửa file này.

**Kiểm tra:** gọi tạm hai hàm trong `App.jsx` và `console.log` kết quả. `getStations()` trả 192 phần tử có `id`, `name`, `lat`, `lon`, `status`; `getRoute(3, 27, 1, '12:00')` trả đối tượng có `total_time` bằng 3720.

### 6.3 `UserDashboard.jsx` (Hà Nam)

1. Khai báo state: danh sách ga, kết quả tìm đường (ban đầu `null`), cờ đang tải.
2. Dùng `useEffect` với mảng phụ thuộc rỗng để gọi `getStations()` một lần khi trang mở.
3. Viết `handleSearch(startId, endId, mode, time)`: bật cờ đang tải; gọi `getRoute`; nếu kết quả có `path` thì lưu vào state, không thì báo cho người dùng và xóa kết quả cũ; cuối cùng tắt cờ đang tải, kể cả khi có lỗi.
4. Bố cục: hai cột cao bằng màn hình; cột trái là `Sidebar`, cột phải là `MapView` chiếm phần còn lại.
5. Truyền xuống `Sidebar`: danh sách ga, `handleSearch`, cờ đang tải, kết quả. Truyền xuống `MapView`: danh sách ga, kết quả.

**Hai chỗ bản Madrid làm thừa, đừng chép:**

- Giữ ba state `pathData`, `totalTime`, `resultData` cho cùng một kết quả. Một state là đủ.
- `Sidebar` tự gọi `getStations()` thêm một lần nữa. Truyền danh sách ga từ `UserDashboard` xuống là đủ.

### 6.4 `Sidebar.jsx` (Hà Nam)

1. State: tên ga đi, tên ga đến, chế độ (mặc định `'1'`), giờ khởi hành.
2. Khi component mở, đặt giờ khởi hành bằng giờ hiện tại của máy, dạng `HH:MM`.
3. Hai ô nhập tên ga dùng chung một `<datalist>`. Danh sách gợi ý chỉ gồm các tên không trùng: 192 nút nhưng chỉ có 99 tên.
4. Một ô `type="time"` cho giờ khởi hành và một `<select>` cho hai chế độ "Nhanh nhất", "Ít chuyển".
5. Khi gửi form: chặn hành vi mặc định; với mỗi tên ga, tìm nút đầu tiên có đúng tên đó **và đang mở** (`status` bằng 1); không tìm được thì báo người dùng chọn lại; tìm được thì gọi `onSearch`.
6. Nút gửi bị vô hiệu và đổi chữ khi đang tải.
7. Đặt `RouteList` ở cuối cột.

**Vì sao gửi nút đầu tiên trùng tên là đủ:** lõi C++ coi mọi nút cùng tên ga với nút đi là điểm xuất phát, và kiểm tra đích theo tên ga.

**Vì sao phải lọc nút đang mở:** `metro.exe` chỉ nạp nút có `status = 1`. Nếu nút đầu tiên của ga đang bị đóng mà các nút khác vẫn mở, bản Madrid vẫn gửi nút bị đóng và nhận về "Not found" dù ga vẫn đi được.

### 6.5 `RouteList.jsx` (Kiệt)

Nhận `result` qua props. Đọc lại hợp đồng JSON ở đầu [tuan4.md](tuan4.md) trước khi viết.

1. Không có `result` hoặc không có `path` thì không hiển thị gì.
2. Hiển thị tổng thời gian bằng phút: `total_time` chia 60, làm tròn.
3. Duyệt `path`, mỗi phần tử một dòng gồm tên ga và, trừ ga đầu, số phút tích lũy (`arrival_time` chia 60).
4. Nếu phần tử **kế tiếp** có `is_transfer` là `true`: hiển thị dưới tên ga dòng "Chờ đổi tuyến (N phút)", với N lấy từ `step_time` của phần tử kế tiếp.
5. Ga đầu và ga cuối in đậm hơn các ga giữa.
6. Dùng chữ "Chờ đổi tuyến", không dùng "Đi bộ" như bản Madrid. Ở Metrolink các tuyến dùng chung sân ga; 300 giây là thời gian chờ chuyến sau.

**Kiểm tra bằng dữ liệu thật:** Manchester Airport đi Rochdale Town Centre lúc 12:00 phải hiện 117 phút, và ga Victoria xuất hiện hai dòng liên tiếp với ghi chú "Chờ đổi tuyến (5 phút)" ở dòng đầu.

### 6.6 `MapView.jsx` (Hà Nam)

1. `import 'leaflet/dist/leaflet.css'`. Thẻ bao bản đồ phải có chiều cao rõ ràng, nếu không bản đồ không hiện.
2. `MapContainer` với tâm `[53.479, -2.236]`, zoom 11; `TileLayer` của OpenStreetMap kèm dòng ghi nguồn.
3. Vẽ mọi ga bằng `CircleMarker` nhỏ màu xám, có `Tooltip` là tên ga. Chỉ vẽ một chấm cho mỗi tên ga (99 chấm, không phải 192). Bỏ chấm xám của những ga đang nằm trên lộ trình.
4. Khi có kết quả: vẽ `Polyline` qua tọa độ các phần tử của `path`, kèm nhãn tổng số phút.
5. Vẽ `Marker` cho từng phần tử của `path` với bốn loại biểu tượng: ga xuất phát, ga kết thúc, ga trung gian, điểm đổi tuyến. Điểm đổi tuyến là phần tử có tên trùng với phần tử kế tiếp.
6. Viết một component con dùng hook `useMap()` và `fitBounds` để thu phóng bản đồ vừa khít lộ trình mỗi khi lộ trình đổi.
7. Thêm khung chú thích ở góc bản đồ.
8. Ghi dòng "Contains Transport for Greater Manchester data" trong khung chú thích. Giấy phép dữ liệu yêu cầu dòng này.

**Hai điểm cần hiểu:**

- Các thuộc tính `center` và `zoom` của `MapContainer` chỉ có tác dụng lúc tạo bản đồ. Muốn di chuyển bản đồ sau đó phải dùng `useMap()` trong một component con (bước 6).
- Bản Madrid lọc chấm xám theo `s.id` của từng phần tử trong `path`, nhưng JSON của `metro.exe` không có trường `id`, nên bộ lọc đó không bao giờ khớp. Hãy lọc theo tên ga.

**Lỗi biểu tượng mặc định:** với Vite, biểu tượng `Marker` mặc định của Leaflet không tự tải được ảnh. Phải khai báo lại đường dẫn ảnh, hoặc dùng `L.divIcon` tự vẽ.

### 6.7 Kiểm thử trên web (Kiệt)

Thử từng dòng, ghi kết quả thật. Hai khung giờ cao điểm phải đang ở giá trị mặc định.

| Ga đi | Ga đến | Chế độ | Giờ | Kết quả đúng | Kết quả thật |
|---|---|---|---|---|---|
| Altrincham | Bury | Nhanh nhất | 12:00 | 62 phút, không đổi tuyến | |
| Manchester Airport | Rochdale Town Centre | Nhanh nhất | 12:00 | 117 phút, đổi tuyến tại Victoria | |
| St Peter's Square | Manchester Airport | Nhanh nhất | 12:00 | 51,5 phút, không đổi tuyến | |
| The Trafford Centre | East Didsbury | Nhanh nhất | 12:00 | 42 phút, đổi tuyến tại Trafford Bar | |
| Altrincham | Bury | Nhanh nhất | 08:00 | 93 phút | |
| Manchester Airport | Rochdale Town Centre | Nhanh nhất | 08:00 | 150,5 phút, đổi tuyến tại St Peter's Square | |
| Manchester Airport | Rochdale Town Centre | Ít chuyển | 12:00 | 117 phút, 1 lần đổi tuyến | |

Giao diện làm tròn về phút nguyên, nên 51,5 hiện thành 52 và 150,5 hiện thành 151. Muốn so chính xác thì xem `total_time` trong tab Network của trình duyệt.

Thử thêm:

- Nhập một tên không có trong danh sách: phải có thông báo, không gửi yêu cầu.
- Tắt `server.js` rồi bấm tìm: phải có thông báo lỗi, nút không bị kẹt ở trạng thái "Đang tính".
- Đóng riêng nút 176 (nút đầu tiên của Victoria) bằng API của tuần 5, rồi tìm Victoria đi Bury: vẫn phải ra đường. Mở lại nút 176 sau khi thử.

### 6.8 Chốt số liệu đánh giá (Hải Nam)

1. Chạy `python evaluate.py` và `python evaluate.py 15` trên bản feed của nhóm.
2. Lập bốn bảng cho mục 4 của báo cáo: tính tối ưu, số nút mở rộng, thời gian chạy (số đo ở 5.10), chất lượng lộ trình.
3. Quyết định `MAX_SPEED` cuối cùng là 20 hay 15 m/s. Nếu đổi, phải sửa đồng thời `main.cpp`, `evaluate.py` và báo cáo, rồi chạy lại `compare_cpp.py`.
4. Review 6.2 và 6.5: frontend đọc đúng nghĩa của `step_time`, `is_transfer`, `arrival_time` chưa.

## Nghiệm thu cuối tuần

- [ ] `npm run dev` chạy; `npm run build` không báo lỗi.
- [ ] Bản đồ hiện 99 chấm ga quanh Manchester.
- [ ] Bảy hành trình ở 6.7 cho đúng kết quả.
- [ ] Lộ trình có đổi tuyến hiện ghi chú "Chờ đổi tuyến" ở `RouteList` và biểu tượng riêng trên bản đồ.
- [ ] Bản đồ tự thu phóng theo lộ trình.
- [ ] Không file nào ngoài `api.js` gọi `fetch`; không chỗ nào viết cứng `localhost:3000` ngoài `vite.config.js`.
- [ ] Dòng ghi nguồn dữ liệu TfGM có trên giao diện.

## Câu hỏi cả nhóm phải trả lời được

1. Từ lúc bấm "Tìm đường" đến lúc bản đồ vẽ lại, dữ liệu đi qua những hàm và file nào?
2. Vì sao kết quả tìm đường đặt ở `UserDashboard`, không đặt trong `Sidebar` hay `MapView`?
3. Vì sao gọi `/api/...` qua proxy thì không cần CORS, còn gọi thẳng `http://localhost:3000` thì cần?
4. Frontend chỉ gửi `node_id` của một nút. Vì sao kết quả vẫn đúng cho mọi tuyến của ga đó? Khi nào thì sai?
5. Trong JSON, một lần đổi tuyến trông như thế nào? `RouteList` và `MapView` mỗi bên nhận ra nó bằng cách nào?
6. Vì sao đổi thuộc tính `center` của `MapContainer` không làm bản đồ di chuyển?

<details>
<summary>Gợi ý đáp án</summary>

1. Xem sơ đồ "Đường đi của dữ liệu" ở đầu file.
2. Cả `Sidebar` (qua `RouteList`) lẫn `MapView` đều cần kết quả. State đặt ở component cha chung gần nhất rồi truyền xuống hai nhánh.
3. CORS chỉ áp dụng khi trang ở nguồn này gọi sang nguồn khác (khác cổng cũng là khác nguồn). Qua proxy, trình duyệt chỉ gọi tới chính nguồn của trang.
4. Lõi C++ khởi tạo mọi nút cùng tên ga với nút đi, và dừng ở bất kỳ nút nào cùng tên ga đích. Sai khi nút được gửi đang bị đóng: lõi không tìm thấy nút đó trong đồ thị và trả "Not found".
5. Hai phần tử liên tiếp cùng tên ga, phần tử sau có `is_transfer` là `true` và `step_time` là thời gian chờ. `RouteList` xem `is_transfer` của phần tử kế tiếp; `MapView` so tên với phần tử kế tiếp.
6. `MapContainer` chỉ đọc các thuộc tính đó một lần khi tạo bản đồ. Sau đó phải gọi phương thức của đối tượng bản đồ lấy từ `useMap()`.

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| Bản đồ là một vùng trống | Thiếu `leaflet.css`, hoặc thẻ bao không có chiều cao |
| Ô gợi ý lặp tên ga nhiều lần | Chưa loại tên trùng trước khi tạo `<option>` |
| `/api/stations` trả 404 từ cổng 5173 | Chưa cấu hình proxy, hoặc chưa khởi động lại `npm run dev` sau khi sửa `vite.config.js` |
| Mọi lần tìm đều "không tìm thấy" | `server.js` chưa chạy, hoặc chạy sai thư mục |
| Class Tailwind không có tác dụng | Thiếu `postcss.config.js` hoặc thiếu dòng `@import "tailwindcss";` |
| Cảnh báo "Each child in a list should have a unique key" | Thiếu `key` khi duyệt mảng; dùng chỉ số cho `path`, vì tên ga lặp lại ở điểm đổi tuyến |
| Biểu tượng ga là ô ảnh vỡ | Chưa khai báo lại đường dẫn ảnh của biểu tượng Leaflet |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 8.1 đến 8.5 và Bước 9.
- Bản Madrid: `Project-AI-2025.2-main/frontend/src/` gồm `services/api.js`, `UserDashboard.jsx`, `components/Sidebar.jsx`, `components/RouteList.jsx`, `components/MapView.jsx`, và `vite.config.js`, `postcss.config.js`.
