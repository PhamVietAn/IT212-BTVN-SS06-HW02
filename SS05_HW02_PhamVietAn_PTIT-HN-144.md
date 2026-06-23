# BÀI 2: Tối ưu Prompt (Kỹ thuật Multiple Options, Trade-offs và Phân tích Giả định)

## 1. Nội dung Prompt sau khi tối ưu
Dưới đây là prompt đã được tối ưu hóa tích hợp cả 3 kỹ thuật: **Multiple Options** (Đa phương án), **Trade-offs** (Đánh đổi), và **Phân tích giả định (What-if)** để làm việc với AI:

```text
Hãy đóng vai trò là một System Architect giàu kinh nghiệm chuyên về hệ sinh thái Spring Boot và thiết kế hệ thống phân tán (Distributed Systems). Nhiệm vụ của bạn là tư vấn thiết kế giải pháp xử lý bất đồng bộ (Asynchronous processing) cho dịch vụ gửi thông báo (Notification Service) qua cả email và SMS tới 1 triệu người dùng khi có sự kiện khuyến mãi, nhằm tránh gây nghẽn (Timeout) cho API chính.

Yêu cầu thực hiện:
1. (Multiple Options): Đề xuất ít nhất 3 phương án kiến trúc bất đồng bộ khác nhau trong Spring Boot (đi từ giải pháp sẵn có trong framework đến giải pháp tích hợp các trung gian Message Broker).
2. (Trade-offs): Lập bảng so sánh chi tiết giữa 3 phương án trên dựa trên các tiêu chí: Độ khó triển khai, Độ tin cậy (Ngăn mất mát dữ liệu khi sập server), Khả năng mở rộng (Scalability), và Tài nguyên hệ thống tiêu thụ.
3. (Phân tích giả định - What-if Scenario): Phân tích chi tiết rủi ro: "Chuyện gì xảy ra đối với từng phương án nếu 1 triệu người dùng cùng lúc được kích hoạt sự kiện khuyến mãi trong vòng vài giây?". Đề xuất cơ chế dự phòng và kiểm soát luồng tải (Throttling/Rate Limiting) cho phương án tối ưu nhất.
4. (Code Generation): Viết mã nguồn Java Spring Boot minh họa hoàn chỉnh (sử dụng thư viện phổ biến) cho phương án mà bạn đánh giá là tối ưu nhất cho quy mô 1 triệu người dùng này.
```

---

## 2. Minh chứng thực tế: Phản hồi của AI dựa trên Prompt tối ưu

### Phản hồi từ AI:
```text
Chào bạn, dưới đây là phân tích kiến trúc chi tiết, bảng so sánh đánh đổi, kịch bản giả định và mã nguồn Java Spring Boot hoàn chỉnh được thiết kế cho hệ thống gửi thông báo quy mô 1 triệu người dùng.

---

### PHẦN 1: ĐỀ XUẤT 3 PHƯƠNG ÁN KIẾN TRÚC BẤT ĐỒNG BỘ (MULTIPLE OPTIONS)

#### Phương án 1: Sử dụng `@Async` và `ThreadPoolTaskExecutor` (Bộ nhớ trong - In-Memory Queue)
- **Cơ chế:** Khi có yêu cầu gửi thông báo, Spring Boot sẽ đẩy task vào một `BlockingQueue` nằm trên RAM của JVM. Một Thread Pool (`ThreadPoolTaskExecutor`) cấu hình sẵn sẽ nhặt các task này ra và thực thi gửi email/SMS bất đồng bộ.
- **Phù hợp:** Hệ thống vừa và nhỏ, lượng traffic thấp đến trung bình.

#### Phương án 2: Sử dụng Message Broker (RabbitMQ hoặc Apache Kafka) làm hàng đợi trung gian
- **Cơ chế:** API chính nhận yêu cầu gửi thông báo và ngay lập tức đẩy một tin nhắn (Message) chứa thông tin người nhận vào hàng đợi (Queue/Topic) của Message Broker (RabbitMQ/Kafka). Phía sau sẽ có các dịch vụ Worker (Consumer) độc lập lắng nghe hàng đợi và thực hiện gửi tin nhắn.
- **Phù hợp:** Hệ thống lớn, yêu cầu tính sẵn sàng cao, khả năng mở rộng tốt và không được phép mất mát dữ liệu.

#### Phương án 3: Sử dụng Database Queue kết hợp Spring Batch / Scheduler (Polled-base Queue)
- **Cơ chế:** Khi có sự kiện, API chính lưu thông tin thông báo cần gửi vào một bảng trong Database (ví dụ: `notification_queue` với trạng thái `PENDING`). Một tiến trình chạy ngầm (Spring Batch hoặc Spring Scheduler) sẽ liên tục truy vấn (poll) DB theo lô (chunks), gửi đi và cập nhật trạng thái thành `SENT`.
- **Phù hợp:** Xử lý hàng loạt (Batch processing) không yêu cầu thời gian thực ngay lập tức.

---

### PHẦN 2: BẢNG SO SÁNH ƯU/NHƯỢC ĐIỂM (TRADE-OFFS)

| Tiêu chí so sánh | Phương án 1: Spring `@Async` | Phương án 2: Message Broker (RabbitMQ/Kafka) | Phương án 3: Database Queue + Batch |
| :--- | :--- | :--- | :--- |
| **Độ khó triển khai** | **Rất Dễ** (Chỉ cần cấu hình Annotation `@EnableAsync` và `@Async`). | **Trung bình - Khó** (Phải cài đặt, vận hành cụm RabbitMQ/Kafka độc lập). | **Dễ - Trung bình** (Tận dụng DB hiện có, viết logic quét DB). |
| **Độ tin cậy (Mất mát dữ liệu)** | **Rất thấp** (Sập server đột ngột là mất sạch task đang đợi trong RAM). | **Rất cao** (Tin nhắn được lưu xuống đĩa cứng của Broker, có cơ chế Ack/Retry). | **Cao** (Dữ liệu đã được lưu trữ bền vững trong Database quan hệ). |
| **Khả năng mở rộng (Scalability)** | **Kém** (Bị giới hạn bởi tài nguyên RAM/CPU của một server đơn lẻ). | **Cực tốt** (Dễ dàng tăng số lượng Consumer độc lập để chia tải). | **Trung bình** (Bị nghẽn ở nút cổ chai kết nối Database - Database Connection Pool). |
| **Tài nguyên tiêu thụ** | Tiêu thụ nhiều RAM/CPU khi luồng queue đầy. | Ít ảnh hưởng RAM ứng dụng Spring Boot, phân tán tải tốt. | Gây tải lớn lên Database do liên tục thực hiện các lệnh đọc/ghi (I/O). |

---

### PHẦN 3: PHÂN TÍCH KỊCH BẢN GIẢ ĐỊNH (WHAT-IF Distributed SCENARIO)

**Giả định: 1 triệu người dùng cùng lúc kích hoạt gửi thông báo khuyến mãi trong vài giây.**

* **Đối với Phương án 1 (Spring `@Async`):**
  - *Hậu quả sập nguồn:* RAM ứng dụng Spring Boot sẽ bị tràn ngay lập tức. Nếu cấu hình hàng đợi của ThreadPool không giới hạn, máy chủ sẽ bị lỗi **OutOfMemoryError (OOM)** và crash JVM. Nếu giới hạn hàng đợi, hệ thống sẽ ném ra `TaskRejectedException`, từ chối và làm mất hàng trăm ngàn lượt gửi của khách hàng.
* **Đối với Phương án 3 (DB Queue + Batch):**
  - *Hậu quả sập nguồn:* Ghi đồng thời 1 triệu bản ghi vào DB sẽ làm nghẽn Database. CPU của Database sẽ tăng vọt lên 100%, gây ảnh hưởng đến toàn bộ các dịch vụ nghiệp vụ khác (như thanh toán, mua sắm).
* **Đối với Phương án 2 (RabbitMQ / Kafka):**
  - *Hậu quả và giải pháp:* Đây là phương án chống chịu tốt nhất. 1 triệu tin nhắn sẽ được đẩy cực nhanh vào Message Broker. Broker đóng vai trò như một bộ đệm giảm chấn (Buffer). Phía sau, chúng ta áp dụng cơ chế **Throttling/Rate Limiting**: Giới hạn số lượng tin nhắn xử lý đồng thời của Consumer thông qua thuộc tính `prefetch_count`. Consumer sẽ gửi thư thong thả (ví dụ: 1000 email/giây) tương thích với hạn mức (Rate limit) của nhà cung cấp dịch vụ Email/SMS (SendGrid, Twilio) mà không làm sập hệ thống.

---

### PHẦN 4: MÃ NGUỒN JAVA SPRING BOOT MINH HỌA (RABBITMQ + CONSUMER RATE LIMITING)

Dưới đây là mã nguồn Spring Boot hoàn chỉnh sử dụng **Spring AMQP (RabbitMQ)** để triển khai hàng đợi bất đồng bộ, có cơ chế giới hạn tốc độ xử lý để gửi 1 triệu tin nhắn an toàn.

#### 1. File cấu hình: `application.yml`
```yaml
spring:
  rabbitmq:
    host: localhost
    port: 5672
    username: guest
    password: guest
    listener:
      simple:
        # Số lượng message tối đa một worker nhận tại một thời điểm
        # Giúp khống chế luồng tải (Throttling) không bị quá tải RAM/CPU
        prefetch: 50
        concurrency: 5 # Số lượng thread chạy song song để consume tin nhắn
        max-concurrency: 10
```

#### 2. Định nghĩa cấu hình hàng đợi: `RabbitMQConfig.java`
```java
package com.example.notification.config;

import org.springframework.amqp.core.*;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class RabbitMQConfig {

    public static final String NOTIFICATION_QUEUE = "promo.notification.queue";
    public static final String NOTIFICATION_EXCHANGE = "promo.notification.exchange";
    public static final String NOTIFICATION_ROUTING_KEY = "promo.notification.routingKey";

    @Bean
    public Queue notificationQueue() {
        // Hàng đợi bền vững (durable = true) để tránh mất mát dữ liệu khi RabbitMQ restart
        return QueueBuilder.durable(NOTIFICATION_QUEUE).build();
    }

    @Bean
    public TopicExchange notificationExchange() {
        return new TopicExchange(NOTIFICATION_EXCHANGE);
    }

    @Bean
    public Binding bindingNotificationQueue(Queue notificationQueue, TopicExchange notificationExchange) {
        return BindingBuilder.bind(notificationQueue).to(notificationExchange).with(NOTIFICATION_ROUTING_KEY);
    }
}
```

#### 3. Định nghĩa DTO: `NotificationMessage.java`
```java
package com.example.notification.dto;

import java.io.Serializable;

public class NotificationMessage implements Serializable {
    private static final long serialVersionUID = 1L;

    private String userId;
    private String email;
    private String phoneNumber;
    private String content;

    // Constructors, Getters and Setters
    public NotificationMessage() {}

    public NotificationMessage(String userId, String email, String phoneNumber, String content) {
        this.userId = userId;
        this.email = email;
        this.phoneNumber = phoneNumber;
        this.content = content;
    }

    public String getUserId() { return userId; }
    public void setUserId(String userId) { this.userId = userId; }
    public String getEmail() { return email; }
    public void setEmail(String email) { this.email = email; }
    public String getPhoneNumber() { return phoneNumber; }
    public void setPhoneNumber(String phoneNumber) { this.phoneNumber = phoneNumber; }
    public String getContent() { return content; }
    public void setContent(String content) { this.content = content; }
}
```

#### 4. Lớp gửi Message vào Queue: `NotificationPublisher.java`
```java
package com.example.notification.publisher;

import com.example.notification.config.RabbitMQConfig;
import com.example.notification.dto.NotificationMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.core.RabbitTemplate;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class NotificationPublisher {
    
    private static final Logger log = LoggerFactory.getLogger(NotificationPublisher.class);

    @Autowired
    private RabbitTemplate rabbitTemplate;

    public void publishToQueue(NotificationMessage message) {
        // Đẩy tin nhắn vào Exchange, RabbitMQ sẽ tự định tuyến vào Queue tương ứng
        rabbitTemplate.convertAndSend(
            RabbitMQConfig.NOTIFICATION_EXCHANGE,
            RabbitMQConfig.NOTIFICATION_ROUTING_KEY,
            message
        );
        log.info("Đã gửi message của User {} vào RabbitMQ Queue thành công.", message.getUserId());
    }
}
```

#### 5. Lớp xử lý Message bất đồng bộ (Consumer): `NotificationConsumer.java`
```java
package com.example.notification.consumer;

import com.example.notification.config.RabbitMQConfig;
import com.example.notification.dto.NotificationMessage;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.amqp.rabbit.annotation.RabbitListener;
import org.springframework.stereotype.Service;

@Service
public class NotificationConsumer {

    private static final Logger log = LoggerFactory.getLogger(NotificationConsumer.class);

    @RabbitListener(queues = RabbitMQConfig.NOTIFICATION_QUEUE)
    public void consumeMessage(NotificationMessage message) {
        log.info("Bắt đầu xử lý gửi thông báo cho User: {}", message.getUserId());
        try {
            // Giả lập kết nối mạng và gửi Email/SMS thông qua API bên thứ ba (SendGrid/Twilio)
            // Quá trình này tốn khoảng 200ms
            sendEmail(message.getEmail(), message.getContent());
            sendSMS(message.getPhoneNumber(), message.getContent());
            
            log.info("Hoàn thành gửi thông báo cho User: {}", message.getUserId());
        } catch (Exception e) {
            log.error("Lỗi khi gửi thông báo cho User: {}, Chi tiết: {}", message.getUserId(), e.getMessage());
            // Có thể bổ sung cơ chế Retry hoặc đẩy vào Dead Letter Queue (DLQ) nếu cần thiết
        }
    }

    private void sendEmail(String email, String content) throws InterruptedException {
        Thread.sleep(100); // Giả lập I/O block
    }

    private void sendSMS(String phone, String content) throws InterruptedException {
        Thread.sleep(100); // Giả lập I/O block
    }
}
```
```
