---
layout: article
title: NestJS – Unit Testing với Jest
tags: [nestjs, testing, jest, unit-test, nodejs]
---
Unit Testing giúp đảm bảo từng service/controller hoạt động đúng, phát hiện bug sớm, và tự tin refactor code. NestJS tích hợp sẵn Jest — bài này hướng dẫn viết unit test thực tế.

## 1. Cấu trúc test trong NestJS

NestJS CLI tự tạo file `*.spec.ts` khi generate module:

```
src/
  users/
    users.service.ts
    users.service.spec.ts    ← Unit test
    users.controller.ts
    users.controller.spec.ts ← Unit test
```

Chạy test:

```bash
npm run test           # Chạy một lần
npm run test:watch     # Watch mode
npm run test:cov       # Coverage report
```

## 2. Unit Test Service

### UsersService

```typescript
// src/users/users.service.ts
@Injectable()
export class UsersService {
  constructor(
    @InjectModel(User.name) private userModel: Model<UserDocument>,
  ) {}

  async findById(id: string): Promise<User> {
    const user = await this.userModel.findById(id).exec();
    if (!user) throw new NotFoundException(`User ${id} không tồn tại`);
    return user;
  }

  async create(dto: CreateUserDto): Promise<User> {
    const existing = await this.userModel.findOne({ email: dto.email });
    if (existing) throw new ConflictException('Email đã tồn tại');
    return this.userModel.create(dto);
  }

  async update(id: string, dto: UpdateUserDto): Promise<User> {
    const user = await this.userModel.findByIdAndUpdate(id, dto, { new: true });
    if (!user) throw new NotFoundException(`User ${id} không tồn tại`);
    return user;
  }
}
```

### Test với Mock

```typescript
// src/users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { getModelToken } from '@nestjs/mongoose';
import { ConflictException, NotFoundException } from '@nestjs/common';
import { UsersService } from './users.service';
import { User } from './schemas/user.schema';

// Mock data
const mockUser = {
  _id: '507f1f77bcf86cd799439011',
  name: 'Nguyen Van A',
  email: 'a@example.com',
  role: 'user',
};

// Mock Mongoose Model
const mockUserModel = {
  findById: jest.fn(),
  findOne: jest.fn(),
  findByIdAndUpdate: jest.fn(),
  create: jest.fn(),
};

describe('UsersService', () => {
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: getModelToken(User.name),
          useValue: mockUserModel,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    jest.clearAllMocks(); // Reset mocks trước mỗi test
  });

  describe('findById', () => {
    it('nên trả về user khi tìm thấy', async () => {
      mockUserModel.findById.mockReturnValue({
        exec: jest.fn().mockResolvedValue(mockUser),
      });

      const result = await service.findById(mockUser._id);

      expect(result).toEqual(mockUser);
      expect(mockUserModel.findById).toHaveBeenCalledWith(mockUser._id);
    });

    it('nên throw NotFoundException khi không tìm thấy', async () => {
      mockUserModel.findById.mockReturnValue({
        exec: jest.fn().mockResolvedValue(null),
      });

      await expect(service.findById('nonexistent-id'))
        .rejects.toThrow(NotFoundException);
    });
  });

  describe('create', () => {
    const dto = { name: 'Tran Van B', email: 'b@example.com', password: '123456' };

    it('nên tạo user mới thành công', async () => {
      mockUserModel.findOne.mockResolvedValue(null); // Email chưa tồn tại
      mockUserModel.create.mockResolvedValue({ _id: 'new-id', ...dto });

      const result = await service.create(dto);

      expect(mockUserModel.findOne).toHaveBeenCalledWith({ email: dto.email });
      expect(mockUserModel.create).toHaveBeenCalledWith(dto);
      expect(result.email).toBe(dto.email);
    });

    it('nên throw ConflictException khi email đã tồn tại', async () => {
      mockUserModel.findOne.mockResolvedValue(mockUser); // Email đã tồn tại

      await expect(service.create(dto)).rejects.toThrow(ConflictException);
      expect(mockUserModel.create).not.toHaveBeenCalled();
    });
  });
});
```

## 3. Unit Test Controller

```typescript
// src/users/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

// Mock toàn bộ UsersService
const mockUsersService = {
  findById: jest.fn(),
  findAll: jest.fn(),
  create: jest.fn(),
  update: jest.fn(),
  remove: jest.fn(),
};

describe('UsersController', () => {
  let controller: UsersController;
  let service: UsersService;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        { provide: UsersService, useValue: mockUsersService },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get<UsersService>(UsersService);
    jest.clearAllMocks();
  });

  describe('findOne', () => {
    it('nên gọi service.findById với đúng id', async () => {
      const mockUser = { _id: '123', name: 'Test User' };
      mockUsersService.findById.mockResolvedValue(mockUser);

      const result = await controller.findOne('123');

      expect(service.findById).toHaveBeenCalledWith('123');
      expect(result).toEqual(mockUser);
    });
  });

  describe('create', () => {
    it('nên tạo user và trả về kết quả', async () => {
      const dto = { name: 'New User', email: 'new@test.com', password: '123' };
      const created = { _id: 'new-id', ...dto };
      mockUsersService.create.mockResolvedValue(created);

      const result = await controller.create(dto as any);

      expect(service.create).toHaveBeenCalledWith(dto);
      expect(result).toEqual(created);
    });
  });
});
```

## 4. Test Service có dependency phức tạp

```typescript
// AuthService phụ thuộc vào UsersService và JwtService
describe('AuthService', () => {
  let authService: AuthService;
  let usersService: jest.Mocked<UsersService>;
  let jwtService: jest.Mocked<JwtService>;

  beforeEach(async () => {
    const module = await Test.createTestingModule({
      providers: [
        AuthService,
        {
          provide: UsersService,
          useValue: {
            findByEmail: jest.fn(),
            create: jest.fn(),
          },
        },
        {
          provide: JwtService,
          useValue: {
            sign: jest.fn().mockReturnValue('mock-jwt-token'),
            verify: jest.fn(),
          },
        },
      ],
    }).compile();

    authService = module.get(AuthService);
    usersService = module.get(UsersService);
    jwtService = module.get(JwtService);
  });

  describe('login', () => {
    it('nên trả về token khi credentials hợp lệ', async () => {
      const user = { _id: '123', email: 'a@test.com', password: await bcrypt.hash('123456', 10) };
      usersService.findByEmail.mockResolvedValue(user as any);

      const result = await authService.login({ email: 'a@test.com', password: '123456' });

      expect(result.access_token).toBe('mock-jwt-token');
      expect(jwtService.sign).toHaveBeenCalledWith({ sub: user._id, email: user.email });
    });

    it('nên throw UnauthorizedException khi sai mật khẩu', async () => {
      const user = { _id: '123', email: 'a@test.com', password: await bcrypt.hash('correct', 10) };
      usersService.findByEmail.mockResolvedValue(user as any);

      await expect(authService.login({ email: 'a@test.com', password: 'wrong' }))
        .rejects.toThrow(UnauthorizedException);
    });
  });
});
```

## 5. Spy và Mock functions

```typescript
describe('OrdersService', () => {
  it('nên emit event sau khi tạo order', async () => {
    const emitSpy = jest.spyOn(eventEmitter, 'emit');
    mockOrderModel.create.mockResolvedValue({ _id: 'order-123', ...dto });

    await service.create(dto);

    expect(emitSpy).toHaveBeenCalledWith('order.created', expect.objectContaining({
      orderId: 'order-123',
    }));
  });

  it('nên log error khi gửi email thất bại', async () => {
    const logSpy = jest.spyOn(service['logger'], 'error');
    mockMailService.send.mockRejectedValue(new Error('SMTP error'));

    await service.notifyUser('user-id'); // Không throw — chỉ log

    expect(logSpy).toHaveBeenCalledWith(expect.stringContaining('SMTP error'));
  });
});
```

## 6. Coverage Report

```bash
npm run test:cov
```

```
--------------------|---------|----------|---------|---------|
File                | % Stmts | % Branch | % Funcs | % Lines |
--------------------|---------|----------|---------|---------|
users.service.ts    |   95.45 |    88.89 |     100 |   95.45 |
users.controller.ts |     100 |      100 |     100 |     100 |
auth.service.ts     |   91.30 |    83.33 |     100 |   91.30 |
--------------------|---------|----------|---------|---------|
```

Mục tiêu: **>80% coverage** cho các service quan trọng (payment, auth, order).

## 7. Kết luận

- **Mock Model**: Dùng `getModelToken()` + mock object thay vì kết nối MongoDB thật
- **`jest.clearAllMocks()`**: Reset state giữa các test để tránh test leak
- **Test behavior, not implementation**: Test "kết quả đúng không?" thay vì "có gọi hàm X không?"
- **Spy**: Dùng khi muốn verify side effect (emit event, ghi log) mà không mock toàn bộ
- **`jest.Mocked<T>`**: TypeScript type helper để có autocomplete khi dùng mock

Unit test tốt = tài liệu sống của code — người mới đọc test là hiểu service làm gì.
