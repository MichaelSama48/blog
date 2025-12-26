---
title: "RESTful API - Ngôn ngữ chung của Internet"
date: 2025-12-15
description: "Đi sâu vào HATEOAS, Mô hình trưởng thành Richardson, Tính Idempotency của HTTP và cách xử lý lỗi chuẩn trong Spring Boot vs Express."
cover:
    image: "images/p3.jpg"
    alt: "REST API Design Diagram"
    caption: "API là cây cầu kết nối các hệ thống độc lập."
tags: ["REST", "API", "JSON", "Spring", "Express", "Deep Dive"]
categories: ["Network Programming", "API Design"]
---

Trong kỷ nguyên kết nối, không ưng dụng nào là một hòn đảo. Mobile App cần nói chuyện với Server, Server cần nói chuyện với Database, và Microservices cần nói chuyện với nhau. **RESTful API** (Representational State Transfer) chính là ngôn ngữ chung (lingua franca) giúp các hệ thống khác biệt về công nghệ có thể hiểu nhau.

Bài viết này không chỉ dạy bạn cách viết `@GetMapping`. Chúng ta sẽ đi sâu vào triết lý thiết kế REST, những nguyên tắc ngầm (Best Practices) và cách hiện thực hóa chúng một cách chuyên nghiệp nhất trên Java và JavaScript.

## 1. Triết lý REST & Mô hình Richardson

REST không phải là luật (Protocol), nó là một bộ quy tắc hướng dẫn (Guideline). Roy Fielding (cha đẻ của REST) đã định nghĩa nó trong luận văn tiến sĩ của mình năm 2000. Để đánh giá một API có "chuẩn REST" hay không, Leonard Richardson đưa ra mô hình 4 cấp độ:

*   **Level 0 (The Swamp of POX):** Dùng HTTP làm đường ống vận chuyển, nhưng chỉ dùng 1 method (thường là POST) cho tất cả. Ví dụ: SOAP XML ngày xưa.
*   **Level 1 (Resources):** Biết chia nhỏ API thành các Tài nguyên. Thay vì `/submitQuery`, ta có `/products`, `/orders`.
*   **Level 2 (HTTP Verbs):** Biết dùng đúng động từ. GET để lấy, POST để tạo, DELETE để xóa. Đây là mức đa số các API hiện nay đạt được.
*   **Level 3 (Hypermedia Controls - HATEOAS):** Mức cao nhất. API trả về kèm theo các đường dẫn (Links) gợi ý client có thể làm gì tiếp theo.

## 2. Ngữ nghĩa HTTP: Đừng dùng sai động từ!

Một sai lầm kinh điển của sinh viên: Dùng `GET` để xóa dữ liệu vì... tiện.
Trong lập trình mạng, việc tuân thủ **Idempotency** (Tính lũy đẳng) là cực kỳ quan trọng.

| Method | Ý nghĩa | Idempotent? | Giải thích |
| :--- | :--- | :--- | :--- |
| **GET** | Lấy dữ liệu | **CÓ** | Gọi 1 lần hay 100 lần thì dữ liệu trên server không đổi (Safe). |
| **POST** | Tạo mới | **KHÔNG** | Gọi 2 lần tạo ra 2 bản ghi khác nhau. |
| **PUT** | Cập nhật (ghi đè toàn bộ) | **CÓ** | Gọi 10 lần với cùng nội dung thì kết quả cuối cùng vẫn như lần 1. |
| **PATCH** | Cập nhật (một phần) | **KHÔNG** | (Thường không Idempotent tùy cách implement). |
| **DELETE** | Xóa | **CÓ** | Xóa lần 1 thì mất. Xóa lần 2 (vào cái đã mất) thì trạng thái hệ thống vẫn vậy (vẫn là không còn bản ghi đó). |

Hiểu bảng này giúp bạn thiết kế API chịu lỗi tốt. Ví dụ: Nếu mạng lag, Client lỡ gửi lệnh `PUT` 2 lần, bạn yên tâm là hệ thống không bị sai lệch. Nhưng nếu lỡ gửi `POST` 2 lần, bạn có thể bị trừ tiền 2 lần!

## 3. Xử lý lỗi (Error Handling): Nghệ thuật của Status Code

Đừng bao giờ trả về HTTP 200 OK kèm theo body `{"error": "Lỗi rồi"}`. Đó là một tội ác!
Hãy dùng đúng HTTP Status Code để Client (Frontend) biết phải làm gì:

*   **2xx (Thành công):** 200 OK, 201 Created (khi tạo xong), 204 No Content (khi xóa xong).
*   **4xx (Lỗi Client):** 400 Bad Request (tham số sai), 401 Unauthorized (chưa đăng nhập), 403 Forbidden (không có quyền), 404 Not Found.
*   **5xx (Lỗi Server):** 500 Internal Server Error (code bị bug, Db sập).

### 3.1. Spring Boot: Global Exception Handler

Spring Boot cung cấp cơ chế `@ControllerAdvice` tuyệt vời để bắt lỗi tại một nơi duy nhất, giúp code Controller sạch đẹp.

```java
// Tạo một class quản lý lỗi chung toàn hệ thống
@ControllerAdvice
public class GlobalExceptionHandler {

    // Bắt lỗi khi không tìm thấy tài nguyên (ResourceNotFoundException tự định nghĩa)
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ApiError> handleNotFound(ResourceNotFoundException ex) {
        ApiError error = new ApiError(
            HttpStatus.NOT_FOUND.value(), 
            "Không tìm thấy dữ liệu", 
            ex.getMessage()
        );
        return new ResponseEntity<>(error, HttpStatus.NOT_FOUND);
    }

    // Bắt lỗi Validate dữ liệu (Ví dụ: Email không đúng định dạng)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<Map<String, String>> handleValidationExceptions(MethodArgumentNotValidException ex) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getAllErrors().forEach((error) -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return new ResponseEntity<>(errors, HttpStatus.BAD_REQUEST);
    }
}
```

### 3.2. Express.js: Error Middleware

Trong Express, Middleware xử lý lỗi là function có 4 tham số `(err, req, res, next)`.

```javascript
// Middleware xử lý lỗi cuối cùng trong chuỗi
const errorHandler = (err, req, res, next) => {
    console.error(err.stack); // Log lỗi ra console server

    // Phân loại lỗi
    if (err.name === 'ValidationError') {
        return res.status(400).json({ 
            status: 'fail',
            message: 'Dữ liệu không hợp lệ',
            errors: err.details 
        });
    }

    if (err.name === 'UnauthorizedError') {
        return res.status(401).json({ message: 'Token không hợp lệ' });
    }

    // Lỗi không xác định (500)
    res.status(500).json({ 
        status: 'error', 
        message: 'Lỗi server nội bộ, vui lòng thử lại sau.' 
    });
};

app.use(errorHandler); // Đăng ký ở cuối file app.js
```

## 4. Phân trang và Lọc (Pagination & Filtering)

Một API chuyên nghiệp không bao giờ trả về `SELECT * ALL`. Nếu bảng có 1 triệu dòng, API đó sẽ giết chết Server và Client.
REST API chuẩn thường có dạng:
`GET /products?page=1&limit=10&category=laptop&sort=-price`

### Spring Data JPA (Magic!)
Spring hỗ trợ việc này tận răng với `Pageable`.

```java
@GetMapping
public Page<Product> getProducts(
    @RequestParam(defaultValue = "0") int page,
    @RequestParam(defaultValue = "10") int size,
    @RequestParam(defaultValue = "id") String sortBy
) {
    Pageable pageable = PageRequest.of(page, size, Sort.by(sortBy));
    return productRepository.findAll(pageable); // Tự động sinh câu SQL LIMIT OFFSET
}
```

### Node.js (Thủ công hoặc thư viện)
Bạn phải tự tính toán `skip` và `limit` cho MongoDB hoặc SQL.

```javascript
router.get('/', async (req, res) => {
    const page = parseInt(req.query.page) || 1;
    const limit = parseInt(req.query.limit) || 10;
    const skip = (page - 1) * limit;

    const products = await ProductModel.find()
        .skip(skip)
        .limit(limit)
        .sort({ createdAt: -1 });

    const total = await ProductModel.countDocuments();

    res.json({
        data: products,
        meta: {
            currentPage: page,
            totalPages: Math.ceil(total / limit),
            totalItems: total
        }
    });
});
```

## 5. Kết luận

Viết API thì dễ, viết API **tốt** mới khó. Một API tốt phải:
1.  Dễ hiểu (URL và Method có nghĩa).
2.  Khó dùng sai (Validate chặt chẽ, Status code chuẩn).
3.  Hiệu năng cao (Phân trang, Caching).

Java Spring Boot cung cấp một khung sườn cứng để ép bạn làm chuẩn ngay từ đầu. Node.js Express cho bạn sự tự do, nhưng đòi hỏi bạn phải có kỷ luật và kiến thức để tự xây dựng cấu trúc chuẩn. Dù chọn công nghệ nào, hãy luôn đặt trải nghiệm của người dùng API (Developer khác) lên hàng đầu.
