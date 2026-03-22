---
layout: article
title: NestJS – Validation & Transform với class-validator
tags: [nestjs, validation, class-validator, class-transformer, dto]
---
Validate dữ liệu đầu vào là bước quan trọng để bảo vệ ứng dụng. NestJS tích hợp `class-validator` và `class-transformer` giúp validate DTO một cách khai báo, sạch sẽ và tái sử dụng được.

## 1. Cài đặt

```bash
npm install --save class-validator class-transformer
```

## 2. Bật Global ValidationPipe

Kích hoạt validate tự động cho toàn bộ app trong `main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,        // Tự động xóa field không có trong DTO
      forbidNonWhitelisted: true, // Throw lỗi nếu có field thừa
      transform: true,        // Tự động chuyển đổi type (string → number...)
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  await app.listen(3000);
}
bootstrap();
```

## 3. Các decorator phổ biến

### Kiểu dữ liệu cơ bản

```typescript
import {
  IsString, IsNumber, IsEmail, IsBoolean, IsDate,
  IsOptional, IsNotEmpty, IsArray, IsEnum,
  Min, Max, MinLength, MaxLength, Length,
  IsUrl, IsPhoneNumber, IsUUID,
} from 'class-validator';
import { Type } from 'class-transformer';

enum Role { ADMIN = 'admin', USER = 'user' }

export class CreateUserDto {
  @IsNotEmpty({ message: 'Tên không được để trống' })
  @IsString()
  @MinLength(2)
  @MaxLength(50)
  name: string;

  @IsEmail({}, { message: 'Email không hợp lệ' })
  email: string;

  @IsString()
  @MinLength(8, { message: 'Mật khẩu tối thiểu 8 ký tự' })
  password: string;

  @IsOptional()
  @IsNumber()
  @Min(18, { message: 'Phải đủ 18 tuổi' })
  @Max(120)
  @Type(() => Number) // Chuyển string → number tự động
  age?: number;

  @IsEnum(Role, { message: 'Role không hợp lệ' })
  role: Role;

  @IsOptional()
  @IsBoolean()
  isActive?: boolean;

  @IsOptional()
  @IsUrl()
  avatarUrl?: string;

  @IsOptional()
  @IsPhoneNumber('VN')
  phone?: string;
}
```

### Mảng và Object lồng nhau

```typescript
import { ValidateNested, ArrayMinSize, ArrayMaxSize } from 'class-validator';
import { Type } from 'class-transformer';

export class AddressDto {
  @IsString()
  street: string;

  @IsString()
  city: string;

  @IsOptional()
  @IsString()
  zipCode?: string;
}

export class CreateOrderDto {
  @ValidateNested()
  @Type(() => AddressDto) // BẮT BUỘC phải có @Type để validate nested object
  shippingAddress: AddressDto;

  @IsArray()
  @ArrayMinSize(1, { message: 'Giỏ hàng phải có ít nhất 1 sản phẩm' })
  @ArrayMaxSize(100)
  @IsString({ each: true }) // Validate từng phần tử trong mảng
  productIds: string[];

  @IsOptional()
  @IsArray()
  @ValidateNested({ each: true })
  @Type(() => OrderItemDto)
  items?: OrderItemDto[];
}

export class OrderItemDto {
  @IsUUID()
  productId: string;

  @IsNumber()
  @Min(1)
  quantity: number;
}
```

## 4. Custom Validator

```typescript
import {
  registerDecorator,
  ValidationOptions,
  ValidationArguments,
} from 'class-validator';

// Decorator tùy chỉnh: kiểm tra password xác nhận khớp
function IsPasswordConfirmed(property: string, validationOptions?: ValidationOptions) {
  return function (object: object, propertyName: string) {
    registerDecorator({
      name: 'isPasswordConfirmed',
      target: object.constructor,
      propertyName,
      constraints: [property],
      options: validationOptions,
      validator: {
        validate(value: any, args: ValidationArguments) {
          const [relatedPropertyName] = args.constraints;
          const relatedValue = (args.object as any)[relatedPropertyName];
          return value === relatedValue;
        },
        defaultMessage(args: ValidationArguments) {
          return 'Mật khẩu xác nhận không khớp';
        },
      },
    });
  };
}

// Sử dụng
export class RegisterDto {
  @IsEmail()
  email: string;

  @IsString()
  @MinLength(8)
  password: string;

  @IsString()
  @IsPasswordConfirmed('password')
  confirmPassword: string;
}
```

## 5. Transform với class-transformer

```typescript
import { Transform, Exclude, Expose } from 'class-transformer';

export class UserDto {
  @Expose()
  id: string;

  @Expose()
  name: string;

  @Expose()
  @Transform(({ value }) => value.toLowerCase())
  email: string;

  @Exclude() // Field này sẽ KHÔNG được serialize
  password: string;

  @Expose()
  @Transform(({ value }) => value ? 'active' : 'inactive')
  status: string;

  @Expose()
  @Transform(({ value }) => new Date(value).toLocaleDateString('vi-VN'))
  createdAt: string;
}

// Trong controller — chuyển entity sang DTO
import { plainToInstance } from 'class-transformer';

@Get(':id')
async findOne(@Param('id') id: string) {
  const user = await this.usersService.findOne(id);
  return plainToInstance(UserDto, user, { excludeExtraneousValues: true });
}
```

## 6. Validate Query Params và Params

```typescript
import { IsOptional, IsNumberString } from 'class-validator';
import { Type } from 'class-transformer';

export class PaginationDto {
  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(1)
  page?: number = 1;

  @IsOptional()
  @Type(() => Number)
  @IsNumber()
  @Min(1)
  @Max(100)
  limit?: number = 10;

  @IsOptional()
  @IsString()
  search?: string;
}

// Controller
@Get()
findAll(@Query() query: PaginationDto) {
  return this.service.findAll(query);
}
```

## 7. Response khi validate lỗi

Khi có lỗi, NestJS tự động trả về:

```json
{
  "statusCode": 400,
  "message": [
    "email phải là email hợp lệ",
    "Mật khẩu tối thiểu 8 ký tự",
    "age phải là số"
  ],
  "error": "Bad Request"
}
```

## 8. Kết luận

`class-validator` + `class-transformer` + `ValidationPipe` là bộ ba không thể thiếu trong NestJS:

- Khai báo validation bằng decorator trực quan
- `whitelist: true` tự động lọc field không mong muốn — bảo mật hơn
- `transform: true` tự động convert type — xử lý query params dễ dàng
- `@ValidateNested` + `@Type` để validate object lồng nhau
- Custom decorator cho business rule phức tạp
