---
title: "Bảo mật trong lập trình mạng "
date: 2025-12-15
description: "Phân tích sâu cơ chế bắt tay SSL/TLS, giải phẫu cấu trúc JWT và các lỗ hổng, luồng hoạt động chuẩn của OAuth 2.0. Demo bảo mật với Spring Security và Node.js."
cover:
    image: "images/p5.jpg"
    alt: "Network Security Architecture"
    caption: "Bảo mật nhiều lớp: Từ đường truyền đến ứng dụng."
tags: ["Security", "JWT", "OAuth2", "Spring Security", "Node.js", "HTTPS", "Deep Dive"]
categories: ["Network Programming", "Cyber Security"]
---

Trong bốn chủ đề trước, chúng ta đã học cách xây dựng ứng dụng chạy nhanh, kết nối rộng. Nhưng một ứng dụng nhanh mà bảo mật kém thì đó là một thảm họa. Trong bài viết chuyên sâu này, chúng ta sẽ không nói những lời khuyên sáo rỗng như "hãy đặt mật khẩu mạnh". Chúng ta sẽ đi sâu vào kỹ thuật: Làm sao để mã hóa đường truyền? Làm sao để xác thực (Authentication) và phân quyền (Authorization) trong các hệ thống hiện đại?

## 1. HTTPS & SSL/TLS: Lá chắn đầu tiên

Ngày nay, trình duyệt sẽ cảnh báo "Not Secure" nếu website của bạn dùng HTTP thường. Tại sao? Vì HTTP gửi dữ liệu dạng **Clear Text**. Mật khẩu, thẻ tín dụng của bạn bay trên Internet như những tấm bưu thiếp, ai cũng đọc được.

### 1.1 SSL/TLS Handshake: Cuộc đàm phán bí mật
HTTPS hoạt động dựa trên cơ chế mã hóa bất đối xứng (Public/Private Key) để trao đổi khóa, sau đó dùng mã hóa đối xứng để truyền dữ liệu.
Quá trình này diễn ra như sau (đơn giản hóa):
1.  **Client Hello:** "Chào Server, tôi hỗ trợ chuẩn mã hóa TLS 1.3, Cipher Suite ABC."
2.  **Server Hello:** "Chào Client, oke dùng TLS 1.3 nhé. Đây là Chứng chỉ số (Certificate) và Public Key của tôi."
3.  **Authentication:** Client kiểm tra Chứng chỉ xem có phải do cơ quan uy tín (CA - như Let's Encrypt) cấp không? Nếu ok -> Tin tưởng.
4.  **Key Exchange:** Client sinh ra một chuỗi ngẫu nhiên (Session Key), mã hóa nó bằng Public Key của Server và gửi đi. Chỉ có Server (có Private Key) mới giải mã được.
5.  **Finished:** Kể từ bây giờ, cả hai dùng Session Key để mã hóa mọi dữ liệu trao đổi. Hacker bắt được gói tin cũng chỉ thấy một mớ rác nhị phân.

> **Lập trình viên cần làm gì?** Bạn không cần code thuật toán mã hóa. Nhiệm vụ của bạn là cài đặt chứng chỉ SSL cho Web Server (Nginx, Tomcat, hoặc cấu hình trong Node.js/Spring Boot).

## 2. JWT (JSON Web Token): Thẻ bài kỹ thuật số

Trong kiến trúc REST API và Microservices, chúng ta không dùng Session (lưu trên RAM server) nữa vì khó mở rộng (Scale). Chúng ta dùng **JWT**.

### 2.1. Giải phẫu JWT
Một chuỗi JWT gồm 3 phần tách nhau bởi dấu chấm `.`:
`CấuTrúcHeader.NộiDungPayload.ChữKýSignature`

*   **Header:** `{"alg": "HS256", "typ": "JWT"}`. Cho biết dùng thuật toán gì.
*   **Payload:** `{"sub": "123456", "name": "Hoang Anh", "role": "admin", "iat": 1516239022}`. Chứa dữ liệu user. **Lưu ý:** Phần này chỉ được mã hóa Base64 đơn giản, ai cũng decode ra đọc được. **Đừng bao giờ để mật khẩu ở đây!**
*   **Signature:** Đây là phần quan trọng nhất.
    `Sign = HMACSHA256(base64(Header) + "." + base64(Payload), SECRET_KEY)`
    Server giữ `SECRET_KEY`. Khi nhận Token, Server tính toán lại chữ ký. Nếu chữ ký không khớp -> Token giả mạo hoặc đã bị sửa đổi payload.

### 2.2. Lỗ hổng kinh điển: None Algorithm
Nếu Hacker sửa Header thành `{"alg": "none"}` và xóa phần Signature đi, một số thư viện cũ sẽ bỏ qua bước kiểm tra chữ ký và chấp nhận Token!
-> **Bài học:** Luôn update thư viện bảo mật và ép buộc kiểm tra thuật toán ký.

## 3. OAuth 2.0: Đừng bắt tôi nhập mật khẩu nữa!

OAuth 2.0 giải quyết bài toán: Cho phép App A (ví dụ: Game) truy cập dữ liệu của User trên App B (ví dụ: Google/Facebook) mà không cần User đưa mật khẩu Google cho Game.

### Luồng Authorization Code (Chuẩn nhất)
1.  **User** bấm "Login with Google".
2.  **App** chuyển hướng User sang trang login của **Google**.
3.  User đăng nhập thành công và bấm "Allow".
4.  **Google** chuyển hướng User quay lại **App** kèm theo một cái mã `code`.
5.  **App** (Server-side) âm thầm cầm `code` này gửi lên **Google API** để đổi lấy `Access Token`.
6.  **Google** trả về Token. App dùng Token này để lấy email, avatar của user.

Cách này an toàn vì `Access Token` không bao giờ lộ ra ở trình duyệt (Frontend), nó chỉ đi từ Server Google sang Server App.

## 4. Thực chiến: Spring Security vs Node.js Middleware

### 4.1. Java: Spring Security (Pháo đài kiên cố)
Đây là Framework bảo mật mạnh nhất (và phức tạp nhất). Nó hoạt động dựa trên một chuỗi các **Filter**.

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    @Autowired
    private JwtRequestFilter jwtRequestFilter;

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            // Tắt CSRF vì ta dùng API Stateless
            .csrf().disable()
            // Cấu hình quyền truy cập
            .authorizeRequests()
                .antMatchers("/api/auth/**").permitAll() // Ai cũng được vào Login/Register
                .antMatchers("/api/admin/**").hasRole("ADMIN") // Chỉ Admin
                .anyRequest().authenticated() // Còn lại phải có Token
            .and()
            // Quản lý Session: Không lưu trạng thái (Stateless)
            .sessionManagement().sessionCreationPolicy(SessionCreationPolicy.STATELESS);
        
        // Chèn Filter kiểm tra JWT vào trước Filter kiểm tra User/Pass mặc định
        http.addFilterBefore(jwtRequestFilter, UsernamePasswordAuthenticationFilter.class);
    }
}
```

### 4.2. Node.js: Express Middleware (Linh hoạt, gọn nhẹ)
Trong Node.js, bạn tự viết hoặc dùng thư viện. Triết lý là đơn giản, dễ hiểu.

```javascript
const jwt = require('jsonwebtoken');

// Middleware bảo vệ Route
const authMiddleware = (req, res, next) => {
    // 1. Lấy token từ Header: "Authorization: Bearer <token>"
    const authHeader = req.headers['authorization'];
    const token = authHeader && authHeader.split(' ')[1];

    if (!token) return res.status(401).json({ error: "Chưa đăng nhập!" });

    // 2. Xác thực Token
    jwt.verify(token, process.env.ACCESS_TOKEN_SECRET, (err, user) => {
        if (err) return res.status(403).json({ error: "Token hết hạn hoặc giả mạo" });
        
        // 3. Token ngon -> Gán info vào req để dùng ở bước sau
        req.user = user;
        next(); // Cho qua
    });
};

// Sử dụng
app.get('/api/protected', authMiddleware, (req, res) => {
    res.json({ message: "Chào " + req.user.username + ", đây là dữ liệu mật!" });
});
```

## 5. Kết luận

Bảo mật là một cuộc chạy đua vũ trang không hồi kết.
*   **Java Spring Security**: Cung cấp giải pháp toàn diện, cấu hình một lần dùng mãi mãi, phù hợp hệ thống lớn yêu cầu tuân thủ tiêu chuẩn khắt khe (Ngân hàng, Chính phủ).
*   **Node.js**: Cho phép bạn hiểu rõ từng dòng code bảo mật, linh hoạt tùy biến, nhưng rủi ro cao nếu bạn quên check một trường hợp nào đó.

Lời khuyên cuối cùng: **"Never roll your own crypto"**. Đừng tự viết thuật toán mã hóa hay logic xác thực. Hãy dùng các thư viện chuẩn đã được cộng đồng kiểm chứng (Bcrypt, JWT.io, Spring Security).
