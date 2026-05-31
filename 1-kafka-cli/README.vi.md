# 1 — Kafka CLI căn bản

Thực hành toàn bộ các công cụ dòng lệnh Kafka, dùng broker tối giản tại [`../docker/base/`](../docker/base/). Không có giao diện web — chỉ có CLI.

## Chuẩn bị

1. **Khởi động broker** (từ thư mục gốc của repo):

   ```powershell
   docker compose -f docker/base/docker-compose.yml up -d
   ```

2. **Mở shell bên trong broker:**

   ```powershell
   docker exec -it kafka bash
   ```

3. **Thêm công cụ vào PATH (mỗi lần mở shell mới làm 1 lần):**

   ```bash
   export PATH=$PATH:/opt/kafka/bin
   ```

Sau đó các lệnh trong các file bên dưới chạy đúng y như viết. Broker luôn là `--bootstrap-server localhost:9092`.

> Các file `.md` là **bài học từng bước** — đọc và chạy từng lệnh một, quan sát kết quả trước khi làm tiếp. Đừng copy toàn bộ một lúc.

## Các bài học (theo thứ tự)

| File | Nội dung |
|------|----------|
| [`0-kafka-topics.md`](0-kafka-topics.md) | Tạo, liệt kê, mô tả, xóa topic; partitions & replication |
| [`1-kafka-console-producer.md`](1-kafka-console-producer.md) | Gửi tin nhắn; `acks`; keys; tự tạo topic |
| [`2-kafka-console-consumer.md`](2-kafka-console-consumer.md) | Nhận tin; `--from-beginning`; in key/timestamp/partition |
| [`3-kafka-console-consumer-in-groups.md`](3-kafka-console-consumer-in-groups.md) | Consumer group; cách partition chia cho nhiều consumer |
| [`4-kafka-consumer-groups.md`](4-kafka-consumer-groups.md) | Xem group, lag, và offset |
| [`5-reset-offsets.md`](5-reset-offsets.md) | Phát lại data bằng cách reset offset của group |

## Mẹo: nhiều terminal

Bài 2, 3 cần producer và consumer chạy đồng thời. Mở thêm cửa sổ PowerShell, chạy `docker exec -it kafka bash`, rồi `export PATH=$PATH:/opt/kafka/bin` trong mỗi cửa sổ mới.
