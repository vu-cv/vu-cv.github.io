---
layout: article
title: React + Redux - User SignUp and SignIn & Example 
tags: [React, ReactJs, Redux, User LogIn, Login]
---

Trong bài này mình sử dụng **React 16.9**


Trong hướng dẫn này, mình sẽ giới thiệu và xây dựng chức năng đăng ký và đăng nhập cho người dùng với React và Redux. Ví dụ này sử dụng JWT để authentication. Nó dựa trên 1 dự án thực tế của mình đang phát triển gần đây.

Example: [https://github.com/vucv138/React-Redux-Login-Example](https://github.com/vucv138/React-Redux-Login-Example)

## 1. Hướng dẫn test ví dụ
1. Cài đặt NodeJs và NPM: [https://nodejs.org/en](https://nodejs.org/en)
2. Download hoặc clone project: [https://github.com/vucv138/React-Redux-Login-Example](https://github.com/vucv138/React-Redux-Login-Example)
3. Cài đăng những packages cần thiết cho project bằng lệnh `npm install` (xem chi tiết các packages tại file package.json)
4. Chạy chương trình bằng lệnh `npm start`

## 2. Cấu trúc thư mục project
Tất cả mã nguồn của project đều nằm trong thư mục `/src`. Trong thư mục src có các folder feature (App, HomePage, LoginPage, RegisterPage) và các thư mục non-feature (_actions, _components, _constants, _helpers, _reducers, _services)

Các thư mục có tiền tố `_` để phân biệt giữa các feature và non-feature. Nó giúp cho việc tìm kiếm, điều hướng xung quanh project nhanh và dễ dàng hơn.

Các files index.js trong mỗi folders nhóm tất cả các module với nhau và cho phép import nhiều module với nhau trong một lần (vd: `import { userActions, alertActions } from '../_actions';`)