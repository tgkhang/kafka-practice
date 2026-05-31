# docker/multi-broker — Cluster 3 broker để thực hành replication

Dùng khi bạn muốn thực hành **replication factor > 1**.  
Cho các bài CLI cơ bản (topic/producer/consumer), `../base/` đơn giản hơn.

---

## Replication factor là gì

Mỗi partition của một topic được lưu trên N broker, trong đó N = replication factor.

```txt
Topic "orders", 3 partitions, RF = 3

  Partition 0 → lưu trên broker 1 (leader), broker 2, broker 3
  Partition 1 → lưu trên broker 2 (leader), broker 3, broker 1
  Partition 2 → lưu trên broker 3 (leader), broker 1, broker 2
```

- **Leader** — broker xử lý toàn bộ đọc/ghi cho partition đó.
- Các broker còn lại là **replica** (follower) — sao chép data từ leader.
- Nếu leader chết, Kafka tự bầu replica mới làm leader.
- `--replication-factor 2`: bất kỳ 2 trong 3 broker đều lưu data. Mất 1 broker vẫn đọc/ghi được.
- `--replication-factor 3`: cả 3 lưu data. Mất 2 broker vẫn sống sót (còn 1 ISR).

Với chỉ 1 broker (`../base/`), không có chỗ để lưu bản sao thứ 2, nên `--replication-factor 2` lỗi. Ở đây có 3 broker, nên 1, 2, hoặc 3 đều hoạt động.

---

## Khởi động / tắt

Chạy từ **thư mục gốc** (`kafka learnnninng/`).

```powershell
# Khởi động 3 broker
docker compose -f docker/multi-broker/docker-compose.yml up -d

# Kiểm tra cả 3 đều hiện "Up"
docker compose -f docker/multi-broker/docker-compose.yml ps

# Tắt, giữ data
docker compose -f docker/multi-broker/docker-compose.yml down

# Tắt và xóa toàn bộ data
docker compose -f docker/multi-broker/docker-compose.yml down -v
```

> **Conflict port:** `../base/` và setup này cùng dùng port 9092. Tắt một cái trước khi khởi động cái kia.

---

## Mở shell để thực hành CLI

Vào **kafka1** (tương tự `../base/README.vi.md`):

```powershell
docker exec -it kafka1 bash
```

```bash
export PATH=$PATH:/opt/kafka/bin
```

Dùng `--bootstrap-server localhost:9092` cho tất cả lệnh — bạn đang nói chuyện với kafka1, và kafka1 biết về cả 3 broker nên sẽ định tuyến đúng.

---

## Những thứ cần thử

**Tạo topic với RF=3 (cả 3 broker đều lưu mỗi partition):**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --topic my-replicated-topic \
  --partitions 3 --replication-factor 3
```

**Mô tả topic — chú ý Leader, Replicas, Isr trên mỗi partition:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic my-replicated-topic
```

Kết quả trông như sau:

```text
Partition: 0   Leader: 3   Replicas: 3,1,2   Isr: 3,1,2
Partition: 1   Leader: 1   Replicas: 1,2,3   Isr: 1,2,3
Partition: 2   Leader: 2   Replicas: 2,3,1   Isr: 2,3,1
```

- **Leader** — broker nào đang xử lý đọc/ghi cho partition này.
- **Replicas** — tất cả broker lưu bản sao (số lượng luôn bằng RF).
- **Isr** (In-Sync Replicas) — replica đã đồng bộ đầy đủ với leader. Bình thường: số Isr = số Replicas.

**Giả lập broker bị lỗi:**

Mở PowerShell thứ 2 và tắt kafka2:

```powershell
docker stop kafka2
```

Quay lại shell trong kafka1, mô tả topic lại:

```bash
kafka-topics.sh --bootstrap-server localhost:9092 \
  --describe --topic my-replicated-topic
```

Bạn thấy Isr thu hẹp lại (kafka2 biến mất) và partition nào có leader là kafka2 sẽ bầu leader mới từ broker còn lại. Topic vẫn đọc/ghi bình thường.

Khôi phục kafka2:

```powershell
docker start kafka2
```

Mô tả lại — kafka2 tái gia nhập Isr khi đồng bộ dữ liệu xong.

---

## ISR vs Replicas

| Replicas | Isr     | Ý nghĩa                                  |
|----------|---------|------------------------------------------|
| `1,2,3`  | `1,2,3` | Tất cả bình thường                       |
| `1,2,3`  | `1,3`   | Broker 2 đang lag hoặc đã ngừng hoạt động |
| `1,2,3`  | `1`     | Chỉ còn 1 broker; rủi ro mất data        |

`min.insync.replicas` (đặt là 2 trong compose file này) nghĩa là producer với `acks=all` cần ít nhất 2 ISR mới xác nhận ghi thành công. Nếu ISR xuống dưới 2, ghi với `acks=all` sẽ bị từ chối — cơ chế bảo vệ chống mất data.
