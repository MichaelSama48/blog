---
title: "Lập trình mạng với API bên thứ ba "
date: 2025-12-26
description: "Chiến lược tích hợp API: Retry với Exponential Backoff, Rate Limiting, xác thực API Key/Bearer, và xử lý lỗi khi 'đứng trên vai người khổng lồ'."
cover:
    image: "/images/p7.jpg"
    alt: "API Integration Diagram"
    caption: "Tích hợp thế giới vào ứng dụng của bạn."
tags: ["API", "Integration", "Java", "JavaScript", "HttpClient", "Axios", "Deep Dive"]
categories: ["Network Programming", "Integration"]
---

Trong phát triển phần mềm hiện đại, quy tắc vàng là: **"Đừng phát minh lại cái bánh xe" (Don't reinvent the wheel)**.
*   Cần bản đồ? Gọi Google Maps API.
*   Cần thanh toán? Gọi Stripe/Momo API.
*   Cần AI? Gọi OpenAI/Gemini API.
*   Cần gửi Email? Gọi SendGrid API.

Kỹ năng "gọi API" nghe có vẻ đơn giản (chỉ là gửi một HTTP Request thôi mà?), nhưng để tích hợp **tin cậy** và **hiệu quả** trong môi trường Production là một câu chuyện hoàn toàn khác. Bài viết này sẽ chia sẻ những Design Pattern cao cấp khi làm việc với các dịch vụ bên thứ ba.

## 1. Các phương thức xác thực (Authentication)

Khi gọi API của người khác, bạn phải chứng minh mình là ai.

### 1.1. API Key
Cách đơn giản nhất. Nhà cung cấp đưa cho bạn một chuỗi ký tự dài ngoằng. Bạn gắn nó vào Query Param (`?api_key=xyz`) hoặc Header (`x-api-key: xyz`).
*   **Ưu điểm:** Dễ dùng.
*   **Nhược điểm:** Mất Key là mất tất cả. Không có cơ chế hết hạn hay phân quyền chi tiết.

### 1.2. Bearer Token (JWT / OAuth2)
Chuẩn mực hơn. Bạn gửi Header `Authorization: Bearer <token>`. Token này thường có hạn sử dụng (ví dụ 1 tiếng).
*   **Quy trình:**
    1.  Gọi API `/login` hoặc `/token` với ClientID/Secret để lấy Access Token.
    2.  Dùng Access Token để gọi các API lấy dữ liệu.
    3.  Khi Token hết hạn (lỗi 401), phải tự động gọi lại bước 1 để lấy Token mới (Refresh Token Flow). Việc này require code xử lý logic tự động (Interceptor).

## 2. Chiến lược xử lý lỗi: Retry & Exponential Backoff

Mạng Internet không hoàn hảo. API đối tác có thể sập trong 1 giây rồi sống lại. Nếu bạn fail ngay lập tức, trải nghiệm user sẽ tệ. Nhưng nếu bạn retry liên tục không nghỉ (`while(true)`), bạn sẽ làm sập server đối tác (DDoS) và bị khóa IP.

Giải pháp chuẩn là **Retry with Exponential Backoff** (Thử lại với thời gian chờ tăng theo lũy thừa).
*   Lần 1 lỗi -> Chờ 1s
*   Lần 2 lỗi -> Chờ 2s
*   Lần 3 lỗi -> Chờ 4s
*   Lần 4 lỗi -> Chờ 8s -> Bỏ cuộc.

### Node.js Implementation (Custom Utility)

```javascript
const wait = (ms) => new Promise(r => setTimeout(r, ms));

async function fetchWithRetry(url, options, retries = 3, delay = 1000) {
    try {
        return await axios(url, options); // Gọi API
    } catch (err) {
        if (retries === 0) throw err; // Hết lượt -> Lỗi luôn
        
        console.log(`Lỗi, thử lại sau ${delay}ms... (Còn ${retries} lượt)`);
        await wait(delay);
        
        // Đệ quy với thời gian chờ gấp đôi
        return fetchWithRetry(url, options, retries - 1, delay * 2);
    }
}
```

## 3. Rate Limiting: Tôn trọng luật chơi

Mọi API public đều có giới hạn (ví dụ: 100 request/phút). Nếu vượt quá, bạn nhận lỗi `429 Too Many Requests`.
Lập trình mạng chuyên nghiệp phải xử lý được cái này.
1.  **Đọc Header:** API thường trả về `X-RateLimit-Remaining: 5`.
2.  **Chủ động chậm lại:** Nếu thấy Remaining sắp về 0, hãy cho Thread ngủ (Sleep) hoặc đưa request vào hàng đợi (Queue) xử lý dần, đừng gửi dồn dập nữa.

## 4. Java 11 HttpClient: Vũ khí mới

Sau nhiều năm chịu đựng `HttpURLConnection` cổ lỗ sĩ, Java 11 đã giới thiệu `java.net.http.HttpClient` xịn xò, hỗ trợ HTTP/2 và Asynchronous.

```java
import java.net.http.*;
import java.net.URI;
import java.time.Duration;

public class ApiService {
    // Tạo Client dùng chung (Nên là biến Singleton)
    private static final HttpClient client = HttpClient.newBuilder()
            .version(HttpClient.Version.HTTP_2)
            .connectTimeout(Duration.ofSeconds(10))
            .build();

    public void callApiAsync() {
        HttpRequest request = HttpRequest.newBuilder()
                .uri(URI.create("https://api.github.com/users/MichaelSama48"))
                .header("Accept", "application/vnd.github+json")
                .GET()
                .build();

        // Gửi bất đồng bộ -> Không chặn luồng chính
        client.sendAsync(request, HttpResponse.BodyHandlers.ofString())
                .thenApply(HttpResponse::body)
                .thenAccept(body -> System.out.println("Kết quả về: " + body))
                .join(); // Chờ kết quả demo (thực tế không nên join)
    }
}
```
**Nhận xét:** Code Java hiện đại (Java 11/17/21) đã rất thanh thoát, hỗ trợ Fluent API và CompletableFuture, không thua kém gì JS Promise.

## 5. JavaScript: Axios Interceptors

Trong JS, **Axios** bá đạo nhờ tính năng Interceptors - cho phép bạn can thiệp vào mọi request/response đi ra đi vào.
Đây là nơi lý tưởng để:
*   Tự động gắn Token.
*   Tự động log lỗi.
*   Tự động chuyển đổi định dạng ngày tháng (Date Formatting).

```javascript
// Request Interceptor
axios.interceptors.request.use(config => {
    console.log(`Đang gọi API: ${config.url}`);
    config.headers['X-Custom-Header'] = 'MySecret';
    return config;
});

// Response Interceptor
axios.interceptors.response.use(
    response => response.data, // Chỉ lấy data, bỏ qua status, headers rườm rà
    error => {
        if (error.response && error.response.status === 401) {
            // Tự động Logout nếu Token hết hạn
            window.location.href = '/login';
        }
        return Promise.reject(error);
    }
);
```

## 6. Kết luận

Tích hợp API bên thứ ba giống như việc mời khách vào nhà. Bạn cần kiểm tra danh tính họ (Auth), chuẩn bị phương án nếu họ đến muộn (Timeout/Async), và xử lý khéo léo nếu họ từ chối phục vụ (Error Handling/Retry).
Một hệ thống tốt là hệ thống vẫn sống khỏe ngay cả khi các dịch vụ bên thứ ba mà nó phụ thuộc gặp sự cố (bằng cách dùng Default Value hoặc Cache). Đó là đỉnh cao của sự bền vững (Resiliency).
