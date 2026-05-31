# Bài 5 — Reset Offsets: Phát lại message từ đầu

> **Chuẩn bị:** `docker exec -it kafka bash` → `export PATH=$PATH:/opt/kafka/bin`  
>
> ⚠️ **Điều kiện bắt buộc:** Để reset offset, group phải **không có consumer nào đang chạy**. Dừng tất cả consumer của group trước khi thực hiện.

---

## 1. Xem trạng thái group trước khi reset

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Tác dụng:** Xem offset hiện tại và lag để biết điểm xuất phát trước khi thay đổi.  
**Kết quả ví dụ:**

```text
TOPIC        PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG
third_topic  0          5               5               0
third_topic  1          4               4               0
third_topic  2          3               3               0
```

> `LAG = 0` → group đã đọc hết. Sau khi reset, `CURRENT-OFFSET` sẽ về 0 và `LAG` sẽ bằng tổng số message đã có trong topic.

---

## 2. Thử trước (dry-run) — xem sẽ reset về đâu mà không thay đổi gì

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-first-application \
  --reset-offsets --to-earliest \
  --topic third_topic \
  --dry-run
```

**Tác dụng:** Tính toán và **in ra** offset sẽ được đặt — nhưng **chưa thực sự thay đổi** gì cả. Dùng để kiểm tra trước khi thực hiện.  
**Kết quả:**

```text
TOPIC        PARTITION  NEW-OFFSET
third_topic  0          0
third_topic  1          0
third_topic  2          0
```

> `NEW-OFFSET = 0` cho tất cả partition — tức là sẽ đọc lại từ tin đầu tiên.

---

## 3. Thực hiện reset

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-first-application \
  --reset-offsets --to-earliest \
  --topic third_topic \
  --execute
```

**Tác dụng:** **Thực sự di chuyển offset** về đầu topic. Phải có `--execute` mới thực sự thay đổi — thiếu flag này Kafka sẽ từ chối.  
**Kết quả:** In ra bảng xác nhận tương tự dry-run, nhưng lần này đã lưu thật.

**Kiểm tra — xác nhận offset đã về 0:**

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** `CURRENT-OFFSET = 0` trên cả 3 partition, `LAG` bây giờ bằng tổng số message trong topic.

---

## 4. Consume lại — thấy toàn bộ message từ đầu

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic third_topic --group my-first-application
```

**Tác dụng:** Vì offset đã được reset về 0, consumer đọc lại **toàn bộ message từ tin đầu tiên** — dù không dùng `--from-beginning`.  
**Kết quả:** Tất cả message cũ trong topic hiện ra, rồi consumer chờ tin mới.

**Kiểm tra sau khi đọc xong:**

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** `LAG = 0` lại — group đã đọc hết một lần nữa.

---

## Các tùy chọn reset khác

| Flag | Ý nghĩa | Ví dụ |
|------|---------|-------|
| `--to-earliest` | Reset về tin đầu tiên (offset = 0) | Phát lại toàn bộ lịch sử |
| `--to-latest` | Nhảy đến cuối, bỏ qua toàn bộ tin cũ | Bỏ qua backlog, chỉ đọc tin mới |
| `--to-offset N` | Set về offset cụ thể | `--to-offset 10` → bắt đầu từ tin thứ 10 |
| `--shift-by N` | Lùi hoặc tiến N bước so với hiện tại | `--shift-by -3` → đọc lại 3 tin cuối |
| `--to-datetime` | Reset về một thời điểm cụ thể | `--to-datetime 2024-01-15T08:00:00.000` |

**Ví dụ — bỏ qua toàn bộ tin cũ, chỉ đọc tin mới từ bây giờ:**

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group my-first-application \
  --reset-offsets --to-latest \
  --topic third_topic \
  --execute
```

**Kiểm tra:**

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** `CURRENT-OFFSET = LOG-END-OFFSET`, `LAG = 0` — consumer sẽ chỉ thấy tin gửi từ bây giờ trở đi.
