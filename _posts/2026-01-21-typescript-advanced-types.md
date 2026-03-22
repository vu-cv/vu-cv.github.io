---
layout: article
title: TypeScript – Advanced Types thực tế trong dự án NodeJS
tags: [typescript, advanced-types, nodejs, nestjs, type-system]
---
TypeScript không chỉ là "JavaScript với type annotations" — hệ thống type nâng cao giúp bắt lỗi compile-time, tự document code, và làm refactor an toàn hơn. Bài này tập trung vào các pattern thực tế.

## 1. Utility Types hay dùng nhất

```typescript
// Partial — tất cả fields optional
type UpdateUserDto = Partial<User>;
// { name?: string; email?: string; password?: string }

// Required — tất cả fields required
type CompleteProfile = Required<User>;

// Pick — chọn một số fields
type UserSummary = Pick<User, 'id' | 'name' | 'avatar'>;

// Omit — loại bỏ một số fields
type CreateUserDto = Omit<User, 'id' | 'createdAt' | 'updatedAt'>;

// Record — mapping type
type RolePermissions = Record<'admin' | 'user' | 'guest', string[]>;
const permissions: RolePermissions = {
  admin: ['read', 'write', 'delete'],
  user: ['read', 'write'],
  guest: ['read'],
};

// Readonly — immutable
type ImmutableUser = Readonly<User>;
// Không thể assign: user.name = 'new name'; // Error!

// ReturnType — lấy return type của function
type ApiResponse = ReturnType<typeof fetchUser>;
// → Promise<User>
```

## 2. Conditional Types

```typescript
// IsArray — kiểm tra xem T có phải array không
type IsArray<T> = T extends any[] ? true : false;
type A = IsArray<string[]>;  // true
type B = IsArray<string>;    // false

// NonNullable — loại bỏ null và undefined
type SafeString = NonNullable<string | null | undefined>; // string

// Unwrap Promise
type Awaited<T> = T extends Promise<infer U> ? Awaited<U> : T;
type UserFromPromise = Awaited<Promise<User>>; // User

// Flatten array
type Flatten<T> = T extends Array<infer U> ? U : T;
type UserFromArray = Flatten<User[]>;  // User
type StringType = Flatten<string>;     // string (không thay đổi)

// Thực tế — Extract type từ API response
async function getUsers(): Promise<{ data: User[]; total: number }> { ... }
type UsersData = Awaited<ReturnType<typeof getUsers>>;
// → { data: User[]; total: number }
type UserArray = UsersData['data'];
// → User[]
```

## 3. Template Literal Types

```typescript
// Tạo event names có type-safety
type EntityName = 'user' | 'order' | 'product';
type Action = 'created' | 'updated' | 'deleted';
type EventName = `${EntityName}.${Action}`;
// → 'user.created' | 'user.updated' | 'user.deleted' | 'order.created' | ...

// Kiểm tra tại compile time
function emit(event: EventName, data: any) { ... }
emit('user.created', data);   // ✅
emit('user.invalid', data);   // ❌ Error!

// CSS properties với type safety
type CSSProperty = `${string}-${string}`;
type FlexDirection = `${'row' | 'column'}${'' | '-reverse'}`;
// → 'row' | 'column' | 'row-reverse' | 'column-reverse'

// API route patterns
type ApiRoute = `/api/${'users' | 'orders' | 'products'}/${string}`;
```

## 4. Mapped Types

```typescript
// Tạo nullable version của type
type Nullable<T> = { [K in keyof T]: T[K] | null };
type NullableUser = Nullable<User>;
// { id: string | null; name: string | null; ... }

// Getter/Setter naming
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
type UserGetters = Getters<{ name: string; email: string }>;
// { getName: () => string; getEmail: () => string }

// Filter by value type
type PickByType<T, Value> = {
  [K in keyof T as T[K] extends Value ? K : never]: T[K]
};
type StringFields = PickByType<User, string>;
// { name: string; email: string; password: string }

// Optional deep
type DeepPartial<T> = {
  [K in keyof T]?: T[K] extends object ? DeepPartial<T[K]> : T[K];
};
```

## 5. Discriminated Unions — Xử lý kết quả API

```typescript
// Pattern Result<T, E>
type Result<T, E = Error> =
  | { success: true; data: T }
  | { success: false; error: E };

async function fetchUser(id: string): Promise<Result<User>> {
  try {
    const user = await userService.findById(id);
    return { success: true, data: user };
  } catch (error) {
    return { success: false, error: error as Error };
  }
}

// Usage với type narrowing
const result = await fetchUser('123');
if (result.success) {
  console.log(result.data.name); // TypeScript biết result.data là User
} else {
  console.error(result.error.message); // TypeScript biết result.error là Error
}

// Event system với discriminated unions
type AppEvent =
  | { type: 'user.created'; payload: { userId: string; email: string } }
  | { type: 'order.placed'; payload: { orderId: string; total: number } }
  | { type: 'payment.failed'; payload: { orderId: string; reason: string } };

function handleEvent(event: AppEvent) {
  switch (event.type) {
    case 'user.created':
      sendWelcomeEmail(event.payload.email); // Type-safe!
      break;
    case 'order.placed':
      notifyWarehouse(event.payload.orderId);
      break;
    case 'payment.failed':
      retryPayment(event.payload.orderId);
      break;
  }
}
```

## 6. Generic Constraints và Infer

```typescript
// Repository pattern với generic
interface Repository<T extends { id: string }> {
  findById(id: string): Promise<T>;
  findAll(filter?: Partial<T>): Promise<T[]>;
  create(data: Omit<T, 'id' | 'createdAt'>): Promise<T>;
  update(id: string, data: Partial<Omit<T, 'id'>>): Promise<T>;
  delete(id: string): Promise<void>;
}

// Dùng
class UserRepository implements Repository<User> {
  async findById(id: string): Promise<User> { ... }
  // TypeScript enforce đúng signature
}

// Extract keys với specific type
function getTypedKeys<T extends object>(obj: T): Array<keyof T> {
  return Object.keys(obj) as Array<keyof T>;
}

const user = { id: '1', name: 'A', email: 'a@test.com' };
const keys = getTypedKeys(user); // ('id' | 'name' | 'email')[]

// Builder pattern với generic accumulation
class QueryBuilder<T extends object, Selected extends keyof T = keyof T> {
  select<K extends keyof T>(...fields: K[]): QueryBuilder<T, K> {
    return new QueryBuilder<T, K>();
  }

  build(): Pick<T, Selected>[] {
    return []; // Return đúng type dựa trên selected fields
  }
}
```

## 7. Type Guards

```typescript
// User-defined type guard
function isUser(obj: any): obj is User {
  return (
    typeof obj === 'object' &&
    typeof obj.id === 'string' &&
    typeof obj.email === 'string'
  );
}

// Assertion function
function assertUser(obj: any): asserts obj is User {
  if (!isUser(obj)) throw new Error('Not a valid User');
}

// Pattern trong API handler
async function getUser(req: Request) {
  const data = await fetchFromDB(req.params.id);

  if (!isUser(data)) {
    throw new NotFoundException('User not found or invalid data');
  }

  return data; // TypeScript biết data là User từ đây
}

// Narrowing với in operator
type Cat = { meow(): void };
type Dog = { bark(): void };
type Animal = Cat | Dog;

function makeSound(animal: Animal) {
  if ('meow' in animal) {
    animal.meow(); // Cat
  } else {
    animal.bark(); // Dog
  }
}
```

## 8. Practical Patterns trong NestJS

```typescript
// Type-safe config
interface AppConfig {
  database: {
    host: string;
    port: number;
    name: string;
  };
  jwt: {
    secret: string;
    expiresIn: string;
  };
  redis: {
    host: string;
    port: number;
  };
}

// Nested key path với type safety
type NestedKeyOf<T, K extends keyof T = keyof T> =
  K extends string
    ? T[K] extends Record<string, any>
      ? `${K}.${NestedKeyOf<T[K]>}` | K
      : K
    : never;

type ConfigPath = NestedKeyOf<AppConfig>;
// 'database' | 'database.host' | 'database.port' | 'jwt' | 'jwt.secret' | ...

// Type-safe environment variables
function getEnv<T extends string>(key: T): string {
  const value = process.env[key];
  if (!value) throw new Error(`Missing env: ${key}`);
  return value;
}

// Readonly config
const config = Object.freeze({
  port: Number(process.env.PORT ?? 3000),
  nodeEnv: process.env.NODE_ENV ?? 'development',
} as const);

type Config = typeof config;
// { readonly port: number; readonly nodeEnv: string }
```

## 9. Kết luận

| Pattern | Khi dùng |
|---------|---------|
| `Partial<T>` / `Required<T>` | DTO, update operations |
| `Pick` / `Omit` | Tạo subset type từ entity |
| Discriminated Unions | Event system, Result type, state machine |
| Template Literal | Event names, API routes, CSS |
| Mapped Types | Transform type structure |
| Type Guards | Validate data tại runtime |
| Generic Constraints | Repository, Service patterns |

TypeScript advanced types không phải để "hack" — chúng là công cụ để **express intent** rõ ràng hơn trong code. Khi type hệ thống mạnh, refactor an toàn, và IDE support tốt hơn đáng kể.
