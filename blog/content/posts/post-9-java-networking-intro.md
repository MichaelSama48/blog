---
title: "Java Networking: Mạng máy tính trong Java là gì?"
date: 2025-12-26
description: "Tổng quan về Networking trong Java. Khái niệm cơ bản về IP, Port, Protocol, và gói java.net. Nền tảng cho mọi ứng dụng phân tán."
cover:
    image: "/images/p9.jpg"
    alt: "Java Networking Concepts"
    caption: "Mô hình mạng trong Java."
tags: ["Java", "Networking", "Theory", "Basic"]
categories: ["Network Programming"]
---

Trả lời câu hỏi căn bản nhất: **Java Networking thực chất là gì?** Nó cung cấp những công cụ gì cho lập trình viên?

Bài viết này sẽ tổng hợp lại bức tranh toàn cảnh về khả năng xử lý mạng của ngôn ngữ Java (J2SE).

## 1. Networking là gì?

Networking (Kết nối mạng) là hành động kết nối hai hoặc nhiều thiết bị máy tính lại với nhau để chia sẻ tài nguyên (như máy in, file) hoặc trao đổi dữ liệu.
Trong Java, Networking chủ yếu đề cập đến việc viết các chương trình chạy trên nhiều máy tính và có thể giao tiếp với nhau thông qua giao thức TCP/IP.

Java là ngôn ngữ hàng đầu cho lập trình mạng nhờ bộ thư viện `java.net` mạnh mẽ, cung cấp một lớp trừu tượng (abstraction) giúp bạn không cần quan tâm đến các chi tiết phức tạp của tầng vật lý hay tầng liên kết dữ liệu.

## 2. Các thuật ngữ cốt lõi

Trước khi code, bạn phải nắm vững danh sách từ vựng này:

### 2.1. IP Address (Địa chỉ IP)
Là định danh duy nhất của một thiết bị trên mạng.
*   IPv4: `192.168.1.1` (phổ biến nhất).
*   IPv6: `2001:0db8:...` (tương lai).
Trong Java, IP được đại diện bởi class `java.net.InetAddress`.

### 2.2. Protocol (Giao thức)
Là bộ quy tắc để các máy tính "hiểu" nhau.
*   **TCP**: Tin cậy, nối kết (Connection-oriented).
*   **UDP**: Nhanh, không nối kết (Connection-less).
*   **HTTP/FTP/SMTP**: Các giao thức tầng ứng dụng chạy trên nền TCP.

### 2.3. Port Number (Số hiệu cổng)
Nếu IP là địa chỉ nhà, thì Port là số phòng. Một máy tính có thể chạy nhiều ứng dụng mạng (Web, Game, Mail). Port giúp OS biết gói tin này thuộc về ứng dụng nào.
*   0 - 1023: Cổng hệ thống (Well-known ports). VD: 80 (Web), 443 (HTTPS), 22 (SSH).
*   1024 - 65535: Cổng người dùng (User ports). Ứng dụng của chúng ta thường chạy ở đây (VD: 8080, 3000).

### 2.4. MAC Address
Địa chỉ vật lý của card mạng (NIC). Java có thể lấy được thông tin này, nhưng trong lập trình ứng dụng, chúng ta ít khi dùng đến nó vì IP quan trọng hơn.

### 2.5. Socket
Là điểm cuối (EndPoint) của một liên kết hai chiều.
`Socket = IP Address + Port Number`.

## 3. Gói `java.net` hỗ trợ những gì?

Java chia hỗ trợ mạng thành 2 mức độ:

### Mức thấp (Low-Level)
Cho phép bạn kiểm soát từng byte gửi đi.
*   **Socket / ServerSocket**: Dùng cho giao thức TCP. (Xem lại Bài 1).
*   **DatagramSocket / DatagramPacket**: Dùng cho giao thức UDP. (Xem lại Bài 1).
*   **MulticastSocket**: Dùng để gửi tin cho một nhóm (Multicasting).

### Mức cao (High-Level)
Cho phép bạn làm việc với Web dễ dàng hơn.
*   **URL (Uniform Resource Locator)**: Đại diện cho một tài nguyên trên Web.
*   **URLConnection / HttpURLConnection**: Giúp gửi GET/POST request mà không cần tự xây dựng gói tin HTTP thủ công.
*   **JarURLConnection**: Đọc dữ liệu từ file .jar qua mạng.

## 4. Ví dụ: Lớp `InetAddress`

Đây là lớp cơ bản nhất, thường dùng để phân giải tên miền (DNS Lookup).

```java
import java.net.InetAddress;

public class NetworkTest {
    public static void main(String[] args) {
        try {
            // Lấy IP của google.com
            InetAddress ip = InetAddress.getByName("www.google.com");
            
            System.out.println("Host Name: " + ip.getHostName());
            System.out.println("IP Address: " + ip.getHostAddress());
            
            // Lấy IP của máy mình (Localhost)
            InetAddress local = InetAddress.getLocalHost();
            System.out.println("My IP: " + local.getHostAddress());
            
        } catch (Exception e) {
            System.out.println(e);
        }
    }
}
```

**Kết quả:**
```
Host Name: www.google.com
IP Address: 142.250.191.68
My IP: 192.168.1.50
```

## 5. Ví dụ: Lớp `URL`

Lớp này giúp phân tách một đường link ra thành từng phần nhỏ.

```java
import java.net.URL;

public class UrlDemo {
    public static void main(String[] args) {
        try {
            URL url = new URL("https://www.example.com:8080/pages/index.html?q=java#home");
            
            System.out.println("Protocol: " + url.getProtocol()); // https
            System.out.println("Host: " + url.getHost());         // www.example.com
            System.out.println("Port: " + url.getPort());         // 8080
            System.out.println("Path: " + url.getPath());         // /pages/index.html
            System.out.println("File: " + url.getFile());         // /pages/index.html?q=java
            
        } catch (Exception e) {
            System.out.println(e);
        }
    }
}
```

## 6. Tại sao chọn Java để làm Networking?

1.  **Đa nền tảng (Write Once, Run Anywhere)**: Code Socket trên Windows đem sang Linux chạy y hệt, không cần sửa đổi (khác với C++ phải dùng thư viện riêng cho từng OS).
2.  **Đa luồng (Multithreading)**: Java hỗ trợ Thread từ trong nhân, giúp xử lý hàng nghìn kết nối đồng thời dễ dàng.
3.  **Bảo mật**: Thư viện SSL/TLS tích hợp sẵn, mô hình Security Manager chặt chẽ.
4.  **Hệ sinh thái phong phú**: Từ `java.net` cơ bản đến `NIO` (Non-blocking I/O) hiệu năng cao, và các framework khổng lồ như Spring Boot, Netty.

## 7. Kết luận

Networking trong Java không chỉ là Socket. Nó là một tập hợp các công cụ từ thấp đến cao giúp bạn xây dựng mọi thứ, từ một ứng dụng Chat đơn giản đến một hệ thống Ngân hàng phân tán phức tạp.

Nắm vững `java.net` là bước đầu tiên để bạn trở thành một Backend Developer thực thụ. Hy vọng series 9 bài viết này đã cung cấp cho bạn đủ hành trang để tự tin bước đi trên con đường này. Hẹn gặp lại các bạn trong những series tiếp theo!
