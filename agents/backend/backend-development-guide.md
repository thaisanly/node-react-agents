# Backend Development Guide

## Overview
This guide defines the standards and patterns for backend development using NestJS, PostgreSQL, and the Repository Pattern with a clean architecture approach.

## Technology Stack

- **Framework**: NestJS
- **API Type**: REST API
- **Database**: PostgreSQL (primary)
- **ORM**: Prisma
- **Language**: TypeScript (strict mode)
- **Validation**: class-validator, class-transformer
- **Testing**: Jest

## Architecture Principles

### Clean Architecture Layers

1. **Controllers**: Handle HTTP requests/responses
2. **Services**: Business logic (database-agnostic)
3. **Repositories**: Data access layer
4. **Domain**: Types, interfaces, entities (shared across layers)

### Key Rules

- Services NEVER import Prisma directly
- Services ONLY work with domain types
- Repositories handle database-specific operations
- Database implementations are swappable through repositories
- Each feature has its own repository implementation

## Project Structure

```
backend/
├── src/
│   ├── common/
│   │   ├── decorators/
│   │   ├── filters/
│   │   ├── guards/
│   │   ├── interceptors/
│   │   └── pipes/
│   ├── domain/
│   │   ├── entities/          # Domain entities (DB-agnostic)
│   │   ├── repositories/      # Repository interfaces
│   │   └── types/             # Shared types
│   ├── infrastructure/
│   │   ├── database/
│   │   │   ├── prisma/
│   │   │   │   ├── schema.prisma
│   │   │   │   └── migrations/
│   │   │   └── repositories/  # Prisma repository implementations
│   │   │       ├── prisma-user-repository.ts
│   │   │       ├── prisma-post-repository.ts
│   │   │       └── ...
│   │   └── config/
│   ├── features/
│   │   └── [feature-name]/
│   │       ├── controllers/
│   │       ├── services/
│   │       ├── dto/
│   │       └── [feature-name].module.ts
│   ├── app.module.ts
│   └── main.ts
├── test/
│   ├── unit/
│   └── e2e/
├── prisma/
│   └── schema.prisma
└── package.json
```

## Development Standards

### 1. Module Structure

Each feature should be organized as a NestJS module:

```typescript
// features/users/users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './controllers/users.controller';
import { UsersService } from './services/users.service';
import { PrismaUserRepository } from '@/infrastructure/database/repositories/prisma-user-repository';
import { PrismaService } from '@/infrastructure/database/prisma/prisma.service';

@Module({
  controllers: [UsersController],
  providers: [
    UsersService,
    PrismaService,
    {
      provide: 'UserRepository', // Use token for DI
      useClass: PrismaUserRepository,
    },
  ],
  exports: [UsersService],
})
export class UsersModule {}
```

### 2. Domain Layer

Define database-agnostic domain entities and repository interfaces:

```typescript
// domain/entities/user.entity.ts
export class User {
  id: string;
  email: string;
  name: string;
  password: string;
  createdAt: Date;
  updatedAt: Date;
}

export class CreateUserData {
  email: string;
  name: string;
  password: string;
}

export class UpdateUserData {
  email?: string;
  name?: string;
  password?: string;
}
```

```typescript
// domain/repositories/user-repository.interface.ts
import { User, CreateUserData, UpdateUserData } from '../entities/user.entity';

export interface UserRepositoryInterface {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  findAll(skip?: number, take?: number): Promise<User[]>;
  create(data: CreateUserData): Promise<User>;
  update(id: string, data: UpdateUserData): Promise<User>;
  delete(id: string): Promise<void>;
  count(): Promise<number>;
}

// Use const token for dependency injection
export const USER_REPOSITORY = 'UserRepository';
```

### 3. Repository Implementation

Implement repositories with database-specific logic (Prisma example):

```typescript
// infrastructure/database/repositories/prisma-user-repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { UserRepositoryInterface } from '@/domain/repositories/user-repository.interface';
import { User, CreateUserData, UpdateUserData } from '@/domain/entities/user.entity';

@Injectable()
export class PrismaUserRepository implements UserRepositoryInterface {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<User | null> {
    const user = await this.prisma.user.findUnique({
      where: { id },
    });
    return user ? this.toDomain(user) : null;
  }

  async findByEmail(email: string): Promise<User | null> {
    const user = await this.prisma.user.findUnique({
      where: { email },
    });
    return user ? this.toDomain(user) : null;
  }

  async findAll(skip: number = 0, take: number = 10): Promise<User[]> {
    const users = await this.prisma.user.findMany({
      skip,
      take,
      orderBy: { createdAt: 'desc' },
    });
    return users.map(this.toDomain);
  }

  async create(data: CreateUserData): Promise<User> {
    const user = await this.prisma.user.create({
      data,
    });
    return this.toDomain(user);
  }

  async update(id: string, data: UpdateUserData): Promise<User> {
    const user = await this.prisma.user.update({
      where: { id },
      data,
    });
    return this.toDomain(user);
  }

  async delete(id: string): Promise<void> {
    await this.prisma.user.delete({
      where: { id },
    });
  }

  async count(): Promise<number> {
    return this.prisma.user.count();
  }

  // Map Prisma model to domain entity
  private toDomain(prismaUser: any): User {
    return {
      id: prismaUser.id,
      email: prismaUser.email,
      name: prismaUser.name,
      password: prismaUser.password,
      createdAt: prismaUser.createdAt,
      updatedAt: prismaUser.updatedAt,
    };
  }
}
```

### 4. Service Layer

Services contain business logic and use repository interfaces:

```typescript
// features/users/services/users.service.ts
import { Injectable, Inject, NotFoundException, ConflictException } from '@nestjs/common';
import { UserRepositoryInterface, USER_REPOSITORY } from '@/domain/repositories/user-repository.interface';
import { User, CreateUserData, UpdateUserData } from '@/domain/entities/user.entity';
import * as bcrypt from 'bcrypt';

@Injectable()
export class UsersService {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: UserRepositoryInterface,
  ) {}

  async findById(id: string): Promise<User> {
    const user = await this.userRepository.findById(id);
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.userRepository.findByEmail(email);
  }

  async findAll(page: number = 1, limit: number = 10): Promise<{ users: User[]; total: number }> {
    const skip = (page - 1) * limit;
    const [users, total] = await Promise.all([
      this.userRepository.findAll(skip, limit),
      this.userRepository.count(),
    ]);
    return { users, total };
  }

  async create(data: CreateUserData): Promise<User> {
    // Check if user already exists
    const existingUser = await this.userRepository.findByEmail(data.email);
    if (existingUser) {
      throw new ConflictException('Email already in use');
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(data.password, 10);

    return this.userRepository.create({
      ...data,
      password: hashedPassword,
    });
  }

  async update(id: string, data: UpdateUserData): Promise<User> {
    // Verify user exists
    await this.findById(id);

    // Check email uniqueness if updating email
    if (data.email) {
      const existingUser = await this.userRepository.findByEmail(data.email);
      if (existingUser && existingUser.id !== id) {
        throw new ConflictException('Email already in use');
      }
    }

    // Hash password if updating
    if (data.password) {
      data.password = await bcrypt.hash(data.password, 10);
    }

    return this.userRepository.update(id, data);
  }

  async delete(id: string): Promise<void> {
    // Verify user exists
    await this.findById(id);
    await this.userRepository.delete(id);
  }
}
```

### 5. DTOs (Data Transfer Objects)

Define DTOs for request validation and response formatting:

```typescript
// features/users/dto/create-user.dto.ts
import { IsEmail, IsString, MinLength, MaxLength } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'user@example.com' })
  @IsEmail({}, { message: 'Invalid email address' })
  email: string;

  @ApiProperty({ example: 'John Doe' })
  @IsString()
  @MinLength(2, { message: 'Name must be at least 2 characters' })
  @MaxLength(100, { message: 'Name is too long' })
  name: string;

  @ApiProperty({ example: 'SecurePass123' })
  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @MaxLength(100, { message: 'Password is too long' })
  password: string;
}
```

```typescript
// features/users/dto/update-user.dto.ts
import { IsEmail, IsString, MinLength, MaxLength, IsOptional } from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class UpdateUserDto {
  @ApiProperty({ example: 'user@example.com', required: false })
  @IsOptional()
  @IsEmail({}, { message: 'Invalid email address' })
  email?: string;

  @ApiProperty({ example: 'John Doe', required: false })
  @IsOptional()
  @IsString()
  @MinLength(2, { message: 'Name must be at least 2 characters' })
  @MaxLength(100, { message: 'Name is too long' })
  name?: string;

  @ApiProperty({ example: 'NewSecurePass123', required: false })
  @IsOptional()
  @IsString()
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @MaxLength(100, { message: 'Password is too long' })
  password?: string;
}
```

```typescript
// features/users/dto/user-response.dto.ts
import { Exclude } from 'class-transformer';
import { ApiProperty } from '@nestjs/swagger';

export class UserResponseDto {
  @ApiProperty()
  id: string;

  @ApiProperty()
  email: string;

  @ApiProperty()
  name: string;

  @Exclude() // Never expose password
  password: string;

  @ApiProperty()
  createdAt: Date;

  @ApiProperty()
  updatedAt: Date;
}
```

### 6. Controllers

Controllers handle HTTP requests and delegate to services:

```typescript
// features/users/controllers/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
  UseInterceptors,
  ClassSerializerInterceptor,
} from '@nestjs/common';
import { ApiTags, ApiOperation, ApiResponse } from '@nestjs/swagger';
import { UsersService } from '../services/users.service';
import { CreateUserDto } from '../dto/create-user.dto';
import { UpdateUserDto } from '../dto/update-user.dto';
import { UserResponseDto } from '../dto/user-response.dto';
import { plainToInstance } from 'class-transformer';

@ApiTags('users')
@Controller('users')
@UseInterceptors(ClassSerializerInterceptor) // Exclude sensitive fields
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  @HttpCode(HttpStatus.CREATED)
  @ApiOperation({ summary: 'Create a new user' })
  @ApiResponse({ status: 201, description: 'User created', type: UserResponseDto })
  @ApiResponse({ status: 422, description: 'Validation error' })
  async create(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
    const user = await this.usersService.create(createUserDto);
    return plainToInstance(UserResponseDto, user);
  }

  @Get()
  @ApiOperation({ summary: 'Get all users' })
  @ApiResponse({ status: 200, description: 'Users retrieved', type: [UserResponseDto] })
  async findAll(
    @Query('page') page: number = 1,
    @Query('limit') limit: number = 10,
  ): Promise<{ users: UserResponseDto[]; total: number; page: number; limit: number }> {
    const { users, total } = await this.usersService.findAll(Number(page), Number(limit));
    return {
      users: plainToInstance(UserResponseDto, users),
      total,
      page: Number(page),
      limit: Number(limit),
    };
  }

  @Get(':id')
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiResponse({ status: 200, description: 'User retrieved', type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  async findOne(@Param('id') id: string): Promise<UserResponseDto> {
    const user = await this.usersService.findById(id);
    return plainToInstance(UserResponseDto, user);
  }

  @Put(':id')
  @ApiOperation({ summary: 'Update user' })
  @ApiResponse({ status: 200, description: 'User updated', type: UserResponseDto })
  @ApiResponse({ status: 404, description: 'User not found' })
  @ApiResponse({ status: 422, description: 'Validation error' })
  async update(
    @Param('id') id: string,
    @Body() updateUserDto: UpdateUserDto,
  ): Promise<UserResponseDto> {
    const user = await this.usersService.update(id, updateUserDto);
    return plainToInstance(UserResponseDto, user);
  }

  @Delete(':id')
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Delete user' })
  @ApiResponse({ status: 204, description: 'User deleted' })
  @ApiResponse({ status: 404, description: 'User not found' })
  async remove(@Param('id') id: string): Promise<void> {
    await this.usersService.delete(id);
  }
}
```

### 7. Prisma Setup

```typescript
// infrastructure/database/prisma/prisma.service.ts
import { Injectable, OnModuleInit, OnModuleDestroy } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

@Injectable()
export class PrismaService extends PrismaClient implements OnModuleInit, OnModuleDestroy {
  async onModuleInit() {
    await this.$connect();
  }

  async onModuleDestroy() {
    await this.$disconnect();
  }
}
```

```prisma
// prisma/schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  email     String   @unique
  name      String
  password  String
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  @@map("users")
}
```

### 8. Environment Configuration

```typescript
// infrastructure/config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  url: process.env.DATABASE_URL,
  type: process.env.DB_TYPE || 'postgresql',
}));
```

```env
# .env
DATABASE_URL="postgresql://user:password@localhost:5432/mydb?schema=public"
NODE_ENV=development
PORT=3000
```

### 9. Exception Handling

See `form-validation-guide.md` for detailed validation error handling.

## Checklist for New Features

When implementing a new feature:

- [ ] Define domain entities in `domain/entities/`
- [ ] Create repository interface in `domain/repositories/`
- [ ] Implement repository in `infrastructure/database/repositories/prisma-[feature]-repository.ts`
- [ ] Create DTOs for request/response in feature's `dto/` directory
- [ ] Implement service with business logic (inject repository interface)
- [ ] Create controller with proper HTTP methods and status codes
- [ ] Add validation decorators to DTOs
- [ ] Define Prisma schema models
- [ ] Register providers in feature module using DI tokens
- [ ] Handle validation errors with proper 422 responses
- [ ] Add API documentation with Swagger decorators
- [ ] Write unit tests for service (with mocked repository)
- [ ] Write e2e tests for endpoints

## Best Practices

1. **Dependency Injection**: Always inject repository interfaces, not concrete implementations
2. **Single Responsibility**: Each service method should do one thing
3. **Error Handling**: Use NestJS built-in exceptions (NotFoundException, ConflictException, etc.)
4. **Password Security**: Always hash passwords with bcrypt
5. **Validation**: Use class-validator decorators in DTOs
6. **Response Transformation**: Use class-transformer to exclude sensitive fields
7. **API Documentation**: Document all endpoints with Swagger decorators
8. **Database Transactions**: Use Prisma transactions for multi-step operations
9. **Logging**: Use NestJS Logger for consistent logging
10. **Configuration**: Use @nestjs/config for environment variables

## Database Migration

```bash
# Create migration
npx prisma migrate dev --name add_user_table

# Run migrations
npx prisma migrate deploy

# Generate Prisma Client
npx prisma generate
```

## Common Patterns

### Pagination
```typescript
interface PaginationParams {
  page: number;
  limit: number;
}

interface PaginatedResponse<T> {
  data: T[];
  total: number;
  page: number;
  limit: number;
  totalPages: number;
}
```

### Soft Delete
```typescript
// Add to domain entity
deletedAt?: Date | null;

// Repository method
async softDelete(id: string): Promise<void> {
  await this.prisma.user.update({
    where: { id },
    data: { deletedAt: new Date() },
  });
}
```

### Transactions
```typescript
async transferData(fromId: string, toId: string): Promise<void> {
  await this.prisma.$transaction(async (prisma) => {
    await prisma.user.update({ where: { id: fromId }, data: { /* ... */ } });
    await prisma.user.update({ where: { id: toId }, data: { /* ... */ } });
  });
}
```

See related guides:
- `repository-pattern-guide.md` for detailed repository implementation
- `form-validation-guide.md` for validation error handling
- `testing-guide.md` for testing strategies
