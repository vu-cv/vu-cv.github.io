---
layout: article
title: AWS Cognito – Authentication as a Service cho NodeJS
tags: [aws, cognito, authentication, jwt, nodejs, nestjs, oauth]
---
AWS Cognito cung cấp authentication/authorization hoàn chỉnh như một dịch vụ — đăng ký, đăng nhập, MFA, OAuth2, quản lý user pool — không cần implement từ đầu. Phù hợp khi đã dùng AWS ecosystem.

## 1. Cognito vs tự implement

| Tính năng | Cognito | Tự implement |
|---------|---------|-------------|
| User pool | ✅ Built-in | Cần code |
| MFA | ✅ Built-in | Cần code |
| OAuth2/OIDC | ✅ | Phức tạp |
| Social login | ✅ (Google, Facebook) | Cần integrate |
| Password policy | ✅ Configurable | Manual |
| Token management | ✅ JWT tự động | Cần implement |
| Chi phí | 50k MAU miễn phí | Chỉ server cost |

## 2. Setup Cognito User Pool

```bash
# Tạo User Pool
aws cognito-idp create-user-pool \
  --pool-name shopxyz-users \
  --policies '{
    "PasswordPolicy": {
      "MinimumLength": 8,
      "RequireUppercase": true,
      "RequireLowercase": true,
      "RequireNumbers": true,
      "TemporaryPasswordValidityDays": 7
    }
  }' \
  --auto-verified-attributes email \
  --username-attributes email \
  --mfa-configuration OPTIONAL \
  --region ap-southeast-1

# Tạo App Client
aws cognito-idp create-user-pool-client \
  --user-pool-id ap-southeast-1_xxxxxxxx \
  --client-name shopxyz-web \
  --explicit-auth-flows ALLOW_USER_SRP_AUTH ALLOW_REFRESH_TOKEN_AUTH \
  --no-generate-secret \
  --region ap-southeast-1
```

## 3. Kết nối từ NestJS

```bash
npm install @aws-sdk/client-cognito-identity-provider amazon-cognito-identity-js
npm install jwks-rsa jsonwebtoken
```

```typescript
// src/auth/cognito.service.ts
import {
  CognitoIdentityProviderClient,
  InitiateAuthCommand,
  SignUpCommand,
  ConfirmSignUpCommand,
  GetUserCommand,
  GlobalSignOutCommand,
  ForgotPasswordCommand,
  ConfirmForgotPasswordCommand,
  ChangePasswordCommand,
  AdminCreateUserCommand,
  AdminSetUserPasswordCommand,
  AdminGetUserCommand,
  AdminDeleteUserCommand,
} from '@aws-sdk/client-cognito-identity-provider';
import * as crypto from 'crypto';

@Injectable()
export class CognitoService {
  private client: CognitoIdentityProviderClient;
  private readonly userPoolId: string;
  private readonly clientId: string;
  private readonly clientSecret: string;

  constructor() {
    this.client = new CognitoIdentityProviderClient({
      region: process.env.AWS_REGION ?? 'ap-southeast-1',
    });
    this.userPoolId = process.env.COGNITO_USER_POOL_ID!;
    this.clientId = process.env.COGNITO_CLIENT_ID!;
    this.clientSecret = process.env.COGNITO_CLIENT_SECRET ?? '';
  }

  // Tính SECRET_HASH khi App Client có secret
  private getSecretHash(username: string): string {
    if (!this.clientSecret) return '';
    const hmac = crypto.createHmac('sha256', this.clientSecret);
    hmac.update(username + this.clientId);
    return hmac.digest('base64');
  }

  // Đăng ký
  async signUp(email: string, password: string, name: string) {
    const command = new SignUpCommand({
      ClientId: this.clientId,
      Username: email,
      Password: password,
      SecretHash: this.getSecretHash(email),
      UserAttributes: [
        { Name: 'email', Value: email },
        { Name: 'name', Value: name },
      ],
    });

    return this.client.send(command);
  }

  // Xác nhận email (OTP từ Cognito gửi)
  async confirmSignUp(email: string, code: string) {
    const command = new ConfirmSignUpCommand({
      ClientId: this.clientId,
      Username: email,
      ConfirmationCode: code,
      SecretHash: this.getSecretHash(email),
    });

    return this.client.send(command);
  }

  // Đăng nhập → trả về JWT tokens
  async signIn(email: string, password: string): Promise<{
    accessToken: string;
    idToken: string;
    refreshToken: string;
    expiresIn: number;
  }> {
    const command = new InitiateAuthCommand({
      AuthFlow: 'USER_PASSWORD_AUTH',
      ClientId: this.clientId,
      AuthParameters: {
        USERNAME: email,
        PASSWORD: password,
        SECRET_HASH: this.getSecretHash(email),
      },
    });

    const result = await this.client.send(command);
    const auth = result.AuthenticationResult!;

    return {
      accessToken: auth.AccessToken!,
      idToken: auth.IdToken!,
      refreshToken: auth.RefreshToken!,
      expiresIn: auth.ExpiresIn!,
    };
  }

  // Refresh token
  async refreshTokens(refreshToken: string, email: string) {
    const command = new InitiateAuthCommand({
      AuthFlow: 'REFRESH_TOKEN_AUTH',
      ClientId: this.clientId,
      AuthParameters: {
        REFRESH_TOKEN: refreshToken,
        SECRET_HASH: this.getSecretHash(email),
      },
    });

    const result = await this.client.send(command);
    return result.AuthenticationResult;
  }

  // Quên mật khẩu
  async forgotPassword(email: string) {
    const command = new ForgotPasswordCommand({
      ClientId: this.clientId,
      Username: email,
      SecretHash: this.getSecretHash(email),
    });
    return this.client.send(command);
  }

  // Đặt lại mật khẩu
  async confirmForgotPassword(email: string, code: string, newPassword: string) {
    const command = new ConfirmForgotPasswordCommand({
      ClientId: this.clientId,
      Username: email,
      ConfirmationCode: code,
      Password: newPassword,
      SecretHash: this.getSecretHash(email),
    });
    return this.client.send(command);
  }

  // Đăng xuất tất cả thiết bị
  async signOut(accessToken: string) {
    const command = new GlobalSignOutCommand({ AccessToken: accessToken });
    return this.client.send(command);
  }
}
```

## 4. Verify JWT Token

```typescript
// src/auth/cognito-jwt.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import * as jwksClient from 'jwks-rsa';

@Injectable()
export class CognitoJwtStrategy extends PassportStrategy(Strategy, 'cognito-jwt') {
  constructor() {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      // Cognito tự quản lý public keys — lấy từ JWKS endpoint
      secretOrKeyProvider: jwksClient.passportJwtSecret({
        cache: true,
        rateLimit: true,
        jwksRequestsPerMinute: 5,
        jwksUri: `https://cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}/.well-known/jwks.json`,
      }),
      audience: process.env.COGNITO_CLIENT_ID,
      issuer: `https://cognito-idp.${process.env.AWS_REGION}.amazonaws.com/${process.env.COGNITO_USER_POOL_ID}`,
      algorithms: ['RS256'],
    });
  }

  async validate(payload: any) {
    // payload chứa: sub, email, cognito:username, token_use
    return {
      userId: payload.sub,
      email: payload.email,
      name: payload.name,
      groups: payload['cognito:groups'] ?? [],
    };
  }
}
```

## 5. Auth Controller

```typescript
// src/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  constructor(private cognitoService: CognitoService) {}

  @Post('register')
  async register(@Body() dto: { email: string; password: string; name: string }) {
    await this.cognitoService.signUp(dto.email, dto.password, dto.name);
    return { message: 'Đã gửi email xác nhận. Vui lòng kiểm tra hộp thư.' };
  }

  @Post('confirm')
  async confirmEmail(@Body() dto: { email: string; code: string }) {
    await this.cognitoService.confirmSignUp(dto.email, dto.code);
    return { message: 'Xác nhận email thành công!' };
  }

  @Post('login')
  async login(@Body() dto: { email: string; password: string }) {
    return this.cognitoService.signIn(dto.email, dto.password);
  }

  @Post('refresh')
  async refresh(@Body() dto: { refreshToken: string; email: string }) {
    return this.cognitoService.refreshTokens(dto.refreshToken, dto.email);
  }

  @Post('logout')
  @UseGuards(CognitoJwtAuthGuard)
  async logout(@Headers('authorization') auth: string) {
    const token = auth.replace('Bearer ', '');
    await this.cognitoService.signOut(token);
    return { message: 'Đã đăng xuất' };
  }

  @Get('me')
  @UseGuards(CognitoJwtAuthGuard)
  getProfile(@CurrentUser() user: any) {
    return user;
  }
}
```

## 6. Hosted UI — Đăng nhập qua Cognito UI

```typescript
// Redirect đến Cognito Hosted UI (có sẵn UI login/register)
const loginUrl = `https://shopxyz.auth.ap-southeast-1.amazoncognito.com/login?` +
  `client_id=${clientId}` +
  `&response_type=code` +
  `&scope=openid+email+profile` +
  `&redirect_uri=${encodeURIComponent('https://shopxyz.com/auth/callback')}`;

// Callback — đổi code lấy tokens
@Get('callback')
async callback(@Query('code') code: string) {
  const tokens = await this.cognitoService.exchangeCodeForTokens(code);
  // Lưu tokens, redirect về app
}
```

## 7. Kết luận

- **User Pool**: Quản lý users, sessions, policies — không cần DB riêng cho auth
- **JWT verify**: Dùng JWKS endpoint của Cognito — keys tự rotate
- **Hosted UI**: Nhanh nhất — redirect đến Cognito UI, không cần build form
- **SECRET_HASH**: Bắt buộc khi App Client có secret — tính từ username + clientId
- **Nhược điểm**: Vendor lock-in AWS, customize UI phức tạp, giá tăng khi > 50k MAU

Cognito phù hợp khi đã dùng AWS và không muốn maintain auth service — tradeoff là flexibility thấp hơn.
