# Handbook Infrastructure (`/infra`)

Thư mục này chứa toàn bộ cấu hình hạ tầng (Infrastructure) dưới dạng Docker và Docker Compose cho dự án **Handbook**, bao gồm các môi trường chạy cục bộ (Local Development), môi trường Production (DigitalOcean Droplet), và hệ thống giám sát (Monitoring) tích hợp Prometheus & Grafana.

---

## 📂 Cấu trúc thư mục

```text
infra/
├── grafana/
│   └── provisioning/
│       ├── dashboards/
│       │   ├── dashboard.yml            # Khai báo nhà cung cấp dashboard
│       │   └── handbook_dashboard.json  # Định nghĩa Dashboard giám sát CPU/RAM
│       └── datasources/
│           └── datasource.yml          # Cấu hình tự động nạp nguồn dữ liệu Prometheus
├── prometheus/
│   └── prometheus.yml                  # Cấu hình tần suất scrape và các targets giám sát
├── docker-compose.yml                  # Cấu hình Docker Compose cho Local Development
├── docker-compose.prod.yml             # Cấu hình Docker Compose cho Production
└── README.md                           # Hướng dẫn chi tiết hạ tầng (Tài liệu này)
```

---

## 🐳 Môi trường Docker Compose

Dự án sử dụng mạng cầu nối (Bridge Network) `handbook-network` để các dịch vụ giao tiếp với nhau qua tên container (Container Name).

### 1. Local Development (`docker-compose.yml`)
Khởi chạy toàn bộ hệ thống gồm các dịch vụ backend, frontend và cơ sở dữ liệu local:

| Dịch vụ | Docker Image / Build Context | Cổng Host | Mục đích |
| :--- | :--- | :--- | :--- |
| **mongodb** | `mongo:latest` | `27017` | Database chính local |
| **redis** | `redis:alpine` | `6379` | Cache, event broker & rate limiting |
| **server-api** | `./server-api` | `8000` | REST API Server (Express.js) |
| **realtime-server** | `./realtime-server` | `5000` | Socket.IO Server (Realtime) |
| **client** | `./client` | `3000` | Frontend (Next.js 15 App Router) |
| **prometheus** | `prom/prometheus:latest` | `9090` | Thu thập metrics hiệu năng |
| **grafana** | `grafana/grafana:latest` | `4000` | Trực quan hóa metrics dạng đồ thị |

### 2. Production (`docker-compose.prod.yml`)
Sử dụng khi triển khai trên server thực tế (ví dụ: DigitalOcean Droplet). Môi trường production tối ưu hóa tài nguyên bằng cách:
* **Không chạy container `client`**: Next.js client được tối ưu hóa triển khai độc lập trên Vercel.
* **Không chạy container `mongodb`**: Dữ liệu lưu trữ tập trung trên dịch vụ đám mây MongoDB Atlas.
* Chạy các thành phần backend và giám sát: `server-api`, `realtime-server`, `redis`, `prometheus`, `grafana`.

---

## 📈 Giám sát hệ thống (Monitoring System)

Hạ tầng tích hợp sẵn bộ đôi **Prometheus & Grafana** để theo dõi hiệu năng hệ thống theo thời gian thực.

### Thu thập dữ liệu (Metrics Scraping)
* **Prometheus** được cấu hình tự động gửi request lấy thông số tại đường dẫn `/metrics` của `server-api` và `realtime-server` sau mỗi `15s`.
* Các endpoint phát xuất dữ liệu:
  * Express API Server: `http://localhost:8000/metrics`
  * Realtime Socket Server: `http://localhost:5000/metrics`

### Giao diện giám sát (UIs)
* **Prometheus UI**: Truy cập `http://localhost:9090` để thực hiện các truy vấn PromQL trực tiếp.
* **Grafana Dashboard**: Truy cập `http://localhost:4000`
  * **Tài khoản mặc định**: `admin` / `admin`
  * **Nguồn dữ liệu (Datasource)**: Đã tự động kết nối sẵn tới Prometheus (`http://prometheus:9090`).
  * **Bản đồ giám sát (Dashboard)**: Đã cấu hình nạp sẵn dashboard **Handbook System Monitoring** hiển thị 2 thông số quan trọng:
    1. **Node.js Resident Memory Usage (RAM)** (`process_resident_memory_bytes`): Theo dõi bộ nhớ RAM thực tế đang tiêu thụ bởi mỗi tiến trình Node.js.
    2. **CPU Usage (V8)** (`rate(process_cpu_user_seconds_total[1m])`): Theo dõi tải CPU của V8 engine trong mỗi 1 phút.

---

## 🚀 Các lệnh vận hành nhanh (Cheat Sheet)

Mọi câu lệnh cần được thực thi từ thư mục `infra/`.

### Quản lý Local Development
```bash
# Khởi chạy tất cả các dịch vụ chạy nền
docker compose up -d

# Xem log thời gian thực của toàn bộ hệ thống
docker compose logs -f

# Xem log của một dịch vụ cụ thể (vd: server-api)
docker compose logs -f server-api

# Dừng và xóa toàn bộ container & network local (giữ lại data volumes)
docker compose down

# Dừng và xóa toàn bộ container kèm theo dữ liệu volume (Clean setup)
docker compose down -v
```

### Quản lý Production
```bash
# Khởi chạy & build lại các container khi có cập nhật code mới
docker compose -f docker-compose.prod.yml up -d --build

# Xem log trên môi trường production
docker compose -f docker-compose.prod.yml logs -f

# Dừng hệ thống production
docker compose -f docker-compose.prod.yml down
```

---

## 💾 Phục hồi & Dữ liệu (Volumes)
Dữ liệu của các cơ sở dữ liệu và hệ thống giám sát được duy trì ổn định (persistent) qua các Docker volume sau:
* `mongodb-data`: Lưu trữ dữ liệu MongoDB (chỉ ở môi trường local).
* `redis-data`: Lưu trữ cache và phiên làm việc của Redis.
* `prometheus-data`: Lưu trữ dữ liệu lịch sử các điểm metrics đã thu thập.
* `grafana-data`: Lưu trữ cấu hình người dùng và trạng thái của Grafana.
