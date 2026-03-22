---
layout: article
title: NestJS – Config Module & Validation biến môi trường
tags: [nestjs, config, environment, validation, nodejs, typescript]
---
Quản lý configuration đúng cách là nền tảng của mọi ứng dụng production-ready. NestJS `@nestjs/config` kết hợp `class-validator` giúp validate env variables ngay khi khởi động — app sẽ crash ngay nếu thiếu config thay vì lỗi runtime.

## 1. Cài đặt

```bash
npm install @nestjs/config joi
# Hoặc dùng class-validator thay cho joi
npm install @nestjs/config class-validator class-transformer
```

## 2. Cấu hình cơ bản

```typescript
// app.module.ts
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,       // Không cần import ở từng module
      envFilePath: [
        `.env.${process.env.NODE_ENV ?? 'development'}`, // .env.development, .env.production
        '.env',             // Fallback
      ],
      cache: true,          // Cache giá trị — tránh đọc lại nhiều lần
      expandVariables: true, // Hỗ trợ ${OTHER_VAR} trong .env
    }),
  ],
})
export class AppModule {}
```

```bash
# .env.development
NODE_ENV=development
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=mydb_dev
REDIS_URL=redis://localhost:6379
JWT_SECRET=dev-secret-not-for-prod

# .env.production
NODE_ENV=production
PORT=8080
DB_HOST=${RDS_HOSTNAME}
DB_PORT=5432
DB_NAME=mydb_prod
REDIS_URL=${ELASTICACHE_URL}
JWT_SECRET=${JWT_SECRET_FROM_SSM}
```

## 3. Validate với Joi

```typescript
import * as Joi from 'joi';

ConfigModule.forRoot({
  isGlobal: true,
  validationSchema: Joi.object({
    NODE_ENV: Joi.string().valid('development', 'production', 'test').default('development'),
    PORT: Joi.number().default(3000),

    // Database
    DB_HOST: Joi.string().required(),
    DB_PORT: Joi.number().default(5432),
    DB_USER: Joi.string().required(),
    DB_PASSWORD: Joi.string().required(),
    DB_NAME: Joi.string().required(),

    // Redis
    REDIS_URL: Joi.string().uri().required(),

    // JWT
    JWT_SECRET: Joi.string().min(32).required(),
    JWT_EXPIRES_IN: Joi.string().default('7d'),

    // AWS (optional)
    AWS_ACCESS_KEY_ID: Joi.string().when('NODE_ENV', {
      is: 'production',
      then: Joi.required(),
    }),
    AWS_SECRET_ACCESS_KEY: Joi.string().when('NODE_ENV', {
      is: 'production',
      then: Joi.required(),
    }),
  }),
  validationOptions: {
    allowUnknown: true,  // Cho phép env vars không trong schema
    abortEarly: false,   // Hiển thị tất cả lỗi một lúc
  },
}),
```

Khi thiếu `DB_HOST`, app không start được:

```
Error: Config validation error:
  "DB_HOST" is required
  "JWT_SECRET" length must be at least 32 characters long
```

## 4. Type-safe Config với namespace

```typescript
// src/config/database.config.ts
import { registerAs } from '@nestjs/config';

export const databaseConfig = registerAs('database', () => ({
  host: process.env.DB_HOST!,
  port: parseInt(process.env.DB_PORT ?? '5432', 10),
  username: process.env.DB_USER!,
  password: process.env.DB_PASSWORD!,
  name: process.env.DB_NAME!,
  ssl: process.env.NODE_ENV === 'production',
  poolSize: parseInt(process.env.DB_POOL_SIZE ?? '10', 10),
}));

export type DatabaseConfig = ReturnType<typeof databaseConfig>;

// src/config/jwt.config.ts
export const jwtConfig = registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET!,
  expiresIn: process.env.JWT_EXPIRES_IN ?? '7d',
  refreshExpiresIn: process.env.JWT_REFRESH_EXPIRES_IN ?? '30d',
}));

// src/config/mail.config.ts
export const mailConfig = registerAs('mail', () => ({
  host: process.env.SMTP_HOST!,
  port: parseInt(process.env.SMTP_PORT ?? '587', 10),
  user: process.env.SMTP_USER!,
  pass: process.env.SMTP_PASS!,
  fromName: process.env.MAIL_FROM_NAME ?? 'App',
}));
```

```typescript
// app.module.ts — Load các config namespace
ConfigModule.forRoot({
  isGlobal: true,
  load: [databaseConfig, jwtConfig, mailConfig],
}),
```

## 5. Inject Config vào Service

```typescript
// Cách 1: ConfigService (generic)
@Injectable()
export class AppService {
  constructor(private configService: ConfigService) {}

  getPort(): number {
    return this.configService.get<number>('PORT', 3000)!;
  }

  getDatabaseUrl(): string {
    // Lấy config theo namespace
    return this.configService.get<string>('database.host')!;
  }
}

// Cách 2: @InjectConfig với namespace (type-safe hơn)
import { ConfigType } from '@nestjs/config';

@Injectable()
export class DatabaseService {
  constructor(
    @Inject(databaseConfig.KEY)
    private dbConfig: ConfigType<typeof databaseConfig>,
  ) {}

  getConnectionString(): string {
    // IDE autocomplete đầy đủ!
    return `postgresql://${this.dbConfig.username}:${this.dbConfig.password}@${this.dbConfig.host}:${this.dbConfig.port}/${this.dbConfig.name}`;
  }
}
```

## 6. TypeORM dùng Config

```typescript
// src/database/database.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';
import { ConfigModule, ConfigService } from '@nestjs/config';

TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (config: ConfigService) => ({
    type: 'postgres',
    host: config.get('database.host'),
    port: config.get<number>('database.port'),
    username: config.get('database.username'),
    password: config.get('database.password'),
    database: config.get('database.name'),
    ssl: config.get<boolean>('database.ssl'),
    entities: [__dirname + '/../**/*.entity{.ts,.js}'],
    synchronize: config.get('NODE_ENV') !== 'production',
  }),
  inject: [ConfigService],
}),
```

## 7. Validate với class-validator (không dùng Joi)

```typescript
// src/config/env.validation.ts
import { plainToClass } from 'class-transformer';
import { IsEnum, IsNumber, IsString, Min, validateSync } from 'class-validator';

enum Environment {
  Development = 'development',
  Production = 'production',
  Test = 'test',
}

class EnvironmentVariables {
  @IsEnum(Environment)
  NODE_ENV: Environment = Environment.Development;

  @IsNumber()
  @Min(1)
  PORT: number = 3000;

  @IsString()
  DB_HOST: string;

  @IsNumber()
  DB_PORT: number = 5432;

  @IsString()
  JWT_SECRET: string;
}

export function validate(config: Record<string, unknown>) {
  const validatedConfig = plainToClass(EnvironmentVariables, config, {
    enableImplicitConversion: true,
  });

  const errors = validateSync(validatedConfig, { skipMissingProperties: false });

  if (errors.length > 0) {
    throw new Error(errors.toString());
  }

  return validatedConfig;
}

// Dùng trong ConfigModule
ConfigModule.forRoot({
  validate,
}),
```

## 8. Multiple .env files theo environment

```
.env                    ← Shared defaults
.env.development        ← Dev overrides
.env.production         ← Prod overrides
.env.test               ← Test overrides
.env.local              ← Local overrides (gitignore)
```

```bash
# package.json scripts
"start:dev": "NODE_ENV=development nest start --watch",
"start:prod": "NODE_ENV=production node dist/main",
"test": "NODE_ENV=test jest",
```

## 9. Kết luận

- **`ConfigModule.forRoot({ isGlobal: true })`**: Một lần, dùng được mọi nơi
- **Joi validation**: Validate ngay khi khởi động — fail fast, rõ ràng lỗi gì thiếu
- **Namespace `registerAs`**: Nhóm config theo domain (database, jwt, mail) với TypeScript type-safe
- **`@Inject(config.KEY)`**: Inject có type-checking đầy đủ — IDE autocomplete
- **`.env.{environment}`**: Phân tách config theo môi trường, không hard-code

Config validation là tuyến phòng thủ đầu tiên — đừng để thiếu `JWT_SECRET` gây lỗi sau 2 giờ runtime.
