# Bài 1 — kafka-console-producer.sh: Gửi tin nhắn

> **Chuẩn bị:** `docker exec -it kafka bash` → `export PATH=$PATH:/opt/kafka/bin`  
> Broker: `localhost:9092`

---

## 1. Tạo topic để thực hành

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic first_topic --create --partitions 1
```

**Tác dụng:** Tạo `first_topic` với 1 partition để producer gửi tin vào.  
**Kết quả:** `Created topic first_topic.`

**Kiểm tra:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
```

**Kết quả:** `first_topic` có trong danh sách.

---

## 2. Gửi tin nhắn cơ bản (chế độ tương tác)

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic first_topic
```

**Tác dụng:** Mở chế độ nhập liệu tương tác. Mỗi dòng bạn gõ và nhấn Enter là 1 message được gửi vào topic.  
**Kết quả:** Dấu nhắc `>` xuất hiện, chờ bạn gõ.

Gõ vài tin (nhấn Enter sau mỗi dòng):

```text
> Hello World
> My name is Kafka
> I am learning Kafka
> ^C
```

> `Ctrl + C` để thoát producer.

**Kiểm tra — đọc lại các tin vừa gửi:**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic first_topic --from-beginning
```

**Kết quả:** 3 tin nhắn hiện ra, rồi consumer chờ thêm (`Ctrl+C` để thoát consumer).

---

## 3. Gửi tin với `acks=all` (xác nhận an toàn nhất)

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic first_topic --producer-property acks=all
```

**Tác dụng:** Producer không coi gửi là thành công cho đến khi **tất cả ISR xác nhận** đã nhận được message. Đây là chế độ bền vững nhất — tránh mất data ngay cả khi broker chết ngay sau khi gửi.  
**Kết quả:** Tương tự trước, dấu `>` xuất hiện.

```text
> tin nhắn được đảm bảo
> gửi an toàn
> ^C
```

> Với 1 broker (setup hiện tại), `acks=all` và `acks=1` hoạt động giống nhau vì chỉ có 1 ISR. Sự khác biệt rõ ràng hơn khi có nhiều broker.

---

## 4. Gửi vào topic chưa tồn tại — Kafka tự tạo

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic new_topic
```

**Tác dụng:** Kafka nhận diện `new_topic` chưa tồn tại và **tự động tạo** topic đó với số partition mặc định (`KAFKA_NUM_PARTITIONS=1`).  
**Kết quả:** Có thể hiện warning "UnknownTopicOrPartitionException" rồi dấu `>` xuất hiện.

```text
> hello world
> ^C
```

**Kiểm tra — xem topic được tự tạo với mấy partition:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --list
kafka-topics.sh --bootstrap-server localhost:9092 --topic new_topic --describe
```

**Kết quả:** `new_topic` có 1 partition (mặc định).

> ⚠️ **Thực tế:** Nên tạo topic thủ công với số partition phù hợp **trước** khi produce. Topic tự tạo thường chỉ có 1 partition — không đủ throughput cho production. Thay đổi số partition mặc định bằng `KAFKA_NUM_PARTITIONS` trong `docker/base/docker-compose.yml`.

---

## 5. Thử tự tạo topic với partition mặc định cao hơn

Thay đổi `KAFKA_NUM_PARTITIONS: 3` trong `docker/base/docker-compose.yml`, sau đó:

```powershell
docker compose -f docker/base/docker-compose.yml down -v
docker compose -f docker/base/docker-compose.yml up -d
docker exec -it kafka bash
```

```bash
export PATH=$PATH:/opt/kafka/bin
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic new_topic_2
```

```text
> hello again
> ^C
```

**Kiểm tra:**

```bash
kafka-topics.sh --bootstrap-server localhost:9092 --topic new_topic_2 --describe
```

**Kết quả:** `new_topic_2` lần này có 3 partition — vì mặc định đã đổi.

---

## 6. Gửi tin kèm key

```bash
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic first_topic \
  --property parse.key=true \
  --property key.separator=:
```

**Tác dụng:** Bật chế độ gửi kèm key. Format nhập: `key:value`. Key xác định message đi vào partition nào — **cùng key luôn vào cùng partition**, đảm bảo thứ tự xử lý cho cùng một đối tượng.  
**Kết quả:** Dấu `>` xuất hiện.

```text
> user_1:{"action":"login"}
> user_2:{"action":"purchase"}
> user_1:{"action":"logout"}
> ^C
```

**Kiểm tra — đọc lại kèm key để xác nhận:**

```bash
kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic first_topic \
  --property print.key=true \
  --property key.separator=" => " \
  --from-beginning
```

**Kết quả:** Thấy key và value riêng biệt, ví dụ:

```text
user_1 => {"action":"login"}
user_2 => {"action":"purchase"}
user_1 => {"action":"logout"}
```

> Hai message của `user_1` luôn nằm trên cùng 1 partition → được xử lý theo đúng thứ tự `login` trước `logout`.
