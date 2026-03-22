---
layout: article
title: NestJS – GraphQL cơ bản với Code-First approach
tags: [nestjs, graphql, apollo, typescript, nodejs]
---
GraphQL là query language cho API — client chỉ lấy đúng data cần thiết, tránh over-fetching và under-fetching. NestJS hỗ trợ GraphQL tích hợp sẵn với Apollo Server, theo hai cách: **Code-first** (TypeScript → Schema) và Schema-first. Bài này dùng Code-first.

## 1. Cài đặt

```bash
npm install @nestjs/graphql @nestjs/apollo @apollo/server graphql
```

## 2. Cấu hình AppModule

```typescript
// app.module.ts
import { GraphQLModule } from '@nestjs/graphql';
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { join } from 'path';

@Module({
  imports: [
    GraphQLModule.forRoot<ApolloDriverConfig>({
      driver: ApolloDriver,
      autoSchemaFile: join(process.cwd(), 'src/schema.gql'), // Tự tạo schema file
      sortSchema: true,
      playground: process.env.NODE_ENV !== 'production',
      context: ({ req }) => ({ req }), // Truyền request vào context (cho auth)
    }),
    UsersModule,
    PostsModule,
  ],
})
export class AppModule {}
```

## 3. Object Types — Định nghĩa data model

```typescript
// src/users/user.type.ts
import { ObjectType, Field, ID, Int } from '@nestjs/graphql';

@ObjectType()
export class User {
  @Field(() => ID)
  id: string;

  @Field()
  name: string;

  @Field()
  email: string;

  @Field(() => String, { nullable: true })
  avatar?: string;

  @Field()
  createdAt: Date;

  // Password không expose ra GraphQL
  password: string;
}

@ObjectType()
export class Post {
  @Field(() => ID)
  id: string;

  @Field()
  title: string;

  @Field()
  content: string;

  @Field(() => Int, { defaultValue: 0 })
  viewCount: number;

  @Field()
  published: boolean;

  @Field(() => User)
  author: User;  // Nested type

  @Field()
  createdAt: Date;
}

@ObjectType()
export class PaginatedPosts {
  @Field(() => [Post])
  items: Post[];

  @Field(() => Int)
  total: number;

  @Field(() => Int)
  page: number;

  @Field(() => Int)
  limit: number;
}
```

## 4. Input Types — DTO cho mutation

```typescript
// src/users/dto/create-user.input.ts
import { InputType, Field } from '@nestjs/graphql';
import { IsEmail, IsString, MinLength } from 'class-validator';

@InputType()
export class CreateUserInput {
  @Field()
  @IsString()
  name: string;

  @Field()
  @IsEmail()
  email: string;

  @Field()
  @MinLength(6)
  password: string;
}

@InputType()
export class UpdateUserInput {
  @Field({ nullable: true })
  name?: string;

  @Field({ nullable: true })
  avatar?: string;
}

@InputType()
export class PostsArgs {
  @Field(() => Int, { defaultValue: 1 })
  page: number = 1;

  @Field(() => Int, { defaultValue: 10 })
  limit: number = 10;

  @Field({ nullable: true })
  search?: string;
}
```

## 5. Resolver — Query và Mutation

```typescript
// src/users/users.resolver.ts
import { Resolver, Query, Mutation, Args, ID, ResolveField, Parent } from '@nestjs/graphql';

@Resolver(() => User)
export class UsersResolver {
  constructor(
    private usersService: UsersService,
    private postsService: PostsService,
  ) {}

  // Query — đọc dữ liệu
  @Query(() => [User], { name: 'users' })
  async getUsers(): Promise<User[]> {
    return this.usersService.findAll();
  }

  @Query(() => User, { name: 'user' })
  async getUser(@Args('id', { type: () => ID }) id: string): Promise<User> {
    return this.usersService.findById(id);
  }

  // Mutation — thay đổi dữ liệu
  @Mutation(() => User)
  async createUser(@Args('input') input: CreateUserInput): Promise<User> {
    return this.usersService.create(input);
  }

  @Mutation(() => User)
  @UseGuards(GqlAuthGuard)  // Guard cho GraphQL
  async updateUser(
    @Args('id', { type: () => ID }) id: string,
    @Args('input') input: UpdateUserInput,
    @CurrentUser() user: User,
  ): Promise<User> {
    if (user.id !== id) throw new ForbiddenException();
    return this.usersService.update(id, input);
  }

  @Mutation(() => Boolean)
  @UseGuards(GqlAuthGuard)
  async deleteUser(@Args('id', { type: () => ID }) id: string): Promise<boolean> {
    await this.usersService.remove(id);
    return true;
  }

  // ResolveField — resolve nested field
  @ResolveField(() => [Post])
  async posts(@Parent() user: User): Promise<Post[]> {
    return this.postsService.findByUserId(user.id);
  }
}
```

```typescript
// src/posts/posts.resolver.ts
@Resolver(() => Post)
export class PostsResolver {
  constructor(private postsService: PostsService) {}

  @Query(() => PaginatedPosts)
  async posts(@Args() args: PostsArgs): Promise<PaginatedPosts> {
    return this.postsService.findAll(args);
  }

  @Query(() => Post)
  async post(@Args('id', { type: () => ID }) id: string): Promise<Post> {
    return this.postsService.findById(id);
  }

  @Mutation(() => Post)
  @UseGuards(GqlAuthGuard)
  async createPost(
    @Args('title') title: string,
    @Args('content') content: string,
    @CurrentUser() user: User,
  ): Promise<Post> {
    return this.postsService.create({ title, content, authorId: user.id });
  }

  // ResolveField — lấy author từ userId
  @ResolveField(() => User)
  async author(@Parent() post: Post): Promise<User> {
    return this.usersService.findById(post.authorId);
  }
}
```

## 6. Auth Guard cho GraphQL

```typescript
// src/auth/gql-auth.guard.ts
import { ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { GqlExecutionContext } from '@nestjs/graphql';

@Injectable()
export class GqlAuthGuard extends AuthGuard('jwt') {
  getRequest(context: ExecutionContext) {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req; // Lấy req từ GraphQL context
  }
}

// CurrentUser decorator
import { createParamDecorator } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (_, context: ExecutionContext) => {
    const ctx = GqlExecutionContext.create(context);
    return ctx.getContext().req.user;
  },
);
```

## 7. DataLoader — N+1 Problem

```typescript
// Không dùng DataLoader → N+1 queries
// 100 posts → 100 queries lấy author

// Dùng DataLoader → batch 100 userIds thành 1 query
import DataLoader from 'dataloader';

@Injectable()
export class UsersLoader {
  constructor(private usersService: UsersService) {}

  createLoader(): DataLoader<string, User> {
    return new DataLoader<string, User>(async (userIds: readonly string[]) => {
      const users = await this.usersService.findByIds([...userIds]);
      const userMap = new Map(users.map(u => [u.id, u]));
      return userIds.map(id => userMap.get(id) ?? new Error(`User ${id} not found`));
    });
  }
}

// Trong resolver
@ResolveField(() => User)
async author(@Parent() post: Post, @Context() ctx: any): Promise<User> {
  return ctx.usersLoader.load(post.authorId); // Tự động batch
}
```

## 8. Query thử trong Playground

```graphql
# Lấy danh sách users với posts
query {
  users {
    id
    name
    email
    posts {
      id
      title
      viewCount
    }
  }
}

# Mutation tạo user
mutation {
  createUser(input: {
    name: "Nguyen Van A"
    email: "a@example.com"
    password: "123456"
  }) {
    id
    name
    email
  }
}

# Query có pagination
query {
  posts(page: 1, limit: 5, search: "nestjs") {
    items {
      id
      title
      author {
        name
      }
    }
    total
    page
  }
}
```

## 9. Kết luận

- **Code-first**: Dùng TypeScript decorator → tự generate schema — phù hợp team TypeScript
- **`@ObjectType`/`@InputType`**: Định nghĩa response/request types
- **`@ResolveField`**: Resolve nested relations — kết hợp DataLoader tránh N+1
- **GqlAuthGuard**: Extend `AuthGuard` và override `getRequest()` để lấy req từ GraphQL context
- **Playground**: Môi trường test query tích hợp sẵn — disable trong production

GraphQL phù hợp khi: nhiều client khác nhau (mobile/web/third-party), schema phức tạp, cần flexible querying. REST vẫn tốt hơn cho simple CRUD API.
