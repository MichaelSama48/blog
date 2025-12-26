---
title: "Xây dựng Web Server - Từ Java Servlet đến Node.js Express"
date: 2025-12-15
description: "Hiểu sâu về HTTP Request-Response. Cách dựng Web Server thủ công với Servlet và hiện đại với Spring Boot / Express.js."
cover:
    image: "/images/p2.jpg"
    alt: "Web Server Architecture"
    caption: "Spring Boot và Node.js: Hai thế lực Web Server."
tags: ["Java", "Spring Boot", "Node.js", "Express", "Servlet"]
categories: ["Web Development"]
---

## 1. Bản chất của Web: Request & Response

Dù bạn dùng công nghệ gì, giao tiếp Web luôn tuân theo HTTP:
1.  **Client (Browser/Mobile)** gửi **Request** (GET/POST, Headers, Body).
2.  **Web Server** xử lý logic.
3.  **Web Server** trả về **Response** (Status 200/404, Headers, Body là HTML/JSON).

## 2. Java: Sự tiến hóa từ Servlet đến Spring Boot

Ngày xưa, để làm Java Web, bạn phải cấu hình `web.xml` kinh hoàng và viết `HttpServlet`.

### Java Servlet (Cổ điển)

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        resp.setContentType("text/html");
        PrintWriter out = resp.getWriter();
        out.println("<h1>Xin chào từ Java Servlet!</h1>");
    }
}
```

Mỗi request được Servlet Container (Tomcat) gán cho một Thread. Servlet rất mạnh nhưng code dài dòng. Ngày nay, chúng ta dùng **Spring Boot**.

### Spring Boot (Hiện đại)

Spring Boot tự động nhúng sẵn Tomcat, tự cấu hình mọi thứ. Bạn chỉ cần tập trung vào Controller.

```java
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String sayHello() {
        return "Xin chào từ Spring Boot! Code siêu ngắn.";
    }
}
```

## 3. JavaScript: Web Server với Express.js

Nếu Java cần máy ảo JVM và Tomcat nặng nề, thì Node.js chỉ cần vài dòng code để dựng server HTTP. Framework phổ biến nhất là **Express.js**.

```javascript
const express = require('express');
const app = express();
const port = 3000;

// Middleware xử lý JSON
app.use(express.json());

// Định nghĩa Route
app.get('/hello', (req, res) => {
    res.send('Xin chào từ Node.js Express!');
});

app.post('/data', (req, res) => {
    const data = req.body;
    console.log("Nhận dữ liệu:", data);
    res.json({ status: "success", received: data });
});

app.listen(port, () => {
    console.log(`Server đang chạy tại http://localhost:${port}`);
});
```

## 4. So sánh Web Client: Khi Server gọi Server

Đôi khi Web Server này cần gọi API của Web Server kia. Chúng ta đóng vai trò là Web Client.

*   **Java**: Dùng `RestTemplate` (cũ) hoặc `WebClient` (mới - Reactive).
*   **JavaScript**: Dùng `fetch` (native) hoặc `axios` (thư viện mạnh hơn).

```javascript
// Node.js dùng Axios gọi API
const axios = require('axios');

async function getData() {
    try {
        const response = await axios.get('https://api.example.com/data');
        console.log(response.data);
    } catch (error) {
        console.error(error);
    }
}
```

## 5. Kết luận

*   **Java (Spring Boot)**: Phù hợp hệ thống Enterprise, cần cấu trúc chặt chẽ, Dependency Injection, bảo mật cao, giao dịch phức tạp.
*   **JavaScript (Express)**: Phù hợp Startup, làm MVP nhanh, hệ thống I/O nhiều, JSON native (vì cả DB là MongoDB và Client là JS đều dùng JSON).
