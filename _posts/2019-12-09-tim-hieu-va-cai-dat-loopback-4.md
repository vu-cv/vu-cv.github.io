---
layout: article
title: Tìm hiểu & Cài đặt Loopback 4
tags: [Tìm hiểu loopback, Cài đặt Loopback, Loopback 4]
---
Bạn đang cần tìm một framework để thiết kế API thì Loopback 4 là một sự lựa chọn phù hợp cho bạn
## 1. Giới thiệu
Loopback 4 này là một framework mã nguồn mở dựa trên NodeJs và Express. Nó cho phép bạn nhanh chóng tạo ra APIs và microservices từ phía Backend như là SOAP hoặc REST services

Ưu điểm mà ta có thể nhận thấy ngay là khả năng tạo ra các controller, model, repository, entity một cách nhanh chóng và Loopback 4 hỗ trợ kết nối với nhiều cơ sở dữ liệu trong một project.

Dưới đây là sơ đồ minh họa Loopback làm cầu nối giữa các incoming requests và outgoing integrations

![ảnh minh họa incoming requests và outgoing integrations](/assets/images/lb4-high-level.png "Logo Title Text 1")

Đây là một framework rất hay, các bạn có thể tham khảo chi tiết tại địa chỉ: [https://loopback.io](https://loopback.io)

## 2. Cài đặt
### 2.1 Điều kiện tiên quyết
Cài đặt [NodeJs](https://nodejs.org/en/download/){:target="_blank"} (version 8.9 trở lên) nếu nó chưa được cài đặt trên máy của bạn

### 2.2 Cài đặt LoopBack 4 CLI
```npm
npm i -g @loopback/cli
```