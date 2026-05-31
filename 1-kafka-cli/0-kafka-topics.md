# Bài 0 — kafka-topics.sh: Quản lý Topic

> **Chuẩn bị trước khi bắt đầu:**
>
> ```powershell
> docker compose -f docker/base/docker-compose.yml up -d
> docker exec -it kafka bash
> ```
>
> ```bash
> export PATH=$PATH:/opt/kafka/bin
> ```
>
> Chạy từng lệnh bên dưới **một cái một**, quan sát kết quả trước khi làm tiếp.

---

## 1. Xem trợ giúp của công cụ

```bash
kafka-topics.sh
```

**Tác dụng:** In ra tất cả tùy chọn có sẵn của lệnh `kafka-topics.sh`.  
**Kết quả:** Danh sách dài các flag như `--create`, `--list`, `--describe`, `--delete`, v.v.

---

## 2. Liệt kê tất cả topic hiện có

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Tác dụng:** Kết nối đến broker và in tên tất cả topic đang tồn tại.  
**Kết quả mong đợi:** Trống (chưa có topic nào), hoặc danh sách tên nếu đã tạo trước đó.

---

## 3. Tạo topic với số partition mặc định

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic first_topic --create
```

**Tác dụng:** Tạo topic tên `first_topic` với số partition mặc định là 1 (do `KAFKA_NUM_PARTITIONS=1` trong `docker/base/docker-compose.yml`).  
**Kết quả:**

```text
Created topic first_topic.
```

**Kiểm tra — xác nhận topic đã xuất hiện:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Kết quả:** `first_topic` có trong danh sách.

---

## 4. Tạo topic với 3 partitions

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic second_topic --create --partitions 3
```

**Tác dụng:** Tạo `second_topic` với đúng 3 partition. Nhiều partition = nhiều consumer đọc song song được, throughput cao hơn.  
**Kết quả:**

```text
Created topic second_topic.
```

**Kiểm tra — mô tả topic để thấy 3 partition:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic second_topic --describe
```

**Kết quả:** 3 dòng, mỗi dòng ứng với `Partition: 0`, `Partition: 1`, `Partition: 2`.

---

## 5. Thử tạo topic với replication-factor 2 — sẽ lỗi

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic third_topic --create --partitions 3 --replication-factor 2
```

**Tác dụng:** Thử yêu cầu 2 bản sao dữ liệu — nhưng chỉ có 1 broker, không có chỗ để lưu bản sao thứ 2.  
**Kết quả mong đợi (lỗi có chủ ý):**

```text
Replication factor: 2 larger than available brokers: 1
```

> Đây là kết quả **đúng** — không phải lỗi cấu hình. Setup này chỉ có 1 broker nên RF tối đa là 1. Để thực hành RF > 1 xem `../docker/multi-broker/`.

---

## 6. Tạo topic với replication-factor 1 — thành công

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic third_topic --create --partitions 3 --replication-factor 1
```

**Tác dụng:** Tạo `third_topic` với 3 partition, mỗi partition 1 bản sao — hợp lệ với 1 broker.  
**Kết quả:**

```text
Created topic third_topic.
```

**Kiểm tra — liệt kê để thấy cả 3 topic:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Kết quả:** Cả 3 xuất hiện: `first_topic`, `second_topic`, `third_topic`.

---

## 7. Mô tả chi tiết một topic

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic first_topic --describe
```

**Tác dụng:** Hiển thị cấu trúc đầy đủ của topic: số partition, replication factor, leader broker của từng partition, danh sách replica và ISR.  
**Kết quả mong đợi:**

```text
Topic: first_topic  PartitionCount: 1  ReplicationFactor: 1  Configs:
  Topic: first_topic  Partition: 0  Leader: 1  Replicas: 1  Isr: 1
```

**Đọc kết quả:**

| Trường | Ý nghĩa |
|--------|---------|
| `Leader: 1` | Broker số 1 đang xử lý partition này |
| `Replicas: 1` | Bản sao nằm trên broker 1 |
| `Isr: 1` | Broker 1 đã đồng bộ đầy đủ (In-Sync) |

**So sánh với second_topic có 3 partition:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic second_topic --describe
```

**Kết quả:** 3 dòng Partition với cùng Leader/Replicas/Isr (vì chỉ có 1 broker).

---

## 8. Xóa một topic

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic first_topic --delete
```

**Tác dụng:** Xóa hoàn toàn topic `first_topic` và toàn bộ message bên trong.  
**Kết quả:** Không có output — lệnh im lặng khi thành công.

**Kiểm tra — xác nhận đã xóa:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Kết quả:** `first_topic` không còn trong danh sách nữa.
