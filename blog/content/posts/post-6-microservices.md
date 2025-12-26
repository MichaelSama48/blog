---
title: "Lập trình mạng với Microservices và Hệ thống phân tán"
date: 2025-12-15
description: "Tại sao chia nhỏ lại tốt hơn? Đi sâu vào Service Discovery (Eureka), API Gateway, Circuit Breaker và giao tiếp bất đồng bộ qua Message Queue."
cover:
    image: "/images/p6.jpg"
    alt: "Microservices Ecosystem"
    caption: "Hệ sinh thái Microservices phức tạp nhưng mạnh mẽ."
tags: ["Microservices", "Spring Cloud", "Node.js", "RabbitMQ", "Architecture", "Deep Dive"]
categories: ["System Architecture", "Network Programming"]
---

"Chia để trị" (Divide and Conquer) là một chiến thuật quân sự cổ điển, và nó cũng đúng tuyệt đối trong thiết kế phần mềm hiện đại. Khi một ứng dụng lớn đến mức việc build mất 30 phút, việc fix một bug nhỏ làm sập cả hệ thống, đó là lúc **Monolith (Nguyên khối)** cần phải nhường chỗ cho **Microservices (Vi dịch vụ)**.

Bài viết này sẽ không chỉ dừng lại ở lý thuyết. Chúng ta sẽ phân tích các design pattern quan trọng nhất để các service "nói chuyện" với nhau một cách tin cậy trong môi trường mạng đầy rẫy rủi ro.

## 1. Thách thức của Hệ thống phân tán

Trong Monolith, module A gọi module B là một lời gọi hàm (Function Call) trong cùng bộ nhớ RAM. Nhanh, tin cậy tuyệt đối.
Trong Microservices, Service A gọi Service B là một **Network Call** (HTTP/gRPC) qua mạng.
Mạng thì:
*   Chậm (Latency).
*   Không ổn định (Packet loss).
*   Có thể bị đứt cáp, Service B có thể sập.

Lập trình mạng cho Microservices là nghệ thuật xử lý sự không ổn định này. Chúng ta cần những công cụ đặc biệt.

## 2. Service Discovery: Danh bạ điện thoại của hệ thống

Giả sử bạn có 100 instance của Service "Product" chạy trên các IP và Port ngẫu nhiên (do Docker/Kubernetes cấp). Service "Order" làm sao biết IP nào để gọi?
Chúng ta cần một "Cuốn danh bạ động" - **Service Discovery**.

*   **Eureka (Spring Cloud):** Là server trung tâm.
    *   Khi Product Service khởi động, nó tự báo cáo: "Tôi là Product, IP 10.0.0.1, Port 8080".
    *   Eureka ghi nhớ vào danh sách.
    *   Khi Order Service cần gọi, nó hỏi Eureka: "Cho xin địa chỉ Product". Eureka trả về list, Order Service tự chọn 1 cái (Client-side Load Balancing).
*   **Consul / Etcd:** Các giải pháp thay thế phổ biến ngoài hệ sinh thái Java.

## 3. API Gateway: Người gác cổng

Chúng ta không thể để Client (Mobile/Web) gọi trực tiếp 100 cái microservices được. Quá rối rắm và rủi ro bảo mật.
**API Gateway** (như Spring Cloud Gateway, Nginx, Kong) là cửa ngõ duy nhất.
*   Client chỉ gọi `api.com/products`.
*   Gateway nhận request, phân tích, chuyển tiếp đến đúng Service Product nội bộ.
*   **Chức năng cộng thêm:** Xác thực (Auth), Giới hạn tốc độ (Rate Limiting), Log tập trung.

## 4. Giao tiếp đồng bộ vs Bất đồng bộ

### 4.1. Đồng bộ (Synchronous): OpenFeign (Java)
Service A chờ Service B trả lời xong mới chạy tiếp. Trong Spring Boot, **OpenFeign** giúp việc gọi API mạng trông "giả trân" như gọi hàm interface thông thường.

```java
// Khai báo: Đây là client để gọi Order Service
@FeignClient(name = "order-service")
public interface OrderClient {
    @GetMapping("/orders/{id}")
    OrderDTO getOrderById(@PathVariable("id") Long id);
}

// Sử dụng:
@Autowired OrderClient orderClient;
OrderDTO order = orderClient.getOrderById(123); // Nhìn như gọi hàm local, nhưng thực ra là HTTP Request
```

### 4.2. Bất đồng bộ (Asynchronous): Message Queue
Service A ném tin nhắn vào Queue rồi đi làm việc khác. Service B lúc nào rảnh thì lấy ra xử lý.
Giúp hệ thống **Decouple** (giảm phụ thuộc). Service B sập thì tin nhắn vẫn nằm trong Queue, không mất đi đâu cả. Service B sống lại sẽ xử lý tiếp.
**RabbitMQ** và **Apache Kafka** là hai cái tên sừng sỏ nhất.

**Ví dụ Node.js Producer (Gửi tin):**
```javascript
const amqp = require('amqplib');

async function sendEmailTask() {
    const conn = await amqp.connect('amqp://localhost');
    const channel = await conn.createChannel();
    const queue = 'email_tasks';

    await channel.assertQueue(queue, { durable: true });
    
    const msg = JSON.stringify({ to: "user@gmail.com", subject: "Welcome" });
    channel.sendToQueue(queue, Buffer.from(msg), { persistent: true });
    
    console.log(" [x] Sent '%s'", msg);
}
```

## 5. Circuit Breaker: Cầu dao tự ngắt

Điều gì xảy ra nếu Service Product bị lỗi và phản hồi cực chậm (30s)?
Service Gateway gọi Product -> Treo 30s.
Hàng ngàn user cùng gọi -> Hàng ngàn thread của Gateway bị treo -> Gateway sập -> Cả hệ thống sập (Cascading Failure).

**Circuit Breaker (Resilience4j)** giải quyết việc này:
*   Nó đếm lỗi. Nếu thấy Product lỗi liên tục 50% trong 10s.
*   Nó **Ngắt Cầu Dao (Open State)**.
*   Các request tiếp theo gọi Product sẽ bị chặn ngay lập tức (Fail Fast), trả về lỗi hoặc Default value, không cho gọi thật nữa để bảo vệ hệ thống.
*   Sau một lúc, nó hé mở (Half-open) để thăm dò xem Product sống lại chưa.

## 6. Kết luận: Java hay Node.js cho Microservices?

*   **Java (Spring Cloud/Netflix OSS):** Vô đối. Nó có một hệ sinh thái đồng bộ, hoàn chỉnh, "chuẩn sách giáo khoa" cho Microservices. Mọi ngân hàng, tập đoàn lớn đều tin dùng.
*   **Node.js (NestJS/Seneca):** Cực kỳ tốt để viết các service nhỏ, nhẹ, thiên về I/O. Thường được deploy dạng Docker Container trên Kubernetes. Node.js khởi động nhanh hơn Java nhiều, giúp Auto-scaling (Tự động tăng giảm số lượng server) hiệu quả hơn.

Xu hướng hiện đại là **Polyglot Microservices**: Kết hợp cả hai. Service Payment quan trọng dùng Java. Service Notification nhẹ nhàng dùng Node.js. Miễn là chúng nói chuyện chung ngôn ngữ HTTP/JSON hoặc gPRC, tất cả đều có thể sống hòa thuận.
