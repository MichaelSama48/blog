---
title: "Lập trình mạng không đồng bộ "
date: 2025-12-26
description: "So sánh chi tiết Java Memory Model (Multithreading) và Node.js Event Loop. Cách sử dụng CompletableFuture và Promise/Async/Await hiệu quả."
cover:
    image: "images/p8.jpg"
    alt: "Async Processing Diagram"
    caption: "Bất đồng bộ giúp CPU không bao giờ phải chờ đợi."
tags: ["Async", "Multithreading", "Event Loop", "Promise", "CompletableFuture", "Deep Dive"]
categories: ["Network Programming", "Performance"]
---


Trong lập trình mạng, I/O (Input/Output) là tác vụ chậm nhất. Đọc một file từ ổ cứng mất vài mili-giây, nhưng tải một file từ mạng có thể mất cả giây. Nếu CPU (thứ xử lý được hàng tỷ lệnh mỗi giây) phải ngồi chơi xơi nước chờ mạng tải xong, đó là sự lãng phí tài nguyên khủng khiếp.

Chủ đề này sẽ mổ xẻ hai triết lý giải quyết vấn đề "chờ đợi" (Blocking vs Non-blocking) của hai thế lực: **Java (Multithreading)** và **JavaScript (Event Loop)**.

## 1. Java: Sức mạnh của Đa luồng (Multithreading)

Java chọn cách tiếp cận "lấy thịt đè người". Nếu có 1 công việc cần chờ, tôi sẽ để 1 nhân viên (Thread) ngồi chờ việc đó, còn tôi đi thuê nhân viên khác làm việc khác.

### 1.1. Java Thread & Thread Pool
Tạo Thread trong Java rất dễ, nhưng quản lý nó mới khó. Mỗi Thread hệ điều hành tốn khoảng 1MB bộ nhớ Stack.
Nếu server phải xử lý 10.000 request cùng lúc -> Cần 10.000 Threads -> Tốn 10GB RAM!
Do đó, Java dùng **Thread Pool** (Hồ chứa các luồng) để tái sử dụng.
Ví dụ: Có 10 nhân viên cố định. Có 100 job thì 10 nhân viên làm trước, 90 job còn lại xếp hàng vào Queue.

### 1.2. CompletableFuture: Async hiện đại trong Java
Từ Java 8, `CompletableFuture` ra đời giúp viết code Async theo phong cách "Promise" của JS, tránh tình trạng "Callback Hell" (các hàm lồng nhau quá sâu).

```java
import java.util.concurrent.CompletableFuture;

public class AsyncJava {
    public static void main(String[] args) {
        System.out.println("Main Thread bắt đầu làm việc A...");

        // Giao việc B (nặng) cho một Thread khác chạy ngầm
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            try { Thread.sleep(2000); } catch (InterruptedException e) {} // Giả vờ tải nặng
            return "Dữ liệu tải xong";
        });

        // Khi nào việc B xong thì làm tiếp việc C (Non-blocking)
        future.thenAccept(result -> {
            System.out.println("Nhận được: " + result);
            System.out.println("Xử lý tiếp việc C...");
        });

        System.out.println("Main Thread vẫn làm việc D trong lúc chờ B...");
        
        // Giữ main thread sống để chờ kết quả (demo only)
        try { Thread.sleep(3000); } catch (Exception e) {}
    }
}
```

## 2. JavaScript: Ảo thuật gia Event Loop

JavaScript chọn con đường khác. Nó chỉ có **DUY NHẤT 1 LUỒNG** (Single Thread).
Thoạt nghe có vẻ yếu đuối. Làm sao 1 người làm được nhiều việc cùng lúc?
Bí quyết là: JS là một "người quản lý" (Manager) chứ không phải "công nhân" (Worker).
Khi gặp tác vụ nặng (đọc file, gọi mạng), JS Manager không tự làm. Nó ký giấy giao việc cho Hệ điều hành (Kernel) hoặc C++ API bên dưới làm, dặn "Làm xong thì báo tôi". Sau đó JS đi làm việc khác ngay lập tức.
Cơ chế "nhận báo cáo" đó gọi là **Event Loop** (Vòng lặp sự kiện).

### 2.1. Các pha của Event Loop
1.  **Microtasks Queue:** Chứa các `Promise.then`, `process.nextTick`. (Ưu tiên cao nhất, làm hết mới thôi).
2.  **Timers:** `setTimeout`, `setInterval`.
3.  **Poll I/O:** Chờ sự kiện mạng/file từ OS.
4.  **Check:** `setImmediate`.

### 2.2. Promise & Async/Await: Cú pháp "ngọt ngào" (Syntactic Sugar)
Dù viết code trông có vẻ tuần tự (nhờ `await`), nhưng bản chất bên dưới vẫn là bất đồng bộ.

```javascript
const fs = require('fs').promises;

async function readFileAsync() {
    console.log("1. Bắt đầu");

    // Lệnh này không chặn luồng chính!
    // Nó ném việc đọc file cho OS và đăng ký một sự kiện "khi nào xong thì quay lại đây"
    try {
        const data = await fs.readFile('big-file.txt', 'utf-8');
        console.log("3. Đọc xong, kích thước: " + data.length);
    } catch (err) {
        console.error("Lỗi:", err);
    }

    console.log("2. Dòng này sẽ chạy TRƯỚC khi đọc file xong");
}

readFileAsync();
```

## 3. So sánh hiệu năng và Ứng dụng

| Đặc điểm | Java (Thread-based) | Node.js (Event-loop) |
| :--- | :--- | :--- |
| **Sức mạnh CPU** | Tốt. Có thể dùng nhiều Core CPU cùng lúc để tính toán ma trận, xử lý ảnh. | Yếu. Nếu 1 request tính toán lâu, nó chặn luôn Loop, cả server treo. |
| **Sức mạnh I/O** | Khá. Tốn RAM nếu nhiều kết nối. (Trừ khi dùng Java NIO/Netty). | Tuyệt vời. Xử lý hàng chục ngàn kết nối I/O với cực ít RAM. |
| **Độ khó** | Khó. Phải lo về Race Condition, Deadlock, synchronize dữ liệu giữa các luồng. | Dễ hơn. Không cần lo lock dữ liệu (vì chỉ có 1 luồng). |

## 4. Dự án nào chọn cái nào?

*   **Chọn Node.js khi:** Làm Real-time Chat, Streaming Server, API Gateway, Dashboard (nơi chủ yếu là đẩy dữ liệu qua lại, ít tính toán).
*   **Chọn Java khi:** Làm hệ thống xử lý giao dịch ngân hàng, Xử lý ảnh/video, AI, các tác vụ cần tính toán logic phức tạp và độ chính xác cao/đa luồng song song thực sự.

## 5. Kết luận

Lập trình mạng chuyên sâu là cuộc chơi của việc quản lý tài nguyên.
Java dạy bạn cách quản lý đội quân hùng hậu (Threads).
JavaScript dạy bạn cách làm việc thông minh, ủy quyền (Non-blocking I/O).
Cả hai mô hình đều đáng học và bổ trợ cho nhau. Hiểu sâu về Event Loop của JS sẽ giúp bạn hiểu rõ hơn về Reactive Programming trong Java và ngược lại.
