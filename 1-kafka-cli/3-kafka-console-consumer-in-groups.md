# Bài 3 — Consumer Groups: Chia partition cho nhiều consumer

> **Chuẩn bị:** Bài này cần **3–4 terminal chạy song song**.  
> Mỗi terminal: `docker exec -it kafka bash` → `export PATH=$PATH:/opt/kafka/bin`

**Khái niệm cốt lõi:** Khi nhiều consumer trong **cùng một group** đọc cùng một topic, Kafka tự động **chia partition** cho từng consumer. Mỗi partition chỉ được đọc bởi **1 consumer** trong group tại một thời điểm. Thêm consumer = giảm tải cho từng con.

---

## 1. Tạo topic với 3 partitions

**Lệnh (terminal 1):**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic third_topic --create --partitions 3
```

**Kết quả:** `Created topic third_topic.`

---

## 2. Khởi động consumer đầu tiên trong group

**Lệnh (terminal 1):**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic third_topic --group my-first-application
```

**Tác dụng:** Consumer tham gia group `my-first-application`. Là consumer duy nhất trong group, nó nhận **cả 3 partition** (partition 0, 1, 2 đều về terminal này).  
**Kết quả:** Màn hình im lặng, chờ tin.

---

## 3. Gửi tin từ producer với Round-Robin

**Lệnh (terminal 2):**

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 \
  --producer-property partitioner.class=org.apache.kafka.clients.producer.RoundRobinPartitioner \
  --topic third_topic
```

Gõ nhiều tin:

```text
> A
> B
> C
> D
> E
> F
```

**Kết quả (terminal 1):** Tất cả 6 tin hiện ở consumer 1 — nó đang giữ cả 3 partition.

---

## 4. Thêm consumer thứ 2 vào cùng group — Kafka rebalance

**Lệnh (terminal 3):**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic third_topic --group my-first-application
```

**Tác dụng:** Kafka phát hiện group có thêm consumer, tiến hành **rebalance** — tự động phân chia lại partition:

- Consumer 1 (terminal 1): nhận partition 0 và 1
- Consumer 2 (terminal 3): nhận partition 2

**Kết quả:** Gõ thêm tin ở terminal 2 và quan sát — tin vào partition 2 chỉ hiện ở terminal 3, tin vào partition 0 hoặc 1 chỉ hiện ở terminal 1.

**Kiểm tra phân phối partition:**

Mở terminal 4 (PowerShell mới) và chạy:

```bash
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group my-first-application
```

**Kết quả:** Thấy rõ `CONSUMER-ID` khác nhau đang giữ partition khác nhau.

---

## 5. Consumer thứ 3 — partition không còn để chia

Mở thêm terminal 5 và chạy:

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic third_topic --group my-first-application
```

**Tác dụng:** Consumer 3 tham gia group nhưng topic chỉ có 3 partition và đã có 2 consumer giữ hết — consumer thứ 3 **đứng chờ không làm gì**.  

> Số consumer tối đa có ích bằng số partition. Thêm consumer hơn số partition = consumer đứng không.

---

## 6. Consumer trong group khác — offset độc lập

**Lệnh (terminal bất kỳ):**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic third_topic --group my-second-application --from-beginning
```

**Tác dụng:** Group `my-second-application` có **offset hoàn toàn độc lập** với `my-first-application`. Nó đọc lại từ đầu toàn bộ lịch sử topic, không bị ảnh hưởng bởi group kia đã đọc đến đâu.  
**Kết quả:** Tất cả message từ đầu topic hiện ra.

> **Điểm quan trọng nhất của bài này:** Nhiều group cùng đọc 1 topic — mỗi group nhận **đầy đủ** tất cả message, độc lập nhau. Đây là lý do Kafka phù hợp để fan-out: 1 sự kiện → nhiều hệ thống xử lý riêng biệt (analytics, email, logging, v.v.).
