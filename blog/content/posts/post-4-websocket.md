---
title: "WebSocket - Giao tiếp thời gian thực"
date: 2025-12-15
description: "Phân tích sâu giao thức WebSocket Handshake. Chiến lược Scaling WebSocket với Redis Pub/Sub. Các bài toán thực tế: Chat, Notification, Live Tracking."
cover:
    image: "/images/p4.jpg"
    alt: "Real-time WebSocket Connection Diagram"
    caption: "WebSocket duy trì kết nối bền vững hai chiều (Full-duplex)."
tags: ["WebSocket", "Real-time", "Node.js", "Java", "Socket.io", "Deep Dive"]
categories: ["Network Programming", "System Design"]
---

Chúng ta đã nói về HTTP và REST. Chúng tuyệt vời cho việc duyệt web, đọc báo. Nhưng nếu bạn muốn làm một ứng dụng **Chat**, một Game Online, hay bảng theo dõi chứng khoán nhảy múa liên tục? HTTP trở nên bất lực. Nó quá chậm chạp và thụ động.

Chào mừng bạn đến với thế giới của thiết kế mạng hướng sự kiện (Event-driven Networking) với **WebSocket**. Bài viết này sẽ không lặp lại những gì cơ bản, chúng ta sẽ đi sâu vào "nội thất" của giao thức này, cách nó bắt tay, cách nó giữ kết nối sống (Keep-alive) và làm sao để mở rộng (Scale) nó lên hàng triệu người dùng.

## 1. Tại sao HTTP lại "tệ" cho Real-time?

Hãy tưởng tượng bạn đang đợi một lá thư quan trọng.
*   **Polling (Thăm dò):** Cứ 5 phút bạn chạy ra hòm thư mở ra xem. 99 lần mở là hòm thư rỗng. Bạn vừa tốn công, vừa mệt mỏi. Đây là cách HTTP hoạt động cũ.
*   **Long Polling:** Bạn ra hòm thư và... ngồi lì ở đó đợi. Bác đưa thư đến thì đưa cho bạn. Bạn nhận thư xong, chạy về nhà cất, rồi lại chạy ra hòm thư ngồi đợi tiếp. Khá hơn chút, nhưng vẫn tốn thời gian di chuyển (Round trip).
*   **WebSocket:** Bác đưa thư nối một đường ống thẳng vào bàn làm việc của bạn. Có thư bác thả vào ống, thư rơi bộp xuống bàn bạn ngay lập tức. Không cần đi lại, không cần chờ đợi.

Về mặt kỹ thuật, HTTP mỗi lần gửi request phải mang theo một đống **Header** (Cookie, User-Agent...) nặng nề (vài KB), trong khi gói tin WebSocket (Frame) chỉ tốn vài Bytes header. Độ trễ (Latency) của WebSocket thấp hơn HTTP Polling hành chục lần.

## 2. Giải phẫu WebSocket Handshake (Cái bắt tay)

WebSocket không chạy trên một cổng riêng lạ lẫm nào cả. Nó thông minh lách qua cổng 80/443 của Web truyền thống để tránh bị Tường lửa chặn. Nó bắt đầu cuộc đời bằng một HTTP Request bình thường, nhưng với một "âm mưu" nâng cấp.

**Client gửi:**
```http
GET /chat HTTP/1.1
Host: server.example.com
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==
Sec-WebSocket-Version: 13
```

*   `Upgrade: websocket`: "Này Server, mình đổi sang nói chuyện bằng WebSocket nhé?"
*   `Sec-WebSocket-Key`: Một chuỗi ngẫu nhiên để đảm bảo bảo mật.

**Server đáp trả (nếu đồng ý):**
```http
HTTP/1.1 101 Switching Protocols
Upgrade: websocket
Connection: Upgrade
Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=
```

*   `101 Switching Protocols`: "Ok, duyệt! Chuyển giao thức thôi."

Sau giây phút này, kết nối HTTP chấm dứt. Đường ống TCP bên dưới được giữ nguyên và chuyển sang chế độ truyền tin nhị phân (Binary) của WebSocket. Kênh truyền 2 chiều (Full-duplex) chính thức mở ra.

## 3. Tech Stack: Node.js vs Java Spring

### 3.1. Node.js + Socket.io (Sự lựa chọn phổ biến nhất)
Node.js sinh ra là cho Real-time. Event Loop của nó xử lý hàng ngàn kết nối I/O nhẹ nhàng như lông hồng. Thư viện `socket.io` là tiêu chuẩn ngành vì nó có cơ chế **Fallback**.

*Nếu mạng công ty chặn WebSocket? Socket.io tự động chuyển về Long Polling. Khi mạng tốt, nó tự nâng cấp lên lại WebSocket. Quá vi diệu!*

**Server Code (Node.js):**
```javascript
const io = require('socket.io')(3000, {
    cors: { origin: "*" }
});

// Middleware xác thực (như JWT)
io.use((socket, next) => {
    const token = socket.handshake.auth.token;
    if (isValid(token)) next();
    else next(new Error("Không có quyền!"));
});

io.on('connection', (socket) => {
    console.log(`User ${socket.id} đã vào phòng chat`);

    // Tham gia vào một "Phòng" (Room) cụ thể
    socket.join("phong-it-support");

    socket.on('send-message', (data) => {
        // Chỉ gửi cho những người trong phòng này
        io.to("phong-it-support").emit('new-message', {
            user: socket.id,
            msg: data
        });
    });

    socket.on('disconnect', () => {
        console.log("User đã thoát");
    });
});
```

### 3.2. Java Spring Boot + STOMP (Sự lựa chọn Enterprise)
Java thuần (JSR 356) chỉ hỗ trợ WebSocket thô (gửi text/binary). Trong dự án lớn, ta cần một giao thức định dạng tin nhắn rõ ràng hơn. Spring hỗ trợ **STOMP** (Simple Text Oriented Messaging Protocol) chạy trên nền WebSocket. Nó giúp định tuyến tin nhắn (Routing) dễ như viết MVC Controller.

**Server Code (Java Spring):**
```java
@Controller
public class ChatController {

    // Client gửi tới /app/chat
    @MessageMapping("/chat")
    // Server tự động forward tới tất cả ai đang Subscribe /topic/messages
    @SendTo("/topic/messages")
    public ChatMessage sendMessage(ChatMessage message) {
        return new ChatMessage("Server", "Đã nhận: " + message.getContent());
    }
}
```
**Nhận xét:** Spring đóng gói quá kỹ, giúp code gọn nhưng bạn sẽ khó hiểu bên dưới nó chạy thế nào so với Node.js.

## 4. Bài toán Scaling: Khi 1 Server là không đủ

Đây là câu hỏi phỏng vấn Senior: *"Nếu Server A đang giữ kết nối với User 1, Server B giữ kết nối với User 2. User 1 muốn chat với User 2 thì làm sao?"*
Mặc định, User 1 gửi tin lên Server A, Server A ngơ ngác vì không biết User 2 là ai (User 2 đang ở Server B mà).

**Giải pháp: Redis Pub/Sub (Adapter)**
Chúng ta cần một "cái loa chung" ở giữa (Message Broker).
1.  User 1 gửi tin cho Server A.
2.  Server A không tìm thấy User 2, nó "hét" vào Redis: "Có tin cho User 2 nè!".
3.  Tất cả Server (bao gồm Server B) đều đang nghe Redis. Server B nghe thấy, kiểm tra danh sách khách của mình: "À, User 2 đang ở đây".
4.  Server B đẩy tin nhắn xuống cho User 2.

Trong `socket.io`, cấu hình cái này chỉ mất 2 dòng code với `socket.io-redis-adapter`.

## 5. Kết luận

WebSocket là một bước tiến hóa tất yếu của Web.
*   **Dùng Node.js (Socket.io)**: Nếu bạn cần build nhanh, tính năng linh hoạt, hỗ trợ trình duyệt cũ tốt, cộng đồng support cực lớn. (Khuyên dùng cho đồ án/startup).
*   **Dùng Java (Spring WebSocket)**: Nếu hệ thống backend của bạn 100% là Java Microservices và bạn muốn tận dụng Security, Transaction của Spring.

Dù dùng cái nào, hãy nhớ quản lý tài nguyên cẩn thận. Một kết nối WebSocket tốn RAM server liên tục (khác với HTTP request xong là giải phóng). 1 triệu kết nối là bài toán không đơn giản đâu nhé!
