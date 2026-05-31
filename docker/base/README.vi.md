# docker/base — Kafka tối giản để học CLI

Một container Apache Kafka duy nhất, đơn giản nhất có thể để bạn học Kafka **từ dòng lệnh** mà không cần giao diện web Conduktor, database, hay ZooKeeper.

Bao gồm:

- **1 image duy nhất**: `apache/kafka` (image chính thức của Apache).
- **Chế độ KRaft** — Kafka tự quản lý metadata, **không cần ZooKeeper**.
- Broker đóng cả 2 vai: `broker` (nhận/gửi tin) và `controller` (quản lý cluster).
- Toàn bộ công cụ CLI `kafka-*.sh` nằm **bên trong container**, không cần cài Kafka trên máy Windows.

---

## 1. Khởi động / tắt broker

Chạy từ **thư mục gốc** (`kafka learnnninng/`).

```powershell
# Khởi động (chạy nền)
docker compose -f docker/base/docker-compose.yml up -d

# Kiểm tra trạng thái (chờ cho đến khi STATUS hiện "healthy")
docker compose -f docker/base/docker-compose.yml ps

# Xem log realtime
docker compose -f docker/base/docker-compose.yml logs -f kafka

# Tắt, giữ lại data (topic, message vẫn còn)
docker compose -f docker/base/docker-compose.yml down

# Tắt và XÓA toàn bộ data (bắt đầu sạch hoàn toàn)
docker compose -f docker/base/docker-compose.yml down -v
```

---

## 2. Mở shell bên trong broker (Windows)

Công cụ CLI Kafka nằm **trong container** tại `/opt/kafka/bin/`, không có sẵn trên máy Windows.

**Bước 1 — mở PowerShell** (Win + X → "Terminal" hoặc tìm PowerShell).

**Bước 2 — vào trong container:**

```powershell
docker exec -it kafka bash
```

Dấu nhắc lệnh đổi thành dạng `root@kafka:/#` — bạn đang ở bên trong container Linux, không còn là Windows nữa.

**Bước 3 — thêm công cụ vào PATH (làm 1 lần mỗi khi mở shell mới):**

```bash
export PATH=$PATH:/opt/kafka/bin
```

**Bước 4 — chạy lệnh CLI bất kỳ, ví dụ:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Bước 5 — khi xong, thoát khỏi container:**

```bash
exit
```

Broker vẫn tiếp tục chạy sau khi bạn thoát. Shell quay lại PowerShell của Windows.

---

### Cần mở terminal thứ 2? (producer và consumer chạy song song)

Mở **cửa sổ PowerShell mới** và lặp lại bước 2–3. Mỗi cửa sổ là một shell độc lập bên trong cùng một broker đang chạy. Đây là cách thực hành bài 3 và 5 trong `1-kafka-cli/` — cần producer và consumer chạy đồng thời.

---

### Chạy lệnh nhanh (không cần vào interactive shell)

```powershell
docker exec kafka /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
```

---

## 3. Cách kết nối

| Ai đang kết nối                        | Bootstrap server  |
|----------------------------------------|-------------------|
| CLI **bên trong** container `kafka`    | `localhost:9092`  |
| Java demos / ứng dụng **trên Windows** | `localhost:9092`  |

Port `9092` được publish ra host, và broker quảng bá địa chỉ `localhost:9092`, nên cùng 1 địa chỉ dùng được ở cả 2 nơi. Đây cũng là lý do code Java trong `kafka-beginners-course/` (trỏ đến `localhost:9092`) chạy được với broker này không cần thay đổi gì.

---

## 4. Replication factor — và khi nào cần setup multi-broker

**Replication factor** là số broker lưu bản sao của mỗi partition. RF=1 nghĩa là 1 bản sao (không có dự phòng). RF=3 nghĩa là 3 bản sao — nếu 1 broker chết, 2 broker còn lại vẫn có đủ data.

Với chỉ 1 broker ở đây, `--replication-factor` chỉ có thể là `1`. Thử `--replication-factor 2` sẽ lỗi:

```text
Replication factor: 2 larger than available brokers: 1
```

Đây không phải cấu hình sai — chỉ có 1 broker thì không thể lưu 2 bản sao.

**Để thực hành replication factor > 1**, dùng setup 3-broker:

```powershell
# Tắt broker đơn trước (cả 2 dùng port 9092)
docker compose -f docker/base/docker-compose.yml down

# Khởi động cluster 3 broker
docker compose -f docker/multi-broker/docker-compose.yml up -d
```

Xem [`../multi-broker/README.vi.md`](../multi-broker/README.vi.md) để thực hành Leader, Replicas, ISR, và bài giả lập broker bị lỗi.

---

### Nên thực hành gì ở đây trước

- **`KAFKA_NUM_PARTITIONS: 1`** — topic tự tạo mặc định có 1 partition. Các bài CLI dùng điều này có chủ ý: bạn tạo topic không có `--partitions` (được 1 partition), rồi tạo topic khác với `--partitions 3` và so sánh kết quả `--describe`. Muốn thay đổi giá trị mặc định này: sửa trong [`docker-compose.yml`](docker-compose.yml) rồi `down -v` + `up`.

---

## 5. Lỗi thường gặp

- **`Connection refused` ngay sau khi `up`** — broker cần vài giây khởi động. Chờ `ps` báo `healthy`.
- **Muốn bắt đầu lại từ đầu?** `down -v` xóa volume `kafka_data`, xóa toàn bộ topic và message.
- **Port 9092 đã bị chiếm** — có Kafka khác hoặc container cũ đang giữ port. Tắt nó đi, hoặc đổi mapping `9092:9092` trong compose file.
