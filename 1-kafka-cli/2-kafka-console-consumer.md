# Bài 2 — kafka-console-consumer.sh: Nhận tin nhắn

> **Chuẩn bị:** Bài này cần **2 terminal chạy song song**.
>
> - **Terminal 1** (consumer): `docker exec -it kafka bash` → `export PATH=$PATH:/opt/kafka/bin`
> - **Terminal 2** (producer): mở PowerShell mới → `docker exec -it kafka bash` → `export PATH=$PATH:/opt/kafka/bin`

---

## 1. Tạo topic với 3 partitions

**Lệnh (terminal 1):**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic second_topic --create --partitions 3
```

**Kết quả:** `Created topic second_topic.`

---

## 2. Chạy consumer — chờ message đến

**Lệnh (terminal 1):**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic second_topic
```

**Tác dụng:** Consumer kết nối và chờ. **Chỉ nhận message mới từ thời điểm này trở đi** — không đọc lại message cũ đã có trong topic.  
**Kết quả:** Màn hình im lặng, cursor nhấp nháy chờ.

---

## 3. Gửi tin từ terminal 2 — quan sát consumer nhận ngay lập tức

**Lệnh (terminal 2):**

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --producer-property partitioner.class=org.apache.kafka.clients.producer.RoundRobinPartitioner \
  --topic second_topic
```

**Tác dụng:** Mở producer với Round-Robin partitioner — phân phối đều message sang các partition lần lượt thay vì dồn vào 1 partition.

Gõ vài tin:

```text
> tin 1
> tin 2
> tin 3
> ^C
```

**Kết quả (terminal 1):** 3 tin nhắn xuất hiện ở consumer gần như ngay lập tức.

---

## 4. Consumer mặc định không đọc lại tin cũ

Dừng consumer (Ctrl+C ở terminal 1). Gửi thêm vài tin từ terminal 2. Khởi động lại consumer:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic second_topic
```

**Kết quả:** Consumer chờ im, **không hiện** các tin đã gửi trước khi nó chạy. Consumer bình thường chỉ đọc từ vị trí hiện tại trở đi.

---

## 5. Đọc lại từ đầu với `--from-beginning`

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic second_topic --from-beginning
```

**Tác dụng:** Đọc **toàn bộ message từ tin đầu tiên** của topic, bao gồm cả tin đã gửi trước khi consumer khởi động.  
**Kết quả:** Tất cả tin trước đó hiện ra, rồi consumer tiếp tục chờ tin mới.

> **Lưu ý thứ tự:** Thứ tự tin có thể không đúng theo thứ tự gửi vì topic có 3 partition — consumer đọc hết partition 0 trước, rồi 1, rồi 2. Thứ tự **chỉ đảm bảo trong cùng 1 partition**.

---

## 6. Hiển thị đầy đủ metadata: key, partition, timestamp

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic second_topic \
  --formatter kafka.tools.DefaultMessageFormatter \
  --property print.timestamp=true \
  --property print.key=true \
  --property print.value=true \
  --property print.partition=true \
  --from-beginning
```

**Tác dụng:** Hiển thị mỗi message kèm đầy đủ thông tin: thời gian, key, partition nó nằm trên, và nội dung.  
**Kết quả mong đợi:**

```text
CreateTime:1748123456789  Partition:0  null  tin 1
CreateTime:1748123456790  Partition:2  null  tin 2
CreateTime:1748123456791  Partition:1  null  tin 3
```

**Đọc kết quả:**

| Cột | Ý nghĩa |
|-----|---------|
| `CreateTime:...` | Thời điểm message được tạo (Unix timestamp ms) |
| `Partition:0` | Partition mà message này nằm trên |
| `null` | Key — `null` vì producer không gửi kèm key |
| `tin 1` | Nội dung message |

> Chú ý tin 1, 2, 3 nằm trên các partition khác nhau (0, 2, 1) do Round-Robin phân phối.
