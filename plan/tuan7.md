# Tuần 7 (16/11–22/11): Đăng nhập và trang quản trị

| | |
|---|---|
| Phụ trách chính | **Hà Nam** (router, đăng nhập, khung trang quản trị) |
| Cùng viết | **Kiệt** (bảng ga), **Hải Nam** (khung giờ cao điểm) |
| Sản phẩm cuối tuần | Đăng nhập phân quyền; trang quản trị đóng mở ga và sửa khung giờ cao điểm, có hiệu lực ngay ở lần tìm đường kế tiếp |
| Cần có trước | API có kiểm tra token (tuần 5), trang tìm đường (tuần 6) |

Quy ước chung và tên gọi thành viên: xem [tuan1.md](tuan1.md).

Đây là tuần nhiều code nhất (khoảng 800 dòng ở bản Madrid), nên chia ba. Viết phần xử lý trước, phần trình bày sau. Hết giờ thì bỏ hiệu ứng và màu sắc, không bỏ chức năng.

## Cấu trúc tuần này

```text
App (BrowserRouter)
├── /auth   → AuthPage
├── /       → chuyển hướng sang /user
├── /user   → ProtectedRoute → UserDashboard (tuần 6)
└── /admin  → ProtectedRoute (yêu cầu admin) → AdminDashboard
                                                ├── Sidebar   đổi tab, đăng xuất
                                                ├── Header
                                                ├── StationTable → ToggleSwitch, Toast
                                                └── PeakHoursAdmin
```

Vòng đời của token: `server.js` tạo ra khi đăng nhập; `AuthPage` lưu vào `localStorage`; `authFetch` gửi kèm trong header `Authorization`; middleware của `server.js` kiểm tra.

## Bảng phân công

| Mã | Nhiệm vụ | Phụ trách | Review | Hạn |
|---|---|---|---|---|
| 7.1 | `App.jsx`: router và `ProtectedRoute` | Hà Nam | Hải Nam | T2 16/11 |
| 7.2 | `components/AuthPage.jsx` | Hà Nam | Kiệt | T4 18/11 |
| 7.3 | `authFetch` trong `api.js`; nút đăng xuất | Hà Nam | Hải Nam | T4 18/11 |
| 7.4 | Khung trang quản trị: `AdminDashboard`, `Sidebar`, `Header` | Hà Nam | Kiệt | T5 19/11 |
| 7.5 | `Admin/StationTable.jsx` và `Admin/ToggleSwitch.jsx` | Kiệt | Hà Nam | T7 21/11 |
| 7.6 | `Admin/PeakHoursAdmin.jsx` | Hải Nam | Hà Nam | T7 21/11 |
| 7.7 | `Admin/Toast.jsx`, làm sau cùng | Kiệt | Hà Nam | T7 21/11 |
| 7.8 | Tạo tài khoản admin, thử phân quyền | Hà Nam | Hải Nam | CN 22/11 |
| 7.9 | Thử tác động của trang quản trị lên kết quả tìm đường | Hải Nam và Kiệt | Hà Nam | CN 22/11 |
| 7.10 | Nâng cấp tùy chọn | Ai còn thời gian | | Không bắt buộc |
| 7.11 | Buổi giảng lại tuần 7 | Mỗi người trình bày phần mình | Cả nhóm | CN 22/11 |

Kiệt và Hải Nam bắt đầu 7.5 và 7.6 từ thứ Năm 19/11, sau khi 7.3 và 7.4 đã gộp vào `main`. Trước đó hai bạn đọc mã nguồn tuần 6 và thử API quản trị bằng `thu_api.ps1`.

## Hướng dẫn chi tiết

### 7.1 Router và `ProtectedRoute` (Hà Nam)

1. Bọc ứng dụng trong `BrowserRouter`, khai báo bốn route như sơ đồ trên.
2. Viết `ProtectedRoute` nhận `children` và cờ `requireAdmin`:
   - Không có `token` trong `localStorage` thì chuyển hướng sang `/auth`.
   - `requireAdmin` bật mà `role` khác `admin` thì chuyển hướng sang `/user`.
   - Còn lại thì hiển thị `children`.

**Điểm cần hiểu:** `ProtectedRoute` chỉ quyết định hiển thị màn hình nào. Nó không bảo vệ dữ liệu; việc đó thuộc về middleware ở nhiệm vụ 5.7.

**Kiểm tra:** xóa `localStorage`, mở `/user` và `/admin`: cả hai đều chuyển về `/auth`.

### 7.2 `AuthPage.jsx` (Hà Nam)

1. State: đang ở chế độ đăng nhập hay đăng ký, `username`, `password`, cờ đang tải, thông báo lỗi.
2. Một form hai ô nhập và một nút gửi; một liên kết chuyển qua lại giữa hai chế độ, khi chuyển thì xóa ô nhập và lỗi.
3. Khi gửi: gọi `POST /api/login` hoặc `POST /api/register` với body JSON.
4. Phản hồi không thành công (`response.ok` là `false`): hiển thị `message` do máy chủ trả về.
5. Đăng nhập thành công: lưu `token`, `role`, `username` vào `localStorage`; `role` là `admin` thì chuyển sang `/admin`, không thì sang `/user`.
6. Đăng ký thành công: báo cho người dùng và chuyển về chế độ đăng nhập.

**Kiểm tra:** đăng ký một tài khoản mới, đăng nhập, được đưa tới `/user`. Nhập sai mật khẩu: thấy thông báo lỗi của máy chủ, không bị chuyển trang.

### 7.3 `authFetch` và đăng xuất (Hà Nam)

1. Trong `api.js`, viết `authFetch(url, options)`: thêm header `Authorization: Bearer <token>` rồi gọi `fetch`.
2. Nếu phản hồi là 401: xóa ba khóa trong `localStorage` và đưa người dùng về `/auth`.
3. Viết các hàm gọi API quản trị dựa trên `authFetch`: đổi trạng thái nút, lấy, thêm và xóa khung giờ. Kiệt và Hải Nam dùng các hàm này ở 7.5 và 7.6.
4. Nút đăng xuất ở `Sidebar` của trang tìm đường và của trang quản trị: xóa ba khóa, chuyển sang `/auth`.

**Vì sao cần bước 3:** từ tuần 5, mọi đường dẫn `/api/admin` đòi token. Bản Madrid gọi `fetch` trần ở trang quản trị; với backend của nhóm, cách đó nhận về 401.

### 7.4 Khung trang quản trị (Hà Nam)

1. `AdminDashboard`: một state ghi tab đang mở, `stations` hoặc `peakhours`.
2. `Sidebar`: hai nút đổi tab, nút đang chọn có màu khác; tên người dùng lấy từ `localStorage`; nút đăng xuất.
3. `Header`: tiêu đề "Manchester Metrolink Admin Dashboard".
4. Vùng nội dung: hiển thị `StationTable` hoặc `PeakHoursAdmin` theo tab.
5. Tạo sẵn hai file `StationTable.jsx` và `PeakHoursAdmin.jsx`, mỗi file chỉ in ra tên của mình, để Kiệt và Hải Nam có chỗ viết vào.

### 7.5 `StationTable.jsx` và `ToggleSwitch.jsx` (Kiệt)

**`ToggleSwitch`:** một nút có `role="switch"`, nhận `checked`, `disabled`, `onChange`; đổi màu và vị trí núm theo `checked`.

**`StationTable`:**

1. State: danh sách nút, cờ đang tải, lỗi, từ khóa tìm kiếm, và một bảng ghi nút nào đang chờ máy chủ trả lời.
2. Khi mở: gọi `/api/stations`. Ở đây dùng thẳng tên cột của CSDL (`node_id`, `stop_name`…), vì đây là bảng quản trị dữ liệu.
3. Hiển thị bốn trạng thái: đang tải; lỗi kèm nút "Thử lại"; không có kết quả tìm kiếm; bảng dữ liệu.
4. Bảng có bốn cột: ID, tên ga, tọa độ (4 chữ số thập phân), trạng thái kèm công tắc.
5. Ô tìm kiếm lọc theo tên ga hoặc theo ID, không phân biệt hoa thường.
6. Khi gạt công tắc của một nút:
   - Khóa công tắc của nút đó.
   - Gọi hàm đổi trạng thái ở 7.3 với `stationId` và trạng thái mới (0 hoặc 1).
   - Thành công thì cập nhật dòng đó trong state; thất bại thì giữ nguyên và báo lỗi.
   - Mở khóa công tắc dù thành công hay thất bại.
7. Dòng cuối bảng ghi "Hiển thị X / 192 nút".

**Điểm cần hiểu:** bảng có 192 dòng vì mỗi dòng là một nút (ga, tuyến). Đóng hẳn một ga nghĩa là gạt mọi dòng cùng tên.

**Kiểm tra:** gạt một công tắc, tải lại trang: trạng thái vẫn giữ. Chạy `node check_sync.js`: in "ĐỒNG BỘ".

### 7.6 `PeakHoursAdmin.jsx` (Hải Nam)

1. State: danh sách khung giờ; dữ liệu form với bốn trường (mặc định hệ số 1,5 và chờ 300 giây); trạng thái hộp xác nhận xóa.
2. Khi mở: gọi hàm lấy khung giờ ở 7.3.
3. Form thêm: hai ô `type="time"`, một ô số cho hệ số (bước 0,1), một ô số cho giây chờ.
4. Kiểm tra trước khi gửi, đúng các điều kiện của nhiệm vụ 5.5: giờ bắt đầu nhỏ hơn giờ kết thúc, hệ số từ 1 trở lên, giây chờ từ 0 trở lên. Sai thì báo ngay, không gửi.
5. Đổi hệ số và giây chờ sang kiểu số trước khi gửi. Ô nhập của trình duyệt luôn trả về chuỗi.
6. Gửi thành công: tải lại danh sách, đặt form về mặc định.
7. Bảng liệt kê các khung giờ; mỗi dòng có nút xóa.
8. Bấm xóa thì mở hộp xác nhận. Chỉ gọi API khi người dùng xác nhận; xong thì tải lại danh sách.

**Vì sao kiểm tra ở cả hai phía:** kiểm tra ở giao diện để người dùng thấy lỗi ngay; kiểm tra ở máy chủ vì ai cũng có thể gọi thẳng API mà không qua giao diện.

**Kiểm tra:** thêm khung 12:00–13:00, hệ số 2, chờ 600. Sang trang tìm đường, tìm Altrincham đi Bury lúc 12:00: ra 94,5 phút (giao diện hiện 95). Xóa khung giờ đó: trở lại 62 phút.

### 7.7 `Toast.jsx` (Kiệt)

1. `ToastContainer` nhận danh sách thông báo và hàm xóa; hiển thị chồng ở góc dưới phải.
2. Mỗi thông báo tự biến mất sau 3 giây: dùng `setTimeout` trong `useEffect`, và hủy hẹn giờ trong hàm dọn dẹp.
3. Hai kiểu: thành công và lỗi.
4. Gắn vào `StationTable`: báo sau mỗi lần gạt công tắc.

Hiệu ứng chuyển động (bản Madrid dùng `framer-motion`) là không bắt buộc.

### 7.8 Tài khoản admin và phân quyền (Hà Nam)

1. Nếu chưa có: đăng ký tài khoản `admin` qua giao diện, rồi nâng quyền bằng lệnh `UPDATE` ở nhiệm vụ 5.6. Đăng xuất và đăng nhập lại.
2. Thử lần lượt và ghi kết quả:

   | Phép thử | Kết quả đúng |
   |---|---|
   | Chưa đăng nhập, mở `/admin` | Chuyển về `/auth` |
   | Đăng nhập tài khoản thường, mở `/admin` | Chuyển về `/user` |
   | Đăng nhập admin | Vào thẳng `/admin` |
   | Tài khoản thường, sửa `role` trong `localStorage` thành `admin` bằng DevTools, mở `/admin` | Thấy khung trang quản trị, nhưng danh sách khung giờ không tải được và gạt công tắc bị từ chối (403) |
   | Xóa `token` trong `localStorage`, gạt một công tắc | Bị đưa về `/auth` |

Dòng thứ tư là phép thử quan trọng nhất: nó cho thấy lớp bảo vệ thật nằm ở máy chủ.

### 7.9 Thử tác động lên kết quả tìm đường (Hải Nam và Kiệt)

Mọi thao tác làm qua giao diện quản trị, tìm đường qua trang người dùng. Sau mỗi dòng, chạy `node check_sync.js`.

| Thao tác trên trang quản trị | Tìm đường | Kết quả đúng |
|---|---|---|
| Chưa đổi gì | Bury đi Piccadilly, 12:00 | 38 phút, không đổi tuyến |
| Đóng cả 4 nút Market Street | Bury đi Piccadilly, 12:00 | 47 phút, 1 lần đổi tuyến |
| Mở lại Market Street | Bury đi Piccadilly, 12:00 | 38 phút |
| Đóng riêng nút 178 (Victoria, Navy Line) | Manchester Airport đi Rochdale Town Centre, 12:00 | 117 phút, điểm đổi tuyến không còn là Victoria |
| Đóng cả 6 nút Victoria | Manchester Airport đi Rochdale Town Centre, 12:00 | Báo không tìm thấy đường |
| Đóng riêng nút 176 (nút đầu tiên của Victoria), mở các nút Victoria còn lại | Victoria đi Bury, 12:00 | Vẫn tìm được đường |
| Mở lại mọi nút Victoria | Manchester Airport đi Rochdale Town Centre, 12:00 | 117 phút, đổi tuyến tại Victoria |
| Thêm khung 12:00–13:00, hệ số 2, chờ 600 | Altrincham đi Bury, 12:00 | 94,5 phút |
| Xóa khung giờ vừa thêm | Altrincham đi Bury, 12:00 | 62 phút |

Kết thúc: 192 nút đều mở, còn đúng 2 khung giờ mặc định, `check_sync.js` in "ĐỒNG BỘ".

### 7.10 Nâng cấp tùy chọn

Mỗi nâng cấp đi qua nhiều tầng. Chỉ làm khi cả nhóm đồng ý, và ghi lại để tuần 8 sửa báo cáo.

| Nâng cấp | Các chỗ phải sửa |
|---|---|
| Bảng ga hiện tên tuyến của từng nút | Ô 4 của `database_builder.ipynb` (thêm cột `line` vào bảng `tram` của MySQL); `/api/stations`; `StationTable`; import lại MySQL |
| Nút "Đóng cả ga" | `StationTable` (gọi đổi trạng thái cho mọi nút cùng tên), hoặc một endpoint mới nhận tên ga |
| Lộ trình hiện tên tuyến | `main.cpp` (xem bảng nâng cấp của tuần 4); `RouteList` |
| Chế độ 2 tối ưu số chặng | `main.cpp`; `evaluate.py`; mục "Hạn chế" của báo cáo |

## Nghiệm thu cuối tuần

- [ ] Bốn route hoạt động; chưa đăng nhập thì không vào được `/user` và `/admin`.
- [ ] Đăng ký, đăng nhập, đăng xuất chạy trên giao diện.
- [ ] Năm phép thử phân quyền ở 7.8 đều đúng.
- [ ] Bảng ga hiện 192 dòng, tìm kiếm được, gạt công tắc được.
- [ ] Trang khung giờ hiện 2 khung mặc định; thêm và xóa được; đầu vào sai bị chặn ở giao diện.
- [ ] Chín dòng ở 7.9 đều đúng; `check_sync.js` in "ĐỒNG BỘ" sau khi thử.
- [ ] Mọi lời gọi tới `/api/admin` đều qua `authFetch`.
- [ ] `npm run build` không báo lỗi.

## Câu hỏi cả nhóm phải trả lời được

1. Token được tạo ở đâu, lưu ở đâu, gửi lại bằng cách nào, kiểm tra ở đâu?
2. Người dùng thường tự sửa `role` trong `localStorage` thành `admin` thì thấy gì và làm được gì?
3. Gạt công tắc của một nút: liệt kê mọi nơi dữ liệu thay đổi, cho tới khi lần tìm đường kế tiếp cho kết quả khác.
4. Vì sao đóng cả ga Victoria thì không còn đường tới Rochdale, còn đóng cả ga Market Street thì Bury đi Piccadilly vẫn có đường?
5. Vì sao bảng quản trị có 192 dòng chứ không phải 99?
6. Vì sao `ProtectedRoute` không đủ để bảo vệ trang quản trị?

<details>
<summary>Gợi ý đáp án</summary>

1. Tạo ở `POST /api/login` bằng `jwt.sign`; lưu ở `localStorage` của trình duyệt; gửi trong header `Authorization: Bearer …` qua `authFetch`; kiểm tra bằng `jwt.verify` trong middleware của `/api/admin`.
2. Thấy khung trang quản trị và bảng ga (vì `/api/stations` là công khai). Không đọc được khung giờ và không thay đổi được gì, vì token của họ ghi `role` là `user` và máy chủ trả 403.
3. State của `StationTable`; yêu cầu `POST /api/admin/station/status`; bảng `tram` của MySQL; bảng `Tram` của SQLite; lần chạy `metro.exe` kế tiếp nạp nút với điều kiện `status = 1` nên bỏ nút đó cùng mọi cạnh nối với nó.
4. Mọi tàu đi nhánh Oldham và Rochdale đều qua Victoria, nên Victoria là điểm duy nhất nối nhánh đó với phần còn lại. Trung tâm thành phố có hai đường song song (qua Market Street và qua Exchange Square), nên đóng một đường vẫn còn đường kia.
5. Mỗi dòng là một nút (ga, tuyến); 99 ga có tổng cộng 192 nút.
6. Nó chạy trong trình duyệt và chỉ đọc `localStorage`, thứ người dùng tự sửa được. Ai cũng có thể bỏ qua giao diện và gọi thẳng API.

</details>

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| Trang quản trị trả 401 dù đã đăng nhập | Gọi `fetch` trần thay vì `authFetch`; hoặc thiếu chữ `Bearer` trước token |
| Tải lại `/admin` thì bị đưa về `/auth` | `ProtectedRoute` đọc sai tên khóa trong `localStorage` |
| Đăng nhập admin nhưng vào `/user` | Chưa chạy lệnh `UPDATE` nâng quyền, hoặc chưa đăng nhập lại sau khi nâng quyền |
| Công tắc gạt xong tự bật lại | Đọc `status` kiểu chuỗi; MySQL trả số 0 và 1, hãy so bằng số |
| Thêm khung giờ bị máy chủ từ chối 400 | Gửi hệ số và giây chờ dưới dạng chuỗi, hoặc giờ bắt đầu lớn hơn giờ kết thúc |
| Thông báo không tự tắt, hoặc tắt nhầm cái khác | Thiếu hàm dọn dẹp hủy `setTimeout` trong `useEffect` |

## Đối chiếu đáp án

- [HUONG_DAN_BUILD.md](../HUONG_DAN_BUILD.md), Bước 8.2, 8.6 và phần kiểm tra quản trị ở Bước 9.
- Bản Madrid: `Project-AI-2025.2-main/frontend/src/App.jsx`, `components/AuthPage.jsx` và thư mục `components/Admin/`. Bản này không gửi token khi gọi API quản trị.
