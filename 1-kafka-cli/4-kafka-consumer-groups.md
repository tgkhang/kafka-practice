# Bài 4 — kafka-consumer-groups.sh: Xem thông tin group

> **Chuẩn bị:** `docker exec -it kafka bash` → `export PATH=$PATH:/opt/kafka/bin`  
> **Yêu cầu:** Đã làm bài 1–3 và có các group `my-first-application`, `my-second-application` với dữ liệu trong topic.

---

## 1. Xem trợ giúp

```bash
kafka-consumer-groups.sh
```

**Kết quả:** Danh sách các flag: `--list`, `--describe`, `--reset-offsets`, v.v.

---

## 2. Liệt kê tất cả consumer group

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
```

**Tác dụng:** In tên tất cả group đang hoặc đã từng tồn tại trên cluster.  
**Kết quả mong đợi:**

```text
my-first-application
my-second-application
```

---

## 3. Xem chi tiết một group không có consumer đang chạy

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-second-application
```

**Tác dụng:** Hiển thị trạng thái của group: offset hiện tại, offset cuối topic, và **lag** (số message chưa đọc) cho từng partition.  
**Kết quả mong đợi:**

```text
GROUP                  TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG  CONSUMER-ID  HOST  CLIENT-ID
my-second-application  third_topic  0          6               6               0    -            -     -
my-second-application  third_topic  1          4               4               0    -            -     -
my-second-application  third_topic  2          5               5               0    -            -     -
```

**Đọc kết quả:**

| Cột | Ý nghĩa |
|-----|---------|
| `CURRENT-OFFSET` | Offset mà group đã đọc đến (vị trí tin cuối đã xử lý + 1) |
| `LOG-END-OFFSET` | Offset của tin mới nhất trong partition |
| `LAG` | `LOG-END-OFFSET - CURRENT-OFFSET` = số tin chưa đọc |
| `CONSUMER-ID` | `-` nghĩa là không có consumer nào đang hoạt động trong group |

> `LAG = 0` → group đã đọc hết tất cả message. Nếu LAG > 0 → consumer đang bị tụt hậu so với producer.

---

## 4. Xem group trong khi consumer đang chạy

**Mở terminal 2**, khởi động consumer:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic first_topic --group my-first-application
```

**Quay lại terminal 1**, describe lại:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** Cột `CONSUMER-ID` có giá trị (dạng `consumer-my-first-application-1-xxxx`), cột `HOST` hiện IP, và `LAG = 0` vì consumer đang đọc realtime.

**Kiểm tra sau khi dừng consumer:**

Ctrl+C ở terminal 2, rồi describe lại:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** `CONSUMER-ID` trở về `-`, nhưng offset vẫn được lưu — group nhớ đã đọc đến đâu dù không có consumer nào đang chạy.

---

## 5. Gửi thêm tin và xem lag tăng lên

Từ một terminal khác, gửi thêm tin vào `first_topic`:

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic first_topic
```

```text
> tin mới 1
> tin mới 2
> tin mới 3
> ^C
```

Describe group:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** `LAG = 3` — group chưa đọc 3 tin mới vừa gửi.

**Khởi động consumer để xử lý lag:**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic first_topic --group my-first-application
```

**Kết quả:** 3 tin mới hiện ra. Describe lại → `LAG = 0`.

---

## Tổng kết: lag là gì và tại sao quan trọng

LAG là chỉ số sức khỏe quan trọng nhất của consumer group:

- `LAG = 0` → consumer xử lý kịp tốc độ producer.
- `LAG tăng liên tục` → consumer quá chậm, cần tăng số consumer trong group hoặc tối ưu code xử lý.
- `LAG đột ngột lớn` → consumer có thể đã chết và không ai nhận ra.
