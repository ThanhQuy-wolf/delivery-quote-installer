# Delivery Quote Service

Service báo giá giao hàng chạy độc lập, gắn vào hệ thống microservice của bạn qua HTTP.
Bạn gửi hai địa chỉ (hoặc hai toạ độ), service trả về quãng đường thực tế, thời gian đi và
phí ship. Bên dưới dùng **Goong Maps API**: Autocomplete, Place Detail, Geocoding, Distance
Matrix, Directions.

Thực hiện: Thanh Quy — Seminar môn Kiến trúc hướng dịch vụ (SOA), TDTU.

---

## Cài đặt

### Cách 1 — `setup.exe` (Windows)

1. Vào **[Releases](../../releases/latest)**, tải `setup.exe`.
2. Mở `setup.exe`, chọn thư mục project của bạn. Service được tạo ở
   `<project>\delivery-quote-service`.
3. Dán `GOONG_API_KEY`, chọn cổng (mặc định 8000), bấm **Cài đặt**.

Trình cài tải đúng bản service đi kèm nó từ Releases (có kiểm tra checksum), ghi `.env`, rồi:

- **Docker đang chạy** → build image và chạy container (`docker compose up -d`).
- **Docker chưa cài hoặc chưa bật** → tạo `.venv`, cài thư viện, chạy uvicorn trong một cửa sổ
  riêng. Cần Python 3.10+. Trình cài không tự bật Docker Desktop.

Xong, nó gọi thử `/api/health` và một lượt Goong để báo key dùng được hay không. Trong thư
mục service có `start.bat` / `stop.bat` để bật/tắt về sau. Chạy lại `setup.exe` vào cùng
thư mục là cập nhật; key cũ được điền sẵn, các biến bạn tự thêm trong `.env` được giữ.

> ⚠️ `setup.exe` chưa ký số, Windows SmartScreen có thể chặn lần đầu: **More info → Run
> anyway**. Không muốn chạy exe thì dùng Cách 2.

### Cách 2 — tự cài từ file zip (mọi hệ điều hành)

Tải `delivery-quote-service-<version>.zip` ở Releases, giải nén vào project của bạn rồi:

```bash
cd delivery-quote-service
cp .env.example .env            # điền GOONG_API_KEY

# Chạy bằng Python 3.10+
pip install -r requirements.txt
uvicorn app.main:app --host 0.0.0.0 --port 8000

# …hoặc bằng Docker
docker compose up -d --build
```

### Lấy key Goong

Đăng ký ở <https://account.goong.io> và tạo key:

| Key | Bắt buộc | Dùng cho |
|---|---|---|
| **REST API key** → `GOONG_API_KEY` | Có | Mọi endpoint `/api/...`. Chỉ nằm ở server |
| **Maptiles key** → `GOONG_MAPTILES_KEY` | Không | Trang bản đồ demo ở `/` và `/api/map/config`. Key này tới browser, nên khoá URL trong dashboard Goong (vd `http://localhost:8000/*`) |

Mọi lời gọi đều tiêu quota của key trong `.env`, nên mỗi hệ thống dùng key của riêng mình.

---

## Dùng như một microservice

### Gắn vào docker-compose có sẵn

```yaml
services:
  delivery-quote:
    build: ./delivery-quote-service
    env_file: ./delivery-quote-service/.env
    restart: unless-stopped
    # Chỉ service trong compose gọi thì không cần ports; mở ra ngoài thì thêm:
    # ports: ["8000:8000"]

  order-service:                  # service của bạn
    environment:
      QUOTE_SERVICE_URL: http://delivery-quote:8000
    depends_on:
      delivery-quote:
        condition: service_healthy   # healthcheck có sẵn trong Dockerfile
```

### Gọi từ service khác

```python
async with httpx.AsyncClient(base_url=os.environ["QUOTE_SERVICE_URL"], timeout=15) as client:
    r = await client.post("/api/quote", json={
        "origin": {"lat": 10.7770, "lng": 106.6953},
        "destination": {"lat": 10.8700, "lng": 106.8030},
        "vehicle": "bike",        # bike | car | taxi | truck
    })
r.json()  # {"distance_km": 18.1, "duration_min": 69, "fee_vnd": 69000, "vehicle": "bike"}
```

Tài liệu tương tác: `http://<host>:<cổng>/docs` · đặc tả OpenAPI: `/openapi.json`.

### Các endpoint

| Method | Endpoint | Trả về |
|---|---|---|
| `GET` | `/api/health` | `{status}` |
| `GET` | `/api/places/autocomplete?q=&session=&lat=&lng=` | `[{place_id, description}]` |
| `GET` | `/api/places/detail?place_id=&session=` | `{lat, lng, formatted_address, place_id}` |
| `GET` | `/api/geocode?address=` hoặc `?latlng=` | `{lat, lng, formatted_address, place_id}` |
| `POST` | `/api/quote` | `{distance_km, duration_min, fee_vnd, vehicle}` |
| `POST` | `/api/route` | `{distance_km, duration_min, vehicle, path: [[lat, lng], …]}` |
| `GET` | `/api/map/config` | style bản đồ nền cho MapLibre (cần Maptiles key) |

Mọi lỗi đều theo **RFC 9457** (`application/problem+json`):

```json
{"type":"about:blank","title":"Address not found","status":404,
 "detail":"Goong không tìm thấy địa chỉ nào khớp.","instance":"/api/geocode"}
```

Riêng lỗi validate tham số là `422` của FastAPI, `detail` là một mảng.

### Cấu hình (`.env`)

| Biến | Mặc định | Ý nghĩa |
|---|---|---|
| `GOONG_API_KEY` | *(bắt buộc)* | REST key của Goong. Thiếu thì service không khởi động |
| `GOONG_MAPTILES_KEY` | *(trống)* | Key bản đồ nền, chỉ cần cho trang demo |
| `PORT` | `8000` | Cổng khi chạy bằng `docker compose` / `start.bat` |
| `CORS_ORIGINS` | *(trống)* | Frontend nào được gọi thẳng từ trình duyệt, cách nhau bằng dấu phẩy |
| `BASE_FEE_VND` / `FEE_PER_KM_VND` | `15000` / `3000` | Công thức phí: `cố định + theo km`, làm tròn đến nghìn |
| `SUGGESTION_LIMIT` | `6` | Số gợi ý mỗi lần gõ |
| `BIAS_LAT` / `BIAS_LNG` | `10.7769` / `106.7009` | Điểm ưu tiên gợi ý khi client không gửi vị trí |

### Những điều cần biết

- **Session token của Autocomplete:** gửi cùng một `session` (UUID) cho cả phiên gõ, rồi gửi
  lại đúng token đó ở `/api/places/detail`. Không làm vậy thì Goong tính tiền từng lần gõ.
- **Đã có `place_id` thì dùng Place Detail,** đừng geocode lại chuỗi mô tả — geocode là đoán
  lại từ đầu và có thể ra địa điểm khác.
- **Scale:** cache nằm trong bộ nhớ của process; mỗi container một worker, cần chịu tải thì
  chạy thêm container.
- **Bản đồ nền:** truy cập service qua địa chỉ khác `localhost:8000` thì thêm địa chỉ đó vào
  khoá URL của Maptiles key, không thì Goong trả 403.

---

## Tài liệu kèm theo

Có trong thư mục service sau khi cài:

- [`docs/ARCHITECTURE.md`](docs/ARCHITECTURE.md) — vì sao service được chia tầng như vậy,
  vòng đời một request, hợp đồng lỗi, cách tích hợp và mở rộng.
- [`docs/TROUBLESHOOTING.md`](docs/TROUBLESHOOTING.md) — những lỗi thật đã gặp khi làm
  service này và cách sửa.

## Giấy phép

[MIT](LICENSE).
