---
layout: article
title: NestJS – CRUD với MongoDB (Mongoose)
tags: [nestjs, mongodb, mongoose, crud]
---
Trong bài này, chúng ta sẽ xây dựng một API CRUD hoàn chỉnh với NestJS và MongoDB thông qua thư viện Mongoose. Đây là bộ đôi rất phổ biến trong thực tế.

## 1. Chuẩn bị

Bạn cần có project NestJS đã khởi tạo. Nếu chưa có, hãy xem bài [Tìm hiểu & Cài đặt NestJS](/2026/03/23/tim-hieu-va-cai-dat-nestjs.html).

Cài đặt Mongoose cho NestJS:

```bash
npm install --save @nestjs/mongoose mongoose
```

## 2. Kết nối MongoDB

Cập nhật `app.module.ts` để kết nối database:

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    MongooseModule.forRoot('mongodb://localhost:27017/nestjs-demo'),
  ],
})
export class AppModule {}
```

> **Tip:** Nên đưa connection string vào biến môi trường thay vì hardcode trực tiếp.

Cài đặt `@nestjs/config` để quản lý env:
```bash
npm install --save @nestjs/config
```

Tạo file `.env`:
```
MONGODB_URI=mongodb://localhost:27017/nestjs-demo
```

Cập nhật `app.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    MongooseModule.forRootAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        uri: configService.get<string>('MONGODB_URI'),
      }),
      inject: [ConfigService],
    }),
  ],
})
export class AppModule {}
```

## 3. Tạo Resource Users

```bash
nest g resource users
```

Chọn **REST API** và **Yes** cho CRUD entry points.

## 4. Định nghĩa Schema

Tạo file `src/users/schemas/user.schema.ts`:

```typescript
import { Prop, Schema, SchemaFactory } from '@nestjs/mongoose';
import { HydratedDocument } from 'mongoose';

export type UserDocument = HydratedDocument<User>;

@Schema({ timestamps: true })
export class User {
  @Prop({ required: true })
  name: string;

  @Prop({ required: true, unique: true, lowercase: true })
  email: string;

  @Prop()
  age: number;

  @Prop({ default: true })
  isActive: boolean;
}

export const UserSchema = SchemaFactory.createForClass(User);
```

## 5. Đăng ký Schema vào Module

Cập nhật `src/users/users.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { MongooseModule } from '@nestjs/mongoose';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { User, UserSchema } from './schemas/user.schema';

@Module({
  imports: [
    MongooseModule.forFeature([{ name: User.name, schema: UserSchema }]),
  ],
  controllers: [UsersController],
  providers: [UsersService],
})
export class UsersModule {}
```

## 6. Tạo DTO

`src/users/dto/create-user.dto.ts`:
```typescript
export class CreateUserDto {
  name: string;
  email: string;
  age?: number;
}
```

`src/users/dto/update-user.dto.ts`:
```typescript
import { PartialType } from '@nestjs/mapped-types';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(CreateUserDto) {}
```

## 7. Viết Service

Cập nhật `src/users/users.service.ts`:

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { User, UserDocument } from './schemas/user.schema';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>,
  ) {}

  async create(createUserDto: CreateUserDto): Promise<User> {
    const user = new this.userModel(createUserDto);
    return user.save();
  }

  async findAll(): Promise<User[]> {
    return this.userModel.find().exec();
  }

  async findOne(id: string): Promise<User> {
    const user = await this.userModel.findById(id).exec();
    if (!user) throw new NotFoundException(`User #${id} không tồn tại`);
    return user;
  }

  async update(id: string, updateUserDto: UpdateUserDto): Promise<User> {
    const user = await this.userModel
      .findByIdAndUpdate(id, updateUserDto, { new: true })
      .exec();
    if (!user) throw new NotFoundException(`User #${id} không tồn tại`);
    return user;
  }

  async remove(id: string): Promise<void> {
    const result = await this.userModel.findByIdAndDelete(id).exec();
    if (!result) throw new NotFoundException(`User #${id} không tồn tại`);
  }
}
```

## 8. Viết Controller

Cập nhật `src/users/users.controller.ts`:

```typescript
import {
  Controller,
  Get,
  Post,
  Body,
  Param,
  Put,
  Delete,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  findAll() {
    return this.usersService.findAll();
  }

  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Put(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  remove(@Param('id') id: string) {
    return this.usersService.remove(id);
  }
}
```

## 9. Đăng ký UsersModule vào AppModule

```typescript
import { Module } from '@nestjs/common';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { UsersModule } from './users/users.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }),
    MongooseModule.forRootAsync({
      useFactory: (cs: ConfigService) => ({ uri: cs.get('MONGODB_URI') }),
      inject: [ConfigService],
    }),
    UsersModule,
  ],
})
export class AppModule {}
```

## 10. Test API

Chạy server:
```bash
npm run start:dev
```

Dùng curl hoặc Postman để test:

```bash
# Tạo user
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name":"Nguyen Van A","email":"a@example.com","age":25}'

# Lấy danh sách
curl http://localhost:3000/users

# Lấy theo ID
curl http://localhost:3000/users/64abc123def456789012

# Cập nhật
curl -X PUT http://localhost:3000/users/64abc123def456789012 \
  -H "Content-Type: application/json" \
  -d '{"age":26}'

# Xóa
curl -X DELETE http://localhost:3000/users/64abc123def456789012
```

## 11. Kết luận

Chúng ta đã xây dựng xong một REST API CRUD với NestJS + MongoDB:

- **Schema**: Định nghĩa cấu trúc document với `@Schema`, `@Prop`
- **Service**: Xử lý business logic với `Model<UserDocument>`
- **Controller**: Map HTTP endpoints vào service methods
- **DTO**: Validate và type dữ liệu đầu vào

Bài tiếp theo sẽ hướng dẫn thêm **Authentication với JWT** để bảo vệ các API này.
