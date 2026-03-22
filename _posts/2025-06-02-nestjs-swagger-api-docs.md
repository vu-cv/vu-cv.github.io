---
layout: article
title: NestJS – Viết API Documentation với Swagger
tags: [nestjs, swagger, openapi, api-docs, documentation]
---
Swagger (OpenAPI) là chuẩn documentation phổ biến nhất cho REST API. NestJS có module `@nestjs/swagger` cho phép tự động generate Swagger UI từ code, không cần viết YAML thủ công.

## 1. Cài đặt

```bash
npm install --save @nestjs/swagger
```

## 2. Cấu hình Swagger trong main.ts

```typescript
import { NestFactory } from '@nestjs/core';
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API documentation cho ứng dụng của tôi')
    .setVersion('1.0')
    .addBearerAuth(                          // Thêm nút "Authorize" cho JWT
      { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
      'access-token',
    )
    .addTag('users', 'Quản lý người dùng')
    .addTag('products', 'Quản lý sản phẩm')
    .addTag('orders', 'Quản lý đơn hàng')
    .build();

  const document = SwaggerModule.createDocument(app, config);

  // Chỉ expose docs trong môi trường không phải production
  if (process.env.NODE_ENV !== 'production') {
    SwaggerModule.setup('api-docs', app, document, {
      swaggerOptions: {
        persistAuthorization: true, // Giữ token khi reload trang
        tagsSorter: 'alpha',
        operationsSorter: 'alpha',
      },
    });
  }

  await app.listen(3000);
}
bootstrap();
```

Truy cập `http://localhost:3000/api-docs`.

## 3. Annotate DTO với Swagger decorators

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';
import { IsEmail, IsString, IsOptional, MinLength, IsEnum } from 'class-validator';

export enum UserRole {
  ADMIN = 'admin',
  USER = 'user',
  MODERATOR = 'moderator',
}

export class CreateUserDto {
  @ApiProperty({
    description: 'Tên đầy đủ của người dùng',
    example: 'Nguyen Van A',
    minLength: 2,
    maxLength: 50,
  })
  @IsString()
  @MinLength(2)
  name: string;

  @ApiProperty({
    description: 'Địa chỉ email',
    example: 'nguyenvana@example.com',
    format: 'email',
  })
  @IsEmail()
  email: string;

  @ApiProperty({
    description: 'Mật khẩu (tối thiểu 8 ký tự)',
    example: 'SecurePass123!',
    minLength: 8,
  })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({
    description: 'Vai trò người dùng',
    enum: UserRole,
    default: UserRole.USER,
  })
  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole = UserRole.USER;

  @ApiPropertyOptional({
    description: 'Số điện thoại',
    example: '0912345678',
  })
  @IsOptional()
  @IsString()
  phone?: string;
}

// Response DTO
export class UserResponseDto {
  @ApiProperty({ example: '550e8400-e29b-41d4-a716-446655440000' })
  id: string;

  @ApiProperty({ example: 'Nguyen Van A' })
  name: string;

  @ApiProperty({ example: 'nguyenvana@example.com' })
  email: string;

  @ApiProperty({ enum: UserRole })
  role: UserRole;

  @ApiProperty({ example: '2025-01-01T00:00:00.000Z' })
  createdAt: Date;
}
```

## 4. Annotate Controller

```typescript
import {
  ApiTags, ApiOperation, ApiResponse, ApiBearerAuth,
  ApiParam, ApiQuery, ApiBody, ApiConsumes,
} from '@nestjs/swagger';

@ApiTags('users')
@ApiBearerAuth('access-token')    // Yêu cầu JWT cho toàn bộ controller
@Controller('users')
export class UsersController {
  @Post()
  @ApiOperation({ summary: 'Tạo người dùng mới' })
  @ApiResponse({ status: 201, description: 'Tạo thành công', type: UserResponseDto })
  @ApiResponse({ status: 400, description: 'Dữ liệu không hợp lệ' })
  @ApiResponse({ status: 409, description: 'Email đã tồn tại' })
  create(@Body() dto: CreateUserDto): Promise<UserResponseDto> {
    return this.usersService.create(dto);
  }

  @Get()
  @ApiOperation({ summary: 'Lấy danh sách người dùng' })
  @ApiQuery({ name: 'page', required: false, type: Number, example: 1 })
  @ApiQuery({ name: 'limit', required: false, type: Number, example: 10 })
  @ApiQuery({ name: 'search', required: false, type: String })
  @ApiResponse({ status: 200, description: 'Thành công', type: [UserResponseDto] })
  findAll(@Query() query: PaginationDto) {
    return this.usersService.findAll(query);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Lấy thông tin người dùng theo ID' })
  @ApiParam({ name: 'id', description: 'UUID của người dùng', example: '550e8400-e29b-41d4-a716-446655440000' })
  @ApiResponse({ status: 200, type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'Không tìm thấy người dùng' })
  findOne(@Param('id') id: string) {
    return this.usersService.findOne(id);
  }

  @Post('avatar')
  @ApiOperation({ summary: 'Upload ảnh đại diện' })
  @ApiConsumes('multipart/form-data')
  @ApiBody({
    schema: {
      type: 'object',
      properties: {
        file: { type: 'string', format: 'binary' },
      },
    },
  })
  @ApiResponse({ status: 200, description: 'Upload thành công' })
  @UseInterceptors(FileInterceptor('file'))
  uploadAvatar(@UploadedFile() file: Express.Multer.File) {
    return this.usersService.uploadAvatar(file);
  }
}
```

## 5. API Response chuẩn hóa

```typescript
// Generic response wrapper
export class PaginatedResponseDto<T> {
  @ApiProperty({ isArray: true })
  items: T[];

  @ApiProperty({ example: 100 })
  total: number;

  @ApiProperty({ example: 1 })
  page: number;

  @ApiProperty({ example: 10 })
  limit: number;

  @ApiProperty({ example: 10 })
  totalPages: number;
}

// Dùng trong controller
@ApiResponse({
  status: 200,
  schema: {
    allOf: [
      { properties: { items: { type: 'array', items: { $ref: '#/components/schemas/UserResponseDto' } } } },
      { properties: { total: { type: 'number' } } },
    ],
  },
})
```

## 6. Export Swagger JSON để dùng với Postman

```typescript
// Lưu swagger.json khi build
const document = SwaggerModule.createDocument(app, config);
writeFileSync('./swagger.json', JSON.stringify(document, null, 2));
```

Import file này vào Postman: **Import → File → swagger.json** — tất cả endpoints tự động có sẵn.

## 7. Kết luận

`@nestjs/swagger` biến comment code thành documentation tương tác:

- `@ApiProperty` — mô tả field trong DTO, tự generate Swagger schema
- `@ApiOperation` / `@ApiResponse` — mô tả endpoint và responses
- `@ApiTags` — nhóm endpoints
- `@ApiBearerAuth` — thêm authentication vào UI
- Swagger UI cho phép test API ngay trên trình duyệt, không cần Postman
