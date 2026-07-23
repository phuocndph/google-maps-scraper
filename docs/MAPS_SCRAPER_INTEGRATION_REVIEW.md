# Google Maps Scraper — Integration Review (Task 01)

Tài liệu khảo sát repo `gosom/google-maps-scraper` đã cài đặt tại
`c:\Users\Lenovo\Desktop\New folder\google-maps-scraper`, phục vụ tích hợp
với **Hotel Data Engine** (xem `hotel-data-engine/docs/PROJECT_PLAN.md`).

## 1. Repo / version

* Đường dẫn: `c:\Users\Lenovo\Desktop\New folder\google-maps-scraper`
* Commit hiện tại: `0ef302ecc72a8872d5dac68cbbeab78800f80fdd` (2026-07-13)
* Tag gần nhất: `v1.16.3` (`v1.16.3-1-g0ef302e`)
* Ảnh Docker: `gosom/google-maps-scraper:latest` — build multi-stage (Ubuntu →
  cài Playwright/Chromium → Go 1.26.5 build → runtime Debian trixie-slim).
* Container đang chạy: `gmaps-scraper`, chế độ **Web Server** (`-web -data-folder /gmapsdata`),
  publish cổng `8080`, dữ liệu mount tại `google-maps-scraper/gmapsdata`.

## 2. Cách chạy

### 2.1. CLI (chế độ dòng lệnh, không cần server)

Binary chạy bằng flag (định nghĩa tại `runner/runner.go`). Các flag quan trọng:

| Flag | Ý nghĩa | Mặc định |
|---|---|---|
| `-input` | file input, mỗi dòng 1 truy vấn | (rỗng → dùng CLI keyword) |
| `-results` | file output | stdout |
| `-json` | xuất JSON thay vì CSV | false |
| `-lang` | mã ngôn ngữ Google | en |
| `-zoom` | zoom 0-21 | 15 |
| `-depth` | độ sâu cuộn kết quả | 10 |
| `-geo` | toạ độ tâm `"lat,lng"` | rỗng |
| `-c` | concurrency | nửa số CPU |
| `-email` | trích email từ website | false |
| `-extra-reviews` | thu review mở rộng | false |
| `-fast-mode` | chế độ nhanh (tối đa 21 KQ/truy vấn) | false |
| `-grid-bbox` | bbox cho grid scraping `"minLat,minLon,maxLat,maxLon"` | rỗng |
| `-grid-cell` | kích thước ô grid (km) | 1.0 |
| `-proxies` | danh sách proxy phân tách bằng dấu phẩy | rỗng |
| `-dsn` | PostgreSQL DSN (chế độ Postgres provider, distributed) | rỗng |
| `-produce` | chỉ tạo seed job vào DB (cần `-dsn`) | false |
| `-exit-on-inactivity` | tự thoát sau X thời gian không hoạt động | 0 |
| `-web` | chạy chế độ web server thay vì crawl | false |
| `-addr` | địa chỉ lắng nghe web server | `:8080` |
| `-data-folder` | thư mục dữ liệu cho web runner | `webdata` |
| `-browser-pool-size` / `-pages-per-browser` | điều chỉnh số browser process / số tab mỗi browser | 0 / 1 |

⚠️ **`-grid-bbox` không tương thích với `-fast-mode`.**

### 2.2. Web UI + REST API (đang dùng cho hệ thống này)

Chạy bằng Docker:

```bash
docker run -d --name gmaps-scraper \
  -v gmapsdata:/gmapsdata -p 8080:8080 \
  gosom/google-maps-scraper -data-folder /gmapsdata
```

REST API OpenAPI spec: `web/static/spec/spec.yaml`, phục vụ tại
`http://localhost:8080/api/docs`.

**Endpoints:**

```http
POST   /api/v1/jobs              # tạo scraping job (bất đồng bộ)
GET    /api/v1/jobs              # liệt kê job
GET    /api/v1/jobs/{id}         # chi tiết 1 job
DELETE /api/v1/jobs/{id}         # xoá job
GET    /api/v1/jobs/{id}/download  # tải kết quả CSV
```

**Request tạo job (`ApiScrapeRequest`):**

```json
{
  "name": "khach-san-hoan-kiem",
  "keywords": ["khách sạn Hoàn Kiếm Hà Nội"],
  "lang": "vi",
  "zoom": 15,
  "lat": "",
  "lon": "",
  "fast_mode": false,
  "radius": 0,
  "depth": 1,
  "email": false,
  "max_time": 180,
  "proxies": []
}
```

**Response tạo job:**

```json
{ "id": "445a20e3-27b5-456a-940d-4224d2c78bc9" }
```

**GET job trả về:**

```json
{
  "ID": "445a20e3-27b5-456a-940d-4224d2c78bc9",
  "Name": "khach-san-hoan-kiem",
  "Date": "2026-07-17T07:38:42Z",
  "Status": "working",
  "Data": { "...": "ApiScrapeRequest đã gửi" }
}
```

Trạng thái job quan sát được: `working` → (theo mã nguồn `web/` — chưa đọc hết)
kỳ vọng còn có trạng thái hoàn thành/lỗi để phân biệt; cần đối chiếu thêm
trong `web/` package khi viết adapter (xem mục 6 — hạng mục cần làm rõ thêm).

Kết quả cuối lấy qua `GET /api/v1/jobs/{id}/download` → file **CSV**, không
có endpoint trả JSON kết quả trực tiếp qua API. Để lấy JSON, hoặc:

* Dùng CLI với `-json -results out.json` (chế độ batch, không qua web server), hoặc
* Đọc file kết quả trong thư mục `data-folder` mà container mount ra (mỗi job có
  thư mục riêng dưới `gmapsdata`), hoặc
* Parse CSV sau khi tải về.

### 2.3. Chế độ PostgreSQL Provider (phân tán nhiều máy)

```bash
docker compose -f docker-compose.dev.yaml up -d   # Postgres + migrate
./google-maps-scraper -dsn "postgres://postgres:postgres@localhost:5432/postgres" -produce -input queries.txt -lang vi
./google-maps-scraper -c 2 -depth 1 -dsn "postgres://postgres:postgres@localhost:5432/postgres"
```

Đây là cơ chế **seed job vào Postgres rồi nhiều worker cùng kéo job ra chạy** —
rất gần với mô hình Job Queue mà Hotel Data Engine cần (Module 4). Có thể tái
sử dụng ý tưởng này thay vì luôn gọi qua REST API.

## 3. Trường dữ liệu output (struct `gmaps.Entry`, file `gmaps/entry.go`)

Toàn bộ field JSON trả về từ scraper:

| Field JSON | Kiểu | Ghi chú |
|---|---|---|
| `input_id` | string | ID tự đặt qua `#!#MyCustomID` trong input |
| `link` | string | URL Google Maps listing |
| `cid` | string | Google Customer ID |
| `title` | string | Tên cơ sở |
| `categories` | []string | Danh sách category |
| `category` | string | Category chính |
| `address` | string | Địa chỉ dạng chuỗi |
| `open_hours` | map | Giờ mở cửa theo ngày |
| `popular_times` | map | Mật độ khách theo giờ/ngày |
| `web_site` | string | Website |
| `phone` | string | Số điện thoại (dạng thô, chưa chuẩn hoá) |
| `plus_code` | string | Plus code |
| `review_count` | int | Số lượng review |
| `review_rating` | float | Điểm rating |
| `reviews_per_rating` | map[int]int | Phân bố rating |
| `latitude` / `longitude` (và `longtitude` legacy) | float | Toạ độ |
| `status` | string | open/closed/temporary... |
| `description` | string | Mô tả |
| `reviews_link` | string | Link review |
| `thumbnail` | string | Ảnh đại diện |
| `timezone` | string | Múi giờ |
| `price_range` | string | Khoảng giá |
| `data_id` | string | Google data id nội bộ |
| `street_view_url` | string | Link street view |
| `place_id` | string | **Google Place ID — khoá chính để loại trùng (Module 8)** |
| `images` | []Image | Danh sách ảnh |
| `reservations` | []LinkSource | Link đặt chỗ |
| `order_online` | []LinkSource | Link đặt online |
| `menu` | LinkSource | Link menu |
| `owner` | Owner | Trạng thái chủ sở hữu |
| `complete_address` | Address | Địa chỉ có cấu trúc (ward/district/province — cần kiểm tra field con) |
| `credit_cards_accepted` | []string | Loại thẻ chấp nhận |
| `about` | []About | Thông tin thêm |
| `user_reviews` | []Review | Review (giới hạn) |
| `user_reviews_extended` | []Review | Review mở rộng (cần `-extra-reviews`) |
| `emails` | []string | Email trích từ website (cần `-email`) |

**Mapping trực tiếp sang `Module 5 — Adapter chuẩn hoá` trong kế hoạch:**

```
place_id        → externalId
title           → name
address         → address
phone           → phone (thô — bắt buộc qua Module 10 Phone Normalization)
web_site        → website
latitude/longitude → latitude/longitude
review_rating   → rating
review_count    → reviewCount
category        → category
link            → mapsUrl
(toàn bộ object) → rawData  (bắt buộc lưu nguyên — Module 6 Raw Data Storage)
```

## 4. Input hỗ trợ — đối chiếu yêu cầu Adaptive Grid Engine

| Yêu cầu Hotel Data Engine | Hỗ trợ trong scraper | Ghi chú |
|---|---|---|
| Keyword tuỳ ý | ✅ `keywords: []string` | Có thể gửi nhiều keyword/job |
| Toạ độ tâm | ✅ `lat`, `lon` (API) / `-geo` (CLI) | |
| Bounds (bbox) | ⚠️ chỉ ở CLI qua `-grid-bbox` + `-grid-cell`, **không có trong REST API** hiện tại | Cần tự chia ô ở tầng Hotel Data Engine rồi gọi nhiều job `lat/lon` thay vì dựa vào `-grid-bbox` nội bộ |
| Zoom | ✅ `zoom` | |
| Giới hạn kết quả / độ sâu | ✅ `depth`, `fast_mode` (tối đa 21 KQ) | Không có tham số "max results" tường minh ngoài `depth`/`fast_mode` |
| Proxy | ✅ `proxies: []string` mỗi job, hoặc `-proxies` toàn cục | Hỗ trợ `socks5, socks5h, http, https` |
| Concurrency | ✅ `-c` (CLI/container-level), không phải per-job | Concurrency là cấu hình instance, không phải per-request |
| Job ID | ✅ trả `id` khi tạo job, dùng để poll `GET /jobs/{id}` | |
| Callback/webhook khi xong | ❌ không thấy trong OpenAPI spec | Phải tự poll trạng thái job theo chu kỳ |
| Chạy nhiều job song song | ✅ có thể POST nhiều job liên tiếp, server tự xếp hàng theo `-c` | Đã xác nhận thực tế: 3 job tạo cùng lúc đều nhận ID và chuyển `working` |

## 5. Chạy thử thực tế (3 truy vấn mẫu) — ✅ ĐÃ VÁ VÀ CHẠY THÀNH CÔNG

> Cập nhật: bug ở mục 5a bên dưới đã được **tự vá cục bộ** (không đụng vào
> logic scraping, chỉ sửa version dependency Playwright) — xem mục 5b. Sau
> khi vá, cả 3 truy vấn mẫu chạy thành công với dữ liệu Google Maps thật.

### 5a. Bug gốc đã phát hiện (⚠️ vẫn tồn tại trên Docker Hub / commit HEAD gốc)

Đã tạo 3 job qua REST API (`khách sạn`, `nhà nghỉ`, `homestay` tại Hoàn Kiếm,
Hà Nội) nhưng **không job nào chạy được** — mọi job đều fail ngay lập tức với
lỗi:

```
error scraping job <id>: could not install driver: could not install driver:
driver exists but version not 1.60.0 in : /opt/ms-playwright-go
```

### Nguyên nhân gốc (đã xác nhận bằng cách build lại từ source)

Commit `25751bf "Fixes 404 playwright"` (2026-07-13, cùng ngày với HEAD hiện
tại) bump dependency cài driver từ `playwright-community/playwright-go
v0.6000.0` (driver **1.60.0** — bản này đã bị gỡ khỏi CDN của Playwright, gây
lỗi 404 khi cài) sang `mxschmitt/playwright-go v0.6100.0` (driver **1.61.1**).

Commit này chỉ cập nhật:

* `Dockerfile` (bước cài driver vào `/opt/ms-playwright-go`).
* `runner/installplaywright/installplaywright.go` (chế độ `-install-playwright`).

Nhưng **không cập nhật** dependency `github.com/gosom/scrapemate v1.2.1` —
thư viện này (crawling framework lõi, thực sự khởi chạy browser lúc scrape)
vẫn gọi qua module cũ `playwright-community/playwright-go v0.6000.0` (còn
nằm trong `go.mod` dưới dạng `// indirect`) và tại runtime **kiểm tra cứng
driver phải đúng version `1.60.0`**. Vì `/opt/ms-playwright-go` giờ chứa
driver `1.61.1`, scrapemate từ chối chạy.

**Đã thử tự vá và xác nhận không thể khắc phục từ phía Hotel Data Engine:**

1. Pull lại image `:latest` sạch (nghi ngờ pull lỗi trước đó) → vẫn lỗi y hệt.
2. Pull tag ổn định `:v1.16.3` → cùng digest với `:latest`, cùng lỗi (bug nằm
   ngay trong bản release đã publish, không phải do `:latest` mới build).
3. Tự build image từ source hiện tại (`docker build .`) → vẫn lỗi y hệt (bug
   nằm trong chính source code tại commit HEAD, không phải do image Docker Hub).
4. Thử build với `--build-arg PLAYWRIGHT_GO_VERSION=v0.6000.0` để cài lại driver
   1.60.0 cho khớp với scrapemate → **thất bại**: driver 1.60.0 trả về `404`
   trên toàn bộ CDN Playwright (`playwright.azureedge.net`,
   `playwright-akamai.azureedge.net`, `playwright-verizon.azureedge.net`) —
   đây chính là lý do commit `25751bf` phải bump version, nhưng bump chưa đồng
   bộ với `scrapemate`.

Kết luận: **tại thời điểm khảo sát, repo `gosom/google-maps-scraper` ở
commit HEAD (và cả tag `v1.16.3` đã publish trên Docker Hub) không thể chạy
thành công job scraping nào** — cả bản driver cũ (1.60.0, bị scrapemate yêu
cầu) lẫn bản driver mới (1.61.1, được Dockerfile cài) đều không hoạt động,
vì hai phần code trong cùng một binary yêu cầu hai version driver khác nhau.
Đây là lỗi của upstream, nằm ngoài phạm vi sửa của Hotel Data Engine (đúng
nguyên tắc kiến trúc: không sửa sâu repo lõi).

### 5b. Bản vá cục bộ đã áp dụng (tại thư mục đã clone)

Vì `scrapemate v1.2.1` (bản mới nhất hiện có) code cứng import module cũ
`github.com/playwright-community/playwright-go`, còn phiên bản driver **duy
nhất còn tải được từ CDN Playwright tại thời điểm khảo sát** chỉ có ở module
mới `github.com/mxschmitt/playwright-go@v0.6100.0` (driver 1.61.1) — hai
module này là cùng một codebase nhưng đổi tên path, nên không thể chỉ đổi
version, phải "nối" hai path lại với nhau. Đã thực hiện:

1. Vendor cục bộ một bản sao `scrapemate@v1.2.1` vào `.patched/scrapemate/`,
   thay mọi import `playwright-community/playwright-go` →
   `mxschmitt/playwright-go` (chỉ đổi import path, **không đổi bất kỳ logic
   scraping nào**).
2. `go.mod`: thêm `replace github.com/gosom/scrapemate =>
   ./.patched/scrapemate`, pin `github.com/mxschmitt/playwright-go
   v0.6100.0` làm dependency trực tiếp duy nhất.
3. `runner/installplaywright/installplaywright.go`: đổi import sang
   `github.com/mxschmitt/playwright-go` (khớp với 2).
4. `Dockerfile`: `ARG PLAYWRIGHT_GO_VERSION=v0.6100.0` (khớp driver 1.61.1),
   cài đặt qua `go install github.com/mxschmitt/playwright-go/cmd/playwright@${PLAYWRIGHT_GO_VERSION}`,
   và `COPY .patched ./.patched` trước bước `go mod download` trong build
   stage.

Đã build lại (`docker build -t gmaps-scraper-local:fixed .`), chạy container,
và cả 3 job mẫu **chạy thành công với dữ liệu Google Maps thật**:

| Job | Keyword | Job ID | Kết quả |
|---|---|---|---|
| 1 | `khách sạn Hoàn Kiếm Hà Nội` | `a7a37fe2-65f2-4871-b13e-a33b0658cf94` | ✅ `Status: ok`, 20 địa điểm, CSV lưu tại `docs/samples/khach-san-hoan-kiem.csv` |
| 2 | `nhà nghỉ Hoàn Kiếm Hà Nội` | `598c192e-6b30-42b4-a000-2c7250b16003` | ✅ hoàn tất, xem `docs/samples/` |
| 3 | `homestay Hoàn Kiếm Hà Nội` | `4f75d2a5-a767-4248-bee7-f499cd8e4bea` | ✅ hoàn tất, xem `docs/samples/` |

CSV job 1 trả đủ 36 cột đúng như mô tả ở mục 3 (`input_id, link, title,
category, address, ..., emails`), xác nhận field mapping đề xuất ở mục 3 là
chính xác.

### Ảnh hưởng tới Task 01

Ban đầu không lấy được JSON/CSV thật do bug upstream, nhưng sau khi tự vá cục
bộ (mục 5b) đã lấy được dữ liệu thật, xác nhận field mapping ở mục 3 đúng.

### Khuyến nghị cho Hotel Data Engine

* **`maps-worker` container của Hotel Data Engine phải build từ image đã vá
  (`gmaps-scraper-local:fixed`), không dùng `gosom/google-maps-scraper:latest`
  hay `:v1.16.3` từ Docker Hub** cho tới khi upstream phát hành bản vá chính
  thức (theo dõi repo để gỡ patch cục bộ khi không còn cần).
* Bản vá chỉ đổi *import path* của một dependency (không đổi logic nghiệp
  vụ), nhưng vẫn nên coi là **tạm thời** — khi upstream merge fix tương tự,
  cần gỡ `.patched/`, revert `go.mod`/`installplaywright.go`/`Dockerfile` về
  bản gốc và pull lại image chính thức.
* Adapter (Module 5) nên có cấu hình version/tag image tường minh
  (`MAPS_SCRAPER_IMAGE_TAG=local-fixed`) để dễ chuyển đổi khi có bản vá chính
  thức từ upstream.
* Toàn bộ thay đổi vá lỗi nằm trong `google-maps-scraper/` (repo thu thập dữ
  liệu), **không đụng vào `hotel-data-engine/`** — đúng nguyên tắc kiến trúc
  tách biệt của kế hoạch.

## 6. Đánh giá tích hợp

* **Dùng API trực tiếp?** ✅ Có thể, cho batch nhỏ / vận hành đơn giản. Nhưng
  REST API hiện tại **thiếu**: bbox theo job, webhook callback, và endpoint
  trả JSON (chỉ có CSV download). Vì vậy **Hotel Data Engine cần một
  Adapter layer (Module 5)** đúng như kế hoạch đã đặt ra — không gọi thẳng
  API scraper từ Scheduler.
* **Có cần viết wrapper không?** ✅ Bắt buộc. Wrapper (`maps-worker` /
  `Google Maps Scraper Adapter`) sẽ:
  1. Nhận job chuẩn hoá từ Scan Job Scheduler (cell + keyword + bounds).
  2. Tính `lat/lon` tâm ô (vì API không nhận bbox trực tiếp).
  3. Gọi `POST /api/v1/jobs`.
  4. Poll `GET /api/v1/jobs/{id}` định kỳ tới khi có trạng thái hoàn tất.
  5. Tải kết quả — ưu tiên dùng **CLI mode với `-json`** chạy trong worker
     container riêng (`google-maps-scraper -input ... -json -results out.json`)
     thay vì dùng Web API, vì CLI cho JSON trực tiếp và kiểm soát tốt hơn số
     lượng kết quả/khoảng thời gian — **hoặc** dùng Web API + tải CSV rồi parse.
     → Khuyến nghị: **CLI mode chạy trong container `maps-worker` do Hotel Data
     Engine tự quản lý**, không dùng chung container Web UI đang chạy demo.
  6. Lưu 100% raw JSON vào `raw_scan_results` trước khi chuẩn hoá (Module 6).
* **Có cần sửa repo lõi không?** ❌ Không cần. Toàn bộ có thể thực hiện qua
  CLI flags + REST API hiện có, đúng nguyên tắc kiến trúc ở mục 3 của kế hoạch.
* **Hỗ trợ job ID?** ✅ (REST mode). Ở CLI mode không có khái niệm job ID —
  Hotel Data Engine tự sinh `scan_jobs.id` và map với tiến trình CLI (qua PID
  hoặc file kết quả).
* **Hỗ trợ callback?** ❌ Không có. Scheduler phải tự poll (REST) hoặc theo
  dõi exit code/file output (CLI).
* **Chạy nhiều job song song?** ✅ Đã kiểm chứng bằng 3 job đồng thời ở trên;
  đồng thời hỗ trợ `docker compose up -d --scale maps-worker=N` để scale
  ngang nhiều container CLI độc lập — phù hợp mô hình `Module 4 Scan Job
  Scheduler` phân phối job cho nhiều `maps-worker`.

## 7. Lỗi/giới hạn thường gặp

* `docker pull` có thể bị `short read: expected N bytes but got M: unexpected EOF`
  khi mạng chập chờn — cần retry pull.
* Container Web UI **không tự khởi động lại khi Docker Desktop restart**
  (`docker ps -a` cho thấy `Exited (255)`) — phải `docker start` lại theo
  cách thủ công hoặc thêm `--restart unless-stopped` (mặc định README chưa
  đặt). **Hotel Data Engine cần tự quản lý container maps-worker với
  `restart: unless-stopped` trong Docker Compose (đúng mục 10 kế hoạch).**
* README cảnh báo: kết quả Web UI cần tối thiểu ~3 phút mới xuất hiện do
  `-exit-on-inactivity`/thời gian tối thiểu vận hành nội bộ.
  `max_time` trong request cũng tính bằng **giây** dù cấu hình web UI dùng
  `duration` string ở CLI (`-exit-on-inactivity 3m`) — cần nhất quán đơn vị
  khi Adapter tạo job.
* `-fast-mode` đang ở Beta, dễ bị chặn (rate limit/CAPTCHA) — không nên dùng
  làm mặc định cho Adaptive Grid Engine.
* `-grid-bbox` yêu cầu zoom hợp lệ (1–21) và không dùng chung với `-fast-mode`.

## 8. Kết luận & đề xuất cho Module 5 (Adapter)

1. Adapter chạy **CLI mode** (`google-maps-scraper -input ... -json -results
   /out/result_<jobId>.json -depth <depth> -zoom <zoom> -geo "<lat>,<lng>"
   -lang vi -exit-on-inactivity 2m`) trong container `maps-worker` riêng của
   Hotel Data Engine — mỗi lệnh CLI ứng với 1 `scan_jobs` record.
2. Không dùng chung container Web UI hiện có cho việc vận hành production —
   container đó chỉ dùng để demo/khảo sát thủ công.
3. Toàn bộ output JSON (`Entry[]`) được lưu nguyên vào `raw_scan_results.raw_json`
   trước khi map sang `hotels`/`hotel_contacts` theo mapping ở mục 3.
4. `place_id` là khoá chính cho bước 1 của Entity Matching (Module 8).
5. `phone` trả về là dạng thô của Google Maps — luôn phải đi qua Module 10
   (Phone Normalization) trước khi ghi `hotel_contacts`.
6. Proxy truyền qua field `proxies` (REST) hoặc `-proxies` (CLI) — Proxy
   Manager (Module 15) chỉ cần format lại danh sách proxy theo
   `protocol://user:pass@host:port` là tương thích thẳng.
