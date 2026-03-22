---
layout: article
title: NestJS – Authentication với JWT
tags: [nestjs, jwt, auth, passport]
---
Authentication là tính năng không thể thiếu trong bất kỳ ứng dụng backend nào. Bài này hướng dẫn xây dựng hệ thống đăng nhập với JWT (JSON Web Token) trong NestJS sử dụng `@nestjs/passport` và `@nestjs/jwt`.

## 1. Cài đặt thư viện

```bash
npm install --save @nestjs/passport @nestjs/jwt passport passport-jwt bcryptjs
npm install --save-dev @types/passport-jwt @types/bcryptjs
```

## 2. Tạo Auth Module

```bash
nest g module auth
nest g controller auth
nest g service auth
```

## 3. Cấu hình biến môi trường

Thêm vào `.env`:
```
JWT_SECRET=your-super-secret-key-change-this-in-production
JWT_EXPIRES_IN=7d
```

## 4. Tạo User Schema (nếu dùng MongoDB)

`src/users/schemas/user.schema.ts`:
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

  @Prop({ required: true, select: false })
  password: string;
}

export const UserSchema = SchemaFactory.createForClass(User);
```

> `select: false` để password không bị trả về trong query thông thường.

## 5. Viết Auth Service

`src/auth/auth.service.ts`:
```typescript
import {
  Injectable,
  UnauthorizedException,
  ConflictException,
} from '@nestjs/common';
import { InjectModel } from '@nestjs/mongoose';
import { Model } from 'mongoose';
import { JwtService } from '@nestjs/jwt';
import * as bcrypt from 'bcryptjs';
import { User, UserDocument } from '../users/schemas/user.schema';

@Injectable()
export class AuthService {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>,
    private jwtService: JwtService,
  ) {}

  async register(name: string, email: string, password: string) {
    const existing = await this.userModel.findOne({ email });
    if (existing) throw new ConflictException('Email đã được sử dụng');

    const hashed = await bcrypt.hash(password, 10);
    const user = await this.userModel.create({ name, email, password: hashed });

    return this.signToken(user._id.toString(), email);
  }

  async login(email: string, password: string) {
    const user = await this.userModel.findOne({ email }).select('+password');
    if (!user) throw new UnauthorizedException('Email hoặc mật khẩu không đúng');

    const isMatch = await bcrypt.compare(password, user.password);
    if (!isMatch) throw new UnauthorizedException('Email hoặc mật khẩu không đúng');

    return this.signToken(user._id.toString(), email);
  }

  private signToken(userId: string, email: string) {
    const payload = { sub: userId, email };
    return {
      access_token: this.jwtService.sign(payload),
    };
  }
}
```

## 6. Tạo JWT Strategy

`src/auth/jwt.strategy.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  async validate(payload: { sub: string; email: string }) {
    return { userId: payload.sub, email: payload.email };
  }
}
```

## 7. Tạo Guard

`src/auth/jwt-auth.guard.ts`:
```typescript
import { Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class JwtAuthGuard extends AuthGuard('jwt') {}
```

## 8. Tạo DTO

`src/auth/dto/register.dto.ts`:
```typescript
export class RegisterDto {
  name: string;
  email: string;
  password: string;
}
```

`src/auth/dto/login.dto.ts`:
```typescript
export class LoginDto {
  email: string;
  password: string;
}
```

## 9. Viết Auth Controller

`src/auth/auth.controller.ts`:
```typescript
import { Controller, Post, Body } from '@nestjs/common';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';
import { LoginDto } from './dto/login.dto';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  @Post('register')
  register(@Body() dto: RegisterDto) {
    return this.authService.register(dto.name, dto.email, dto.password);
  }

  @Post('login')
  login(@Body() dto: LoginDto) {
    return this.authService.login(dto.email, dto.password);
  }
}
```

## 10. Cấu hình Auth Module

`src/auth/auth.module.ts`:
```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { MongooseModule } from '@nestjs/mongoose';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { JwtStrategy } from './jwt.strategy';
import { User, UserSchema } from '../users/schemas/user.schema';

@Module({
  imports: [
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (cs: ConfigService) => ({
        secret: cs.get<string>('JWT_SECRET'),
        signOptions: { expiresIn: cs.get<string>('JWT_EXPIRES_IN') },
      }),
      inject: [ConfigService],
    }),
    MongooseModule.forFeature([{ name: User.name, schema: UserSchema }]),
  ],
  controllers: [AuthController],
  providers: [AuthService, JwtStrategy],
})
export class AuthModule {}
```

## 11. Bảo vệ Route với Guard

Sử dụng `@UseGuards(JwtAuthGuard)` trong controller cần bảo vệ:

```typescript
import { Controller, Get, UseGuards, Request } from '@nestjs/common';
import { JwtAuthGuard } from '../auth/jwt-auth.guard';

@Controller('users')
export class UsersController {
  // Route này cần JWT token hợp lệ
  @UseGuards(JwtAuthGuard)
  @Get('profile')
  getProfile(@Request() req) {
    return req.user; // { userId, email }
  }
}
```

## 12. Test

```bash
# Đăng ký
curl -X POST http://localhost:3000/auth/register \
  -H "Content-Type: application/json" \
  -d '{"name":"Nguyen Van A","email":"a@example.com","password":"secret123"}'

# Đăng nhập → nhận token
curl -X POST http://localhost:3000/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"a@example.com","password":"secret123"}'

# Gọi route được bảo vệ
curl http://localhost:3000/users/profile \
  -H "Authorization: Bearer <token>"
```

## 13. Kết luận

Chúng ta đã xây dựng Authentication hoàn chỉnh với:

- **bcryptjs** — hash password an toàn
- **JWT** — cấp token sau khi đăng nhập
- **Passport + JwtStrategy** — xác thực token trong mỗi request
- **JwtAuthGuard** — bảo vệ route chỉ dành cho user đã đăng nhập

Bài tiếp theo: **NestJS – Upload File với Multer**.
