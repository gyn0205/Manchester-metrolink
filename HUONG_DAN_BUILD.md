# Hướng dẫn build Manchester Metrolink Routing từ đầu

Tài liệu này dựng lại project tìm đường Madrid Metro cho mạng tram Manchester Metrolink, trong thư mục `C:\Project Ai\Manchester metrolink`. Làm lần lượt từ Bước 1 đến Bước 9; mỗi bước có phần "Kết quả đúng" để bạn tự kiểm tra trước khi đi tiếp.

**Đã chạy thử ngày 03/10/2026 trên máy này (trong thư mục tạm):** lệnh copy ở Bước 2, pipeline dữ liệu ở Bước 4 và 5, phần sửa và biên dịch C++ ở Bước 7, phần sửa frontend ở Bước 8 (build bằng `vite build` thành công), và Bước 10. **Chưa chạy thử:** import MySQL (Bước 6), `server.js` (Bước 7.3–7.4) và giao diện trên trình duyệt, vì cần mật khẩu MySQL của bạn. Các con số "Kết quả đúng" lấy từ feed TfGM tải ngày 03/10/2026; feed cập nhật hằng đêm nên số của bạn có thể lệch đôi chút.

## Tổng quan

Phần nào làm gì so với bản Madrid:

| Phần | Cách làm |
|---|---|
| `data_pipeline/` (2 notebook) | Viết lại hoàn toàn, vì cấu trúc ga của Manchester khác Madrid |
| `backend/main.cpp` | Copy rồi sửa 4 chỗ |
| `backend/server.js` | Copy rồi sửa 3 dòng |
| `frontend/` | Copy rồi sửa tâm bản đồ, các nhãn "Madrid" và danh sách gợi ý ga |
| `database.sql` | Sinh mới từ notebook, không dùng file dump của Madrid |

Cấu trúc thư mục sau khi xong:

```text
Manchester metrolink/
├── backend/
│   ├── main.cpp            # Lõi A* (C++)
│   ├── metro.exe           # Biên dịch ở Bước 7
│   ├── metrolink.db        # SQLite, sinh ở Bước 5
│   ├── server.js           # API Express
│   ├── evaluate.py         # Đánh giá A* với Dijkstra (Bước 10)
│   └── sqlite3.c, sqlite3.h, package.json
├── data_pipeline/
│   ├── raw_data/           # Feed GTFS của TfGM
│   ├── processed_data/
│   ├── data_cleaner.ipynb
│   └── database_builder.ipynb
├── frontend/
├── database.sql            # Sinh ở Bước 5, import vào MySQL ở Bước 6
└── report/
```

## Bước 1. Chuẩn bị công cụ

Tình trạng trên máy bạn (kiểm tra ngày 03/10/2026):

| Công cụ | Tình trạng | Việc cần làm |
|---|---|---|
| Python 3.11.9 | Có | Cài thêm `pandas` và `ipykernel` |
| Node.js v22.15.0 | Có | Không |
| g++ / gcc 8.1.0 (MinGW) | Có | Không |
| MySQL Server 8.0 | Dịch vụ `MySQL80` đang chạy | Cần biết mật khẩu tài khoản `root` |
| MySQL Workbench 8.0 | Có | Không |
| Extension Jupyter của VS Code | Có | Không |

Mở PowerShell và chạy:

```powershell
python -m pip install pandas ipykernel
```

Hai điều cần biết trước:

- **Cổng 3000 đang bị Docker Desktop chiếm.** `server.js` cũng nghe cổng 3000, nên trước Bước 7 bạn phải dừng container đang publish cổng này (`docker ps` để xem, `docker stop <tên>` để dừng). Nếu không, `node server.js` báo `EADDRINUSE`.
- **Chỉ chạy một MySQL.** Máy có cả MySQL80 lẫn XAMPP. Đừng bật MySQL trong XAMPP khi MySQL80 đang chạy, vì cả hai cùng dùng cổng 3306.

## Bước 2. Tạo khung project và copy phần dùng lại

```powershell
$src = "C:\Project Ai\Project-AI-2025.2-main"
$dst = "C:\Project Ai\Manchester metrolink"

New-Item -ItemType Directory -Force "$dst\backend", "$dst\data_pipeline\raw_data" | Out-Null

Copy-Item -Path "$src\backend\main.cpp", "$src\backend\server.js", "$src\backend\package.json", "$src\backend\package-lock.json", "$src\backend\sqlite3.c", "$src\backend\sqlite3.h", "$src\backend\.gitignore" -Destination "$dst\backend"

robocopy "$src\frontend" "$dst\frontend" /E /XD node_modules dist

Copy-Item -Path "$src\.gitignore" -Destination $dst
Add-Content -Path "$dst\.gitignore" -Value "`ndata_pipeline/raw_data/`ndata_pipeline/processed_data/cleaned_stop_times.csv"
```

Ghi chú:

- `robocopy` trả mã thoát 1 khi copy thành công; đó không phải lỗi.
- Không copy `metro.exe`, `sqlite3.o`, `metro_madrid.db`, `Tram.csv`, `Ket_Noi.csv`, `export.py` và `database.sql` của Madrid. Chúng sẽ được tạo mới hoặc không còn cần.
- Dòng cuối thêm dữ liệu thô vào `.gitignore`, vì feed nặng khoảng 250 MB sau khi giải nén.

**Kết quả đúng:** `backend` có 7 file, `frontend` có `src`, `public`, `index.html`, `package.json` và các file cấu hình.

## Bước 3. Tải dữ liệu GTFS của TfGM

```powershell
cd "C:\Project Ai\Manchester metrolink\data_pipeline"
curl.exe -L -o tfgm.zip "https://odata.tfgm.com/opendata/downloads/TfGMgtfsnew.zip"
Expand-Archive tfgm.zip -DestinationPath raw_data -Force
Remove-Item tfgm.zip
```

File zip khoảng 41 MB, chứa cả bus lẫn tram của Greater Manchester. Project chỉ đọc 4 file: `routes.txt`, `trips.txt`, `stop_times.txt` và `stops.txt`.

Giấy phép là ODbL v1.0 và OGL v3.0. Khi dùng trong báo cáo hoặc giao diện, ghi câu: "Contains Transport for Greater Manchester data".

**Kết quả đúng:** `raw_data` có 9 file `.txt`, trong đó `stop_times.txt` khoảng 160 MB.

## Bước 4. Làm sạch dữ liệu: `data_cleaner.ipynb`

Trong VS Code, tạo file `data_pipeline/data_cleaner.ipynb`, chọn kernel Python 3.11, rồi tạo 4 ô code theo thứ tự dưới đây và bấm Run All.

Bốn điểm khác bản Madrid mà notebook này xử lý:

1. **Lọc tuyến.** Tram có `route_type = 0` (Madrid là 1). Feed còn 4 tuyến "Replacement bus" cũng mang type 0, phải bỏ.
2. **Cấu trúc ga.** Madrid có một `stop_id` cho mỗi cặp (ga, tuyến). Manchester có một `stop_id` cho mỗi sân ga theo chiều, và các tuyến dùng chung sân ga. Notebook gộp sân ga thành ga, rồi tạo lại nút theo (ga, tuyến) để quy tắc "cùng tên ga, khác nút = đổi tuyến" của lõi C++ vẫn đúng.
3. **Giờ quá 24h.** Feed có giờ như `25:10:00` cho chuyến qua nửa đêm. Bản Madrid trừ 24 giờ, làm thời gian chạy bị âm; ở đây giữ nguyên.
4. **Tên ga.** Mọi tên có đuôi " (Manchester Metrolink)", cần cắt.

**Ô 1: đọc dữ liệu**

```python
import os
import pandas as pd

RAW = 'raw_data'

routes = pd.read_csv(f'{RAW}/routes.txt')
stops = pd.read_csv(f'{RAW}/stops.txt', dtype={'stop_code': str})
trips = pd.read_csv(f'{RAW}/trips.txt', usecols=['route_id', 'trip_id'], dtype=str)
stop_times = pd.read_csv(
    f'{RAW}/stop_times.txt',
    usecols=['trip_id', 'arrival_time', 'departure_time', 'stop_id', 'stop_sequence'],
    dtype={'trip_id': str, 'arrival_time': str, 'departure_time': str},
)
print('routes:', len(routes), '| trips:', len(trips), '| stop_times:', len(stop_times), '| stops:', len(stops))
```

**Ô 2: lọc riêng mạng tram**

```python
# route_type = 0 là tram; bỏ các tuyến "Replacement bus" (xe buýt thay thế cũng mang type 0)
tram_routes = routes[
    (routes['route_type'] == 0)
    & ~routes['route_short_name'].str.contains('replacement', case=False)
]
tram_trips = trips.merge(tram_routes[['route_id', 'route_short_name']], on='route_id')
tram_trips = tram_trips.rename(columns={'route_short_name': 'line'})

tram_stop_times = stop_times.merge(tram_trips[['trip_id', 'line']], on='trip_id')
tram_stop_times = tram_stop_times.dropna(subset=['arrival_time', 'departure_time'])

print('Tuyến tram:', sorted(tram_routes['route_short_name'].unique()))
print('Chuyến tram:', len(tram_trips), '| dòng stop_times tram:', len(tram_stop_times))
```

**Ô 3: gộp sân ga thành ga, tạo nút (ga, tuyến)**

```python
tram_stops = stops[stops['stop_id'].isin(tram_stop_times['stop_id'])].copy()

# 11 ký tự đầu của stop_code là mã ga, ký tự cuối là số sân ga
tram_stops['station_code'] = tram_stops['stop_code'].str[:11]
tram_stops['stop_name'] = (
    tram_stops['stop_name']
    .str.replace(' (Manchester Metrolink)', '', regex=False)
    .str.strip()
)

stations = tram_stops.groupby('station_code', as_index=False).agg(
    stop_name=('stop_name', 'first'),
    stop_lat=('stop_lat', 'mean'),
    stop_lon=('stop_lon', 'mean'),
)
print('Sân ga:', len(tram_stops), '| Ga:', len(stations))

# Gắn mã ga vào stop_times, rồi thay stop_id bằng khóa nút "<mã ga>|<tuyến>"
tram_stop_times = tram_stop_times.merge(
    tram_stops[['stop_id', 'station_code']], on='stop_id'
)
tram_stop_times['stop_id'] = tram_stop_times['station_code'] + '|' + tram_stop_times['line']

nodes = (
    tram_stop_times[['stop_id', 'station_code', 'line']]
    .drop_duplicates()
    .merge(stations, on='station_code')
    .sort_values(['stop_name', 'line'])
    .reset_index(drop=True)
)
print('Nút (ga, tuyến):', len(nodes))
```

**Ô 4: lưu dữ liệu sạch**

```python
# KHÔNG trừ 24 giờ như bản Madrid: GTFS dùng giờ > 24 cho chuyến qua nửa đêm,
# trừ đi sẽ sinh thời gian chạy âm.
output_dir = 'processed_data'
os.makedirs(output_dir, exist_ok=True)

nodes[['stop_id', 'stop_name', 'stop_lat', 'stop_lon', 'line', 'station_code']].to_csv(
    os.path.join(output_dir, 'cleaned_stops.csv'), index=False
)
tram_stop_times[['trip_id', 'arrival_time', 'departure_time', 'stop_id', 'stop_sequence']].to_csv(
    os.path.join(output_dir, 'cleaned_stop_times.csv'), index=False
)
print(f"Đã dọn dẹp và lưu file vào thư mục '{output_dir}' thành công!")
```

**Kết quả đúng:**

```text
routes: 656 | trips: 69680 | stop_times: 2902195 | stops: 15693
Tuyến tram: ['Blue Line', 'Green Line', 'Navy Line', 'Pink Line', 'Purple Line', 'Red Line', 'Yellow Line']
Chuyến tram: 14745 | dòng stop_times tram: 273290
Sân ga: 199 | Ga: 99
Nút (ga, tuyến): 192
```

Thư mục `processed_data` có `cleaned_stops.csv` (192 dòng) và `cleaned_stop_times.csv` (khoảng 14 MB).

## Bước 5. Dựng đồ thị: `database_builder.ipynb`

Tạo file `data_pipeline/database_builder.ipynb` với 4 ô code dưới đây và bấm Run All.

Notebook này khác bản Madrid ở ba điểm:

- Mỗi cạnh có hướng được gộp thành một dòng bằng **trung vị** thời gian chạy, thay vì giữ 258.000 dòng thô (trong đó có 3 dòng thời gian bằng 0).
- Bảng `Tram` có sẵn cột `status`, nên `metro.exe` chạy được ngay cả khi chưa khởi động `server.js`.
- Giờ cao điểm được ghi vào **cả** SQLite lẫn `database.sql`, nên trang Admin và lõi C++ luôn thấy cùng dữ liệu. Bản Madrid chỉ có trong SQLite, làm trang Admin trống.

Giờ cao điểm 07:00–09:30 và 16:00–18:30 là giả định của nhóm (TfGM không công bố dữ liệu này); đổi ở biến `PEAK_HOURS` nếu muốn.

**Ô 1: tính nút và cạnh**

```python
import sqlite3
import pandas as pd

DB_NAME = 'metrolink.db'

stops = pd.read_csv('processed_data/cleaned_stops.csv')
stop_times = pd.read_csv('processed_data/cleaned_stop_times.csv', dtype={'trip_id': str})

# Mã hóa nút: khóa chữ "<mã ga>|<tuyến>" -> số nguyên 0, 1, 2...
stops['node_id'] = range(len(stops))
stops['status'] = 1
key_to_int = dict(zip(stops['stop_id'], stops['node_id']))

# Tìm ga kế tiếp trong từng chuyến
stop_times = stop_times.sort_values(['trip_id', 'stop_sequence'])
stop_times['next_stop_id'] = stop_times.groupby('trip_id')['stop_id'].shift(-1)
stop_times['next_arrival'] = stop_times.groupby('trip_id')['arrival_time'].shift(-1)
connections = stop_times.dropna(subset=['next_stop_id']).copy()

# Thời gian chạy (giây). pd.to_timedelta hiểu được giờ > 24 như '25:10:00'
connections['travel_time'] = (
    pd.to_timedelta(connections['next_arrival']) - pd.to_timedelta(connections['departure_time'])
).dt.total_seconds()

# Gộp mỗi cạnh có hướng thành một dòng bằng trung vị
edges = connections.groupby(['stop_id', 'next_stop_id'], as_index=False).agg(
    travel_time=('travel_time', 'median'),
    so_chuyen=('travel_time', 'size'),
)
edges['u'] = edges['stop_id'].map(key_to_int)
edges['v'] = edges['next_stop_id'].map(key_to_int)
edges['status'] = 1
```

**Ô 2: kiểm tra**

```python
print('Số dòng cạnh thô:', len(connections), '| thời gian âm:', (connections['travel_time'] < 0).sum(),
      '| bằng 0:', (connections['travel_time'] == 0).sum())
print('Số nút:', len(stops), '| Số ga:', stops['station_code'].nunique(), '| Số cạnh có hướng:', len(edges))
print('Thời gian chạy (giây): min', edges['travel_time'].min(), '| max', edges['travel_time'].max(),
      '| trung bình', round(edges['travel_time'].mean(), 1))
assert edges['travel_time'].gt(0).all(), 'Có cạnh thời gian <= 0'
assert edges[['u', 'v']].notna().all().all(), 'Có cạnh trỏ tới nút không tồn tại'
```

**Ô 3: ghi SQLite cho lõi C++**

```python
PEAK_HOURS = [  # (giờ bắt đầu, giờ kết thúc, hệ số lưu lượng, giây chờ thêm khi đổi tuyến)
    ('07:00', '09:30', 1.5, 300),
    ('16:00', '18:30', 1.3, 180),
]

conn = sqlite3.connect(DB_NAME)
stops[['node_id', 'stop_id', 'stop_name', 'stop_lat', 'stop_lon', 'line', 'status']].to_sql(
    'Tram', conn, if_exists='replace', index=False)
edges[['u', 'v', 'travel_time', 'so_chuyen', 'status']].to_sql(
    'Ket_Noi', conn, if_exists='replace', index=False)

cur = conn.cursor()
cur.execute('CREATE INDEX IF NOT EXISTS idx_u ON Ket_Noi(u)')
cur.execute('CREATE INDEX IF NOT EXISTS idx_v ON Ket_Noi(v)')
cur.execute('DROP TABLE IF EXISTS KhungGioCaoDiem')
cur.execute('''CREATE TABLE KhungGioCaoDiem (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    gio_bat_dau TEXT, gio_ket_thuc TEXT,
    he_so_luu_luong REAL, thoi_gian_cho_tau REAL)''')
cur.executemany(
    'INSERT INTO KhungGioCaoDiem (gio_bat_dau, gio_ket_thuc, he_so_luu_luong, thoi_gian_cho_tau) VALUES (?, ?, ?, ?)',
    PEAK_HOURS)
conn.commit()
conn.close()
print(f"=> HOÀN TẤT! File '{DB_NAME}' đã được tạo.")
```

**Ô 4: xuất `database.sql` cho MySQL**

```python
def q(text):
    return "'" + str(text).replace("\\", "\\\\").replace("'", "''") + "'"

lines = [
    'CREATE DATABASE IF NOT EXISTS metrolink CHARACTER SET utf8mb4;',
    'USE metrolink;',
    'DROP TABLE IF EXISTS tram;',
    '''CREATE TABLE tram (
  node_id int NOT NULL,
  stop_name varchar(255) DEFAULT NULL,
  stop_lat double DEFAULT NULL,
  stop_lon double DEFAULT NULL,
  status tinyint(1) DEFAULT '1',
  PRIMARY KEY (node_id)
);''',
    'DROP TABLE IF EXISTS ket_noi;',
    '''CREATE TABLE ket_noi (
  u int DEFAULT NULL,
  v int DEFAULT NULL,
  travel_time double DEFAULT NULL,
  status tinyint(1) DEFAULT '1'
);''',
    'DROP TABLE IF EXISTS khunggiocaodiem;',
    '''CREATE TABLE khunggiocaodiem (
  id int NOT NULL AUTO_INCREMENT,
  gio_bat_dau varchar(10) NOT NULL,
  gio_ket_thuc varchar(10) NOT NULL,
  he_so_luu_luong float NOT NULL DEFAULT '1',
  thoi_gian_cho_tau float NOT NULL DEFAULT '0',
  PRIMARY KEY (id)
);''',
    '''CREATE TABLE IF NOT EXISTS users (
  id int NOT NULL AUTO_INCREMENT,
  username varchar(255) NOT NULL,
  password varchar(255) NOT NULL,
  role enum('user','admin') DEFAULT 'user',
  created_at timestamp NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY username (username)
);''',
]
tram_rows = ',\n'.join(
    f"({r.node_id},{q(r.stop_name)},{r.stop_lat:.6f},{r.stop_lon:.6f},1)" for r in stops.itertuples())
lines.append(f'INSERT INTO tram (node_id, stop_name, stop_lat, stop_lon, status) VALUES\n{tram_rows};')
edge_rows = ',\n'.join(f"({r.u},{r.v},{r.travel_time},1)" for r in edges.itertuples())
lines.append(f'INSERT INTO ket_noi (u, v, travel_time, status) VALUES\n{edge_rows};')
peak_rows = ',\n'.join(f"({q(a)},{q(b)},{c},{d})" for a, b, c, d in PEAK_HOURS)
lines.append('INSERT INTO khunggiocaodiem (gio_bat_dau, gio_ket_thuc, he_so_luu_luong, thoi_gian_cho_tau) VALUES\n'
             f'{peak_rows};')

with open('database.sql', 'w', encoding='utf-8') as f:
    f.write('\n\n'.join(lines) + '\n')
print("=> Đã xuất 'database.sql'.")
```

**Kết quả đúng:**

```text
Số dòng cạnh thô: 258545 | thời gian âm: 0 | bằng 0: 3
Số nút: 192 | Số ga: 99 | Số cạnh có hướng: 378
Thời gian chạy (giây): min 60.0 | max 360.0 | trung bình 158.1
=> HOÀN TẤT! File 'metrolink.db' đã được tạo.
=> Đã xuất 'database.sql'.
```

Sau đó chuyển hai file về đúng chỗ:

```powershell
cd "C:\Project Ai\Manchester metrolink"
Copy-Item data_pipeline\metrolink.db backend\ -Force
Move-Item data_pipeline\database.sql . -Force
```

Mỗi lần chạy lại notebook này, hãy lặp lại hai lệnh trên **và** import lại `database.sql` (Bước 6), để SQLite và MySQL không lệch `node_id`.

## Bước 6. Tạo database MySQL

File `database.sql` tự tạo database `metrolink` với 4 bảng (`tram`, `ket_noi`, `khunggiocaodiem`, `users`) và nạp dữ liệu. Bảng `users` chỉ được tạo nếu chưa có, nên import lại không làm mất tài khoản.

**Cách 1: MySQL Workbench**

1. Mở Workbench, kết nối vào Local instance MySQL80 bằng tài khoản `root`.
2. Chọn File → Open SQL Script, mở `C:\Project Ai\Manchester metrolink\database.sql`.
3. Bấm biểu tượng tia sét (Execute) để chạy toàn bộ.

**Cách 2: dòng lệnh**

```powershell
cd "C:\Project Ai\Manchester metrolink"
cmd /c '"C:\Program Files\MySQL\MySQL Server 8.0\bin\mysql.exe" -u root -p --default-character-set=utf8mb4 < database.sql'
```

**Kết quả đúng:** chạy truy vấn sau trong Workbench và nhận 192, 378, 2.

```sql
SELECT (SELECT COUNT(*) FROM metrolink.tram) AS so_nut,
       (SELECT COUNT(*) FROM metrolink.ket_noi) AS so_canh,
       (SELECT COUNT(*) FROM metrolink.khunggiocaodiem) AS so_khung_gio;
```

## Bước 7. Backend

### 7.1. Sửa `backend/main.cpp` (4 chỗ)

**Chỗ 1: tên file SQLite** (trong hàm `main`)

```cpp
// Cũ
    sqlite3_open("metro_madrid.db", &db);
// Mới
    sqlite3_open("metrolink.db", &db);
```

**Chỗ 2: cho phép xuất phát từ mọi tuyến tại ga đi** (trong `findPathAStar`)

```cpp
// Cũ
    g_score[start_id] = 0;
    open_set.push({start_id, heuristic(start_id, goal_id), 0});
// Mới
    // Hành khách được lên bất kỳ tuyến nào tại ga xuất phát -> mọi nút cùng tên ga đều là điểm bắt đầu
    for (const auto& pair : graph_nodes) {
        if (pair.second.name == graph_nodes[start_id].name) {
            g_score[pair.first] = 0;
            open_set.push({pair.first, heuristic(pair.first, goal_id), 0});
        }
    }
```

**Chỗ 3 và 4: truy vết đường đi dừng ở bất kỳ nút xuất phát nào** (cũng trong `findPathAStar`)

```cpp
// Cũ
            while (curr != start_id) {
// Mới
            while (came_from.count(curr)) {
```

```cpp
// Cũ
            path_with_time.push_back({start_id, 0});
// Mới
            path_with_time.push_back({curr, 0});
```

Lý do của chỗ 2–4: frontend gửi xuống `node_id` của nút đầu tiên trùng tên ga, tức là một tuyến cố định. Bản gốc buộc hành khách bắt đầu trên đúng tuyến đó, nên phải trả 300 giây "đổi tuyến" ngay tại ga đi nếu tuyến tốt nhất là tuyến khác. Mình đã đo: Altrincham → Bury mất 67 phút với một lần đổi tuyến thừa nếu xuất phát từ nút Purple Line, và 62 phút đi thẳng sau khi sửa.

### 7.2. Biên dịch

```powershell
cd "C:\Project Ai\Manchester metrolink\backend"
gcc -c sqlite3.c -o sqlite3.o
g++ main.cpp sqlite3.o -o metro.exe
```

Thử ngay, không cần server:

```powershell
.\metro.exe 3 27 1 12:00
```

Bốn tham số là: `node_id` ga đi, `node_id` ga đến, chế độ (1 = nhanh nhất, 2 = ít đổi tuyến), giờ khởi hành. Với feed ngày 03/10/2026, nút 3 là Altrincham và nút 27 là Bury.

**Kết quả đúng:** một dòng JSON bắt đầu bằng `{"status": "success","total_time": 3720,` (62 phút, 25 ga).

Nếu `node_id` của bạn khác, tra bằng lệnh:

```powershell
python -c "import sqlite3; [print(r) for r in sqlite3.connect('metrolink.db').execute('SELECT node_id, stop_name, line FROM Tram WHERE stop_name IN (?, ?)', ('Altrincham', 'Bury'))]"
```

### 7.3. Sửa `backend/server.js` (3 dòng)

| Dòng | Cũ | Mới |
|---|---|---|
| 19 | `password: 'MatKhauMoi123!',` | mật khẩu `root` MySQL của bạn |
| 20 | `database: 'metro_madrid'` | `database: 'metrolink'` |
| 24 | `new sqlite3.Database('./metro_madrid.db', ...` | `new sqlite3.Database('./metrolink.db', ...` |

### 7.4. Cài thư viện và chạy

```powershell
cd "C:\Project Ai\Manchester metrolink\backend"
npm install
node server.js
```

Chỉ dùng `npm install` không kèm tên gói. README của Madrid bảo cài `mysql`, nhưng code dùng `mysql2` và `package.json` đã ghi đúng.

Phải chạy `node server.js` từ trong thư mục `backend`, vì server gọi `metro.exe` và mở `metrolink.db` theo đường dẫn tương đối.

**Kết quả đúng:** terminal in `Server is running on port 3000...` và `✅ DB đã có sẵn 2 khung giờ cao điểm!`. Mở `http://localhost:3000/api/stations` trên trình duyệt thấy mảng JSON 192 phần tử.

## Bước 8. Frontend

### 8.1. Sửa tâm bản đồ: `frontend/src/components/MapView.jsx`

```jsx
// Dòng 53, cũ
  const center = [40.4168, -3.7038];
// Mới
  const center = [53.479, -2.236];
```

```jsx
// Dòng 67, cũ
      <MapContainer center={center} zoom={13} className="h-full w-full z-0">
// Mới
      <MapContainer center={center} zoom={11} className="h-full w-full z-0">
```

Mạng Metrolink trải từ vĩ độ 53,37 đến 53,62, rộng hơn Madrid, nên dùng zoom 11.

### 8.2. Đổi nhãn "Madrid" (5 chỗ)

| File | Dòng | Cũ | Mới |
|---|---|---|---|
| `frontend/index.html` | 7 | `Madrid Metro Routing` | `Manchester Metrolink Routing` |
| `frontend/src/components/Sidebar.jsx` | 46 | `MADRID METRO` | `MANCHESTER METROLINK` |
| `frontend/src/components/Admin/Header.jsx` | 8 | `Metro Madrid Admin Dashboard` | `Manchester Metrolink Admin Dashboard` |
| `frontend/src/components/Admin/Sidebar.jsx` | 21 | `Metro Madrid` | `Metrolink` |
| `frontend/src/components/Admin/StationTable.jsx` | 86 | `Danh Sách Ga (Madrid)` | `Danh Sách Ga (Manchester)` |

### 8.3. Bỏ tên ga trùng trong ô gợi ý: `frontend/src/components/Sidebar.jsx`

Mỗi ga có một nút cho mỗi tuyến, nên danh sách gợi ý sẽ lặp tên (St Peter's Square xuất hiện 7 lần). Sửa khối `datalist` ở dòng 72–74:

```jsx
// Cũ
          <datalist id="station-list">
            {stations.map(s => <option key={s.id} value={s.name} />)}
          </datalist>
// Mới
          <datalist id="station-list">
            {[...new Set(stations.map(s => s.name))].map(name => <option key={name} value={name} />)}
          </datalist>
```

Phần còn lại của `Sidebar.jsx` giữ nguyên: nó lấy nút đầu tiên trùng tên, và nhờ chỗ sửa 2 ở Bước 7.1, lõi C++ không còn phụ thuộc nút đó thuộc tuyến nào.

### 8.4. (Tùy chọn) Đổi chữ "Đi bộ đổi tuyến"

Ở Metrolink các tuyến dùng chung sân ga, nên 300 giây đổi tuyến thực chất là thời gian chờ chuyến sau chứ không phải đi bộ. Nếu muốn nhãn đúng nghĩa, đổi "Đi bộ" thành "Chờ tàu" ở `RouteList.jsx` dòng 38 và `MapView.jsx` dòng 110, 142.

### 8.5. Cài và chạy

Mở một terminal thứ hai (terminal đầu vẫn chạy `node server.js`):

```powershell
cd "C:\Project Ai\Manchester metrolink\frontend"
npm install
npm run dev
```

Mở `http://localhost:5173`.

### 8.6. Tạo tài khoản admin

1. Trên trang đăng nhập, đăng ký một tài khoản (ví dụ `admin`).
2. Trong Workbench chạy:

```sql
UPDATE metrolink.users SET role = 'admin' WHERE username = 'admin';
```

3. Đăng xuất rồi đăng nhập lại để vào trang quản trị.

## Bước 9. Kiểm thử toàn hệ thống

Thử các hành trình sau trên web. Thời gian dưới đây là kết quả lõi C++ với feed ngày 03/10/2026.

| Ga đi | Ga đến | Chế độ | Giờ | Kết quả đúng |
|---|---|---|---|---|
| Altrincham | Bury | Nhanh nhất | 12:00 | 62 phút, 25 ga, không đổi tuyến |
| Manchester Airport | Rochdale Town Centre | Nhanh nhất | 12:00 | 117 phút, đổi tuyến tại Victoria |
| St Peter's Square | Manchester Airport | Nhanh nhất | 12:00 | 51,5 phút, không đổi tuyến |
| The Trafford Centre | East Didsbury | Nhanh nhất | 12:00 | 42 phút, đổi tuyến tại Trafford Bar |
| Altrincham | Bury | Nhanh nhất | 08:00 | 93 phút (hệ số cao điểm 1,5) |
| Manchester Airport | Rochdale Town Centre | Nhanh nhất | 08:00 | 150,5 phút, đổi tuyến tại St Peter's Square |
| Manchester Airport | Rochdale Town Centre | Ít chuyển | 12:00 | 117 phút, 1 lần đổi tuyến |

Kiểm tra thêm phần quản trị:

- Trang Giờ cao điểm hiện 2 khung giờ; thêm một khung mới rồi tìm đường lại trong khung đó, thời gian phải tăng.
- Tắt một ga trên tuyến (ví dụ Victoria), tìm lại Manchester Airport → Rochdale Town Centre, lộ trình phải đổi. Lưu ý mỗi dòng trong bảng là một nút (ga, tuyến), nên muốn đóng hẳn một ga phải tắt mọi dòng cùng tên.

## Bước 10. Đánh giá thuật toán (số liệu cho báo cáo)

Tạo file `backend/evaluate.py` với nội dung dưới đây. Script mô phỏng lại đúng logic của `main.cpp` và chạy A* cùng Dijkstra trên toàn bộ 9.702 cặp ga.

```python
"""Đánh giá thuật toán trên toàn bộ cặp ga: so sánh A* với Dijkstra (A* có h = 0).

Script mô phỏng lại đúng logic của main.cpp (cùng đồ thị, cùng heuristic, cùng cạnh đổi tuyến)
để đếm số nút mở rộng và kiểm tra A* có trả về chi phí tối ưu hay không.
Chạy trong thư mục backend:  python evaluate.py [vận tốc heuristic, m/s]
"""
import heapq
import math
import sqlite3
import statistics
import sys
import time

DB_NAME = 'metrolink.db'
TRANSFER_TIME = 300.0   # giây, giống main.cpp
# m/s, mặc định giống main.cpp; thử giá trị khác bằng: python evaluate.py 15
MAX_SPEED = float(sys.argv[1]) if len(sys.argv) > 1 else 20.0

conn = sqlite3.connect(DB_NAME)
nodes = {nid: (name, lat, lon) for nid, name, lat, lon in
         conn.execute('SELECT node_id, stop_name, stop_lat, stop_lon FROM Tram WHERE status = 1')}
adj = {nid: [] for nid in nodes}
for u, v, w in conn.execute('SELECT u, v, MIN(travel_time) FROM Ket_Noi GROUP BY u, v'):
    if u in nodes and v in nodes:
        adj[u].append((v, w, False))
peak_hours = []
for start, end, mult, wait in conn.execute(
        'SELECT gio_bat_dau, gio_ket_thuc, he_so_luu_luong, thoi_gian_cho_tau FROM KhungGioCaoDiem'):
    to_sec = lambda t: int(t[:2]) * 3600 + int(t[3:5]) * 60
    peak_hours.append((to_sec(start), to_sec(end), mult, wait))
conn.close()

by_name = {}
for nid, (name, _, _) in nodes.items():
    by_name.setdefault(name, []).append(nid)
for same_station in by_name.values():
    for u in same_station:
        for v in same_station:
            if u != v:
                adj[u].append((v, TRANSFER_TIME, True))


def haversine_time(u, v):
    lat1, lon1 = map(math.radians, nodes[u][1:])
    lat2, lon2 = map(math.radians, nodes[v][1:])
    a = math.sin((lat2 - lat1) / 2) ** 2 + math.cos(lat1) * math.cos(lat2) * math.sin((lon2 - lon1) / 2) ** 2
    return 6371000.0 * 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a)) / MAX_SPEED


def peak_at(second_of_day):
    for start, end, mult, wait in peak_hours:
        if start <= second_of_day <= end:
            return mult, wait
    return 1.0, 0.0


def search(start_name, goal_name, start_sec, use_heuristic):
    """Trả về (tổng thời gian, số lần đổi tuyến, số nút đã mở rộng)."""
    goal = by_name[goal_name][0]
    h = (lambda n: haversine_time(n, goal)) if use_heuristic else (lambda n: 0.0)
    g = {n: 0.0 for n in by_name[start_name]}
    transfers = {n: 0 for n in by_name[start_name]}
    heap = [(h(n), 0.0, n) for n in by_name[start_name]]
    heapq.heapify(heap)
    expanded = 0
    while heap:
        _, cur_g, u = heapq.heappop(heap)
        if nodes[u][0] == goal_name:
            return cur_g, transfers[u], expanded
        if cur_g > g[u]:
            continue
        expanded += 1
        mult, wait = peak_at((start_sec + int(cur_g)) % 86400)
        for v, w, is_transfer in adj[u]:
            cost = w * mult + (wait if is_transfer else 0.0)
            if cur_g + cost < g.get(v, 1e9):
                g[v] = cur_g + cost
                transfers[v] = transfers[u] + (1 if is_transfer else 0)
                heapq.heappush(heap, (g[v] + h(v), g[v], v))
    return None, None, expanded


names = sorted(by_name)
pairs = [(a, b) for a in names for b in names if a != b]
print(f'Đồ thị: {len(nodes)} nút, {len(names)} ga, '
      f'{sum(1 for e in adj.values() for x in e if not x[2])} cạnh chạy tàu, '
      f'{sum(1 for e in adj.values() for x in e if x[2])} cạnh đổi tuyến | {len(pairs)} cặp ga')

for label, start_sec in [('12:00 (thấp điểm)', 12 * 3600), ('08:00 (cao điểm)', 8 * 3600)]:
    results = {}
    for use_h in (True, False):
        t0 = time.perf_counter()
        results[use_h] = [search(a, b, start_sec, use_h) for a, b in pairs]
        results[use_h + 2] = time.perf_counter() - t0
    astar, dijkstra = results[True], results[False]
    same_cost = sum(abs(x[0] - y[0]) < 1e-6 for x, y in zip(astar, dijkstra))
    exp_a = statistics.mean(x[2] for x in astar)
    exp_d = statistics.mean(x[2] for x in dijkstra)
    times = [x[0] / 60 for x in astar]
    transfer_count = {}
    for x in astar:
        transfer_count[x[1]] = transfer_count.get(x[1], 0) + 1
    print(f'\n== Khởi hành {label} ==')
    print(f'A* trùng chi phí với Dijkstra: {same_cost}/{len(pairs)} cặp')
    print(f'Số nút mở rộng trung bình: A* = {exp_a:.1f}, Dijkstra = {exp_d:.1f} '
          f'(A* giảm {100 * (1 - exp_a / exp_d):.1f}%)')
    print(f'Thời gian chạy Python trung bình mỗi truy vấn: A* = {1000 * results[3] / len(pairs):.3f} ms, '
          f'Dijkstra = {1000 * results[2] / len(pairs):.3f} ms')
    print(f'Thời gian hành trình (phút): trung bình {statistics.mean(times):.1f}, '
          f'trung vị {statistics.median(times):.1f}, dài nhất {max(times):.1f}')
    longest = max(zip(times, pairs))
    print(f'Hành trình dài nhất: {longest[1][0]} -> {longest[1][1]}')
    print('Số lần đổi tuyến:', dict(sorted(transfer_count.items())))
```

Chạy:

```powershell
cd "C:\Project Ai\Manchester metrolink\backend"
python evaluate.py
python evaluate.py 15
```

**Kết quả đúng** (lần chạy mặc định, 20 m/s):

```text
Đồ thị: 192 nút, 99 ga, 378 cạnh chạy tàu, 390 cạnh đổi tuyến | 9702 cặp ga

== Khởi hành 12:00 (thấp điểm) ==
A* trùng chi phí với Dijkstra: 9702/9702 cặp
Số nút mở rộng trung bình: A* = 91.6, Dijkstra = 108.4 (A* giảm 15.6%)
Thời gian chạy Python trung bình mỗi truy vấn: A* = 0.228 ms, Dijkstra = 0.173 ms
Thời gian hành trình (phút): trung bình 41.1, trung vị 40.0, dài nhất 117.0
Hành trình dài nhất: Manchester Airport -> Rochdale Town Centre
Số lần đổi tuyến: {0: 3876, 1: 5826}
```

Với `python evaluate.py 15`, A* giảm 20,9% số nút mở rộng lúc 12:00 và vẫn tối ưu ở cả 9.702 cặp. Vận tốc đường thẳng lớn nhất trong dữ liệu là 14,12 m/s (Whitefield → Radcliffe), nên 15 m/s vẫn là heuristic chấp nhận được. Đừng hạ dưới 14,12 m/s.

## Lỗi thường gặp

| Triệu chứng | Nguyên nhân và cách sửa |
|---|---|
| `Error: listen EADDRINUSE: address already in use :::3000` | Docker Desktop đang chiếm cổng 3000. Dừng container đó rồi chạy lại. |
| `ER_ACCESS_DENIED_ERROR` trong terminal server | Sai mật khẩu MySQL ở `server.js` dòng 19. |
| `ER_BAD_DB_ERROR: Unknown database 'metrolink'` | Chưa import `database.sql` (Bước 6). |
| Web báo "Không tìm thấy đường đi" với mọi cặp ga | `metrolink.db` không nằm trong `backend`, hoặc bạn chạy `node server.js` từ thư mục khác. |
| Web báo "Lỗi thực thi C++" | Chưa biên dịch `metro.exe` (Bước 7.2). |
| Đường đi vô lý sau khi chạy lại notebook | SQLite và MySQL lệch `node_id`. Copy lại `metrolink.db` và import lại `database.sql`. |
| Notebook báo `No module named 'pandas'` | Kernel đang chọn khác Python đã cài pandas. Chọn lại kernel Python 3.11 ở góc trên phải notebook. |
| `ALTER TABLE Tram ADD COLUMN status` báo lỗi trùng cột | Bình thường; `server.js` bỏ qua lỗi này vì cột đã có sẵn. |

## Nguồn dữ liệu

- [GM Public Transport Schedules (GTFS) – data.gov.uk](https://www.data.gov.uk/dataset/c3ca6469-7955-4a57-8bfc-58ef2361b797/gm-public-transport-schedules-gtfs)
- Feed trực tiếp: `https://odata.tfgm.com/opendata/downloads/TfGMgtfsnew.zip`
- Contains Transport for Greater Manchester data.
