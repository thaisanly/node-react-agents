# Testing Guide

## Overview
This guide defines testing standards for both backend (NestJS) and frontend (React) applications, including unit tests and integration/HTTP tests.

## Testing Principles

### Unit Tests
- **Purpose**: Test individual units of code in isolation
- **Scope**: Functions, methods, components
- **Dependencies**: All mocked (no real DB, API calls, external services)
- **Speed**: Fast (milliseconds)
- **Focus**: Logic, edge cases, error handling

### Integration/HTTP Tests (E2E)
- **Purpose**: Test complete request/response flow
- **Scope**: API endpoints, database operations, middleware
- **Dependencies**: Real database (test DB), real services
- **Speed**: Slower (seconds)
- **Focus**: Happy paths, error scenarios, validation

## Backend Testing (NestJS)

### Test Structure

```
backend/
├── src/
│   └── features/
│       └── users/
│           ├── services/
│           │   ├── users.service.ts
│           │   └── users.service.spec.ts      # Unit tests
│           ├── controllers/
│           │   ├── users.controller.ts
│           │   └── users.controller.spec.ts   # Unit tests
│           └── repositories/
│               └── user-repository.spec.ts    # Unit tests (if needed)
└── test/
    ├── unit/                                   # Additional unit tests
    └── e2e/
        └── users.e2e-spec.ts                   # HTTP/Integration tests
```

### Unit Tests Configuration

```typescript
// jest.config.js
module.exports = {
  moduleFileExtensions: ['js', 'json', 'ts'],
  rootDir: 'src',
  testRegex: '.*\\.spec\\.ts$',
  transform: {
    '^.+\\.(t|j)s$': 'ts-jest',
  },
  collectCoverageFrom: [
    '**/*.(t|j)s',
    '!**/*.spec.ts',
    '!**/*.e2e-spec.ts',
  ],
  coverageDirectory: '../coverage',
  testEnvironment: 'node',
  moduleNameMapper: {
    '^@/(.*)$': '<rootDir>/$1',
  },
};
```

### Unit Test: Service

```typescript
// features/users/services/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { NotFoundException, ConflictException } from '@nestjs/common';
import { UsersService } from './users.service';
import {
  UserRepositoryInterface,
  USER_REPOSITORY,
} from '@/domain/repositories/user-repository.interface';
import { User } from '@/domain/entities/user.entity';

describe('UsersService', () => {
  let service: UsersService;
  let repository: jest.Mocked<UserRepositoryInterface>;

  // Mock data
  const mockUser: User = {
    id: '1',
    email: 'test@example.com',
    name: 'Test User',
    password: 'hashedPassword',
    createdAt: new Date(),
    updatedAt: new Date(),
  };

  beforeEach(async () => {
    // Create mock repository
    const mockRepository: jest.Mocked<UserRepositoryInterface> = {
      findById: jest.fn(),
      findByEmail: jest.fn(),
      findAll: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
      count: jest.fn(),
      exists: jest.fn(),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        {
          provide: USER_REPOSITORY,
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get(USER_REPOSITORY);
  });

  afterEach(() => {
    jest.clearAllMocks();
  });

  describe('findById', () => {
    it('should return a user when found', async () => {
      repository.findById.mockResolvedValue(mockUser);

      const result = await service.findById('1');

      expect(result).toEqual(mockUser);
      expect(repository.findById).toHaveBeenCalledWith('1');
      expect(repository.findById).toHaveBeenCalledTimes(1);
    });

    it('should throw NotFoundException when user not found', async () => {
      repository.findById.mockResolvedValue(null);

      await expect(service.findById('999')).rejects.toThrow(NotFoundException);
      expect(repository.findById).toHaveBeenCalledWith('999');
    });
  });

  describe('create', () => {
    const createData = {
      email: 'new@example.com',
      name: 'New User',
      password: 'password123',
    };

    it('should create a new user successfully', async () => {
      repository.findByEmail.mockResolvedValue(null);
      repository.create.mockResolvedValue(mockUser);

      const result = await service.create(createData);

      expect(result).toEqual(mockUser);
      expect(repository.findByEmail).toHaveBeenCalledWith(createData.email);
      expect(repository.create).toHaveBeenCalled();
    });

    it('should throw ConflictException when email already exists', async () => {
      repository.findByEmail.mockResolvedValue(mockUser);

      await expect(service.create(createData)).rejects.toThrow(ConflictException);
      expect(repository.findByEmail).toHaveBeenCalledWith(createData.email);
      expect(repository.create).not.toHaveBeenCalled();
    });
  });

  describe('update', () => {
    const updateData = { name: 'Updated Name' };

    it('should update user successfully', async () => {
      const updatedUser = { ...mockUser, ...updateData };
      repository.findById.mockResolvedValue(mockUser);
      repository.update.mockResolvedValue(updatedUser);

      const result = await service.update('1', updateData);

      expect(result).toEqual(updatedUser);
      expect(repository.findById).toHaveBeenCalledWith('1');
      expect(repository.update).toHaveBeenCalledWith('1', updateData);
    });

    it('should throw NotFoundException when user not found', async () => {
      repository.findById.mockResolvedValue(null);

      await expect(service.update('999', updateData)).rejects.toThrow(NotFoundException);
      expect(repository.update).not.toHaveBeenCalled();
    });
  });

  describe('delete', () => {
    it('should delete user successfully', async () => {
      repository.findById.mockResolvedValue(mockUser);
      repository.delete.mockResolvedValue(undefined);

      await service.delete('1');

      expect(repository.findById).toHaveBeenCalledWith('1');
      expect(repository.delete).toHaveBeenCalledWith('1');
    });

    it('should throw NotFoundException when user not found', async () => {
      repository.findById.mockResolvedValue(null);

      await expect(service.delete('999')).rejects.toThrow(NotFoundException);
      expect(repository.delete).not.toHaveBeenCalled();
    });
  });

  describe('findAll', () => {
    it('should return paginated users', async () => {
      const mockUsers = [mockUser];
      repository.findAll.mockResolvedValue(mockUsers);
      repository.count.mockResolvedValue(1);

      const result = await service.findAll(1, 10);

      expect(result).toEqual({ users: mockUsers, total: 1 });
      expect(repository.findAll).toHaveBeenCalledWith(0, 10);
      expect(repository.count).toHaveBeenCalled();
    });
  });
});
```

### Unit Test: Controller

```typescript
// features/users/controllers/users.controller.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { UsersController } from './users.controller';
import { UsersService } from '../services/users.service';
import { CreateUserDto } from '../dto/create-user.dto';
import { User } from '@/domain/entities/user.entity';

describe('UsersController', () => {
  let controller: UsersController;
  let service: jest.Mocked<UsersService>;

  const mockUser: User = {
    id: '1',
    email: 'test@example.com',
    name: 'Test User',
    password: 'hashedPassword',
    createdAt: new Date(),
    updatedAt: new Date(),
  };

  beforeEach(async () => {
    const mockService = {
      findById: jest.fn(),
      findAll: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
    };

    const module: TestingModule = await Test.createTestingModule({
      controllers: [UsersController],
      providers: [
        {
          provide: UsersService,
          useValue: mockService,
        },
      ],
    }).compile();

    controller = module.get<UsersController>(UsersController);
    service = module.get(UsersService);
  });

  describe('create', () => {
    it('should create a user', async () => {
      const createDto: CreateUserDto = {
        email: 'test@example.com',
        name: 'Test User',
        password: 'password123',
      };

      service.create.mockResolvedValue(mockUser);

      const result = await controller.create(createDto);

      expect(result).toBeDefined();
      expect(service.create).toHaveBeenCalledWith(createDto);
    });
  });

  describe('findOne', () => {
    it('should return a user', async () => {
      service.findById.mockResolvedValue(mockUser);

      const result = await controller.findOne('1');

      expect(result).toBeDefined();
      expect(service.findById).toHaveBeenCalledWith('1');
    });
  });

  describe('findAll', () => {
    it('should return paginated users', async () => {
      service.findAll.mockResolvedValue({
        users: [mockUser],
        total: 1,
      });

      const result = await controller.findAll(1, 10);

      expect(result.users).toHaveLength(1);
      expect(result.total).toBe(1);
      expect(service.findAll).toHaveBeenCalledWith(1, 10);
    });
  });
});
```

### HTTP/E2E Tests

Configure E2E testing:

```typescript
// test/jest-e2e.json
{
  "moduleFileExtensions": ["js", "json", "ts"],
  "rootDir": ".",
  "testEnvironment": "node",
  "testRegex": ".e2e-spec.ts$",
  "transform": {
    "^.+\\.(t|j)s$": "ts-jest"
  },
  "moduleNameMapper": {
    "^@/(.*)$": "<rootDir>/../src/$1"
  }
}
```

Create test database helper:

```typescript
// test/helpers/database.helper.ts
import { PrismaClient } from '@prisma/client';

const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_TEST_URL,
    },
  },
});

export async function clearDatabase() {
  // Delete in order of dependencies (child tables first)
  await prisma.post.deleteMany();
  await prisma.user.deleteMany();
  // Add more tables as needed
}

export async function closeDatabase() {
  await prisma.$disconnect();
}

export { prisma };
```

E2E test example:

```typescript
// test/e2e/users.e2e-spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '@/app.module';
import { clearDatabase, closeDatabase, prisma } from '../helpers/database.helper';
import { ValidationExceptionFilter } from '@/common/filters/validation-exception.filter';

describe('Users (e2e)', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();

    // Apply same configuration as main.ts
    app.useGlobalPipes(
      new ValidationPipe({
        whitelist: true,
        forbidNonWhitelisted: true,
        transform: true,
      }),
    );
    app.useGlobalFilters(new ValidationExceptionFilter());

    await app.init();
  });

  beforeEach(async () => {
    // Clear database before each test
    await clearDatabase();
  });

  afterAll(async () => {
    await closeDatabase();
    await app.close();
  });

  describe('POST /users', () => {
    it('should create a new user', async () => {
      const createDto = {
        email: 'test@example.com',
        name: 'Test User',
        password: 'Password123',
      };

      const response = await request(app.getHttpServer())
        .post('/users')
        .send(createDto)
        .expect(201);

      expect(response.body).toMatchObject({
        id: expect.any(String),
        email: createDto.email,
        name: createDto.name,
        createdAt: expect.any(String),
        updatedAt: expect.any(String),
      });
      expect(response.body.password).toBeUndefined(); // Password should be excluded

      // Verify in database
      const user = await prisma.user.findUnique({
        where: { email: createDto.email },
      });
      expect(user).toBeDefined();
      expect(user?.name).toBe(createDto.name);
    });

    it('should return 422 for invalid email', async () => {
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'invalid-email',
          name: 'Test User',
          password: 'Password123',
        })
        .expect(422);

      expect(response.body).toMatchObject({
        message: 'Validation failed',
        errors: {
          email: expect.arrayContaining([
            expect.stringContaining('Invalid email'),
          ]),
        },
      });
    });

    it('should return 422 for duplicate email', async () => {
      const createDto = {
        email: 'duplicate@example.com',
        name: 'User One',
        password: 'Password123',
      };

      // Create first user
      await request(app.getHttpServer())
        .post('/users')
        .send(createDto)
        .expect(201);

      // Try to create duplicate
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({
          ...createDto,
          name: 'User Two',
        })
        .expect(422);

      expect(response.body).toMatchObject({
        message: 'Validation failed',
        errors: {
          email: ['Email is already in use'],
        },
      });
    });

    it('should return 422 for weak password', async () => {
      const response = await request(app.getHttpServer())
        .post('/users')
        .send({
          email: 'test@example.com',
          name: 'Test User',
          password: 'weak',
        })
        .expect(422);

      expect(response.body.errors.password).toBeDefined();
    });
  });

  describe('GET /users/:id', () => {
    it('should return a user by id', async () => {
      // Create user first
      const user = await prisma.user.create({
        data: {
          email: 'test@example.com',
          name: 'Test User',
          password: 'hashedPassword',
        },
      });

      const response = await request(app.getHttpServer())
        .get(`/users/${user.id}`)
        .expect(200);

      expect(response.body).toMatchObject({
        id: user.id,
        email: user.email,
        name: user.name,
      });
      expect(response.body.password).toBeUndefined();
    });

    it('should return 404 for non-existent user', async () => {
      await request(app.getHttpServer())
        .get('/users/non-existent-id')
        .expect(404);
    });
  });

  describe('GET /users', () => {
    it('should return paginated users', async () => {
      // Create multiple users
      await prisma.user.createMany({
        data: [
          { email: 'user1@example.com', name: 'User 1', password: 'pass' },
          { email: 'user2@example.com', name: 'User 2', password: 'pass' },
          { email: 'user3@example.com', name: 'User 3', password: 'pass' },
        ],
      });

      const response = await request(app.getHttpServer())
        .get('/users?page=1&limit=2')
        .expect(200);

      expect(response.body).toMatchObject({
        users: expect.arrayContaining([
          expect.objectContaining({
            email: expect.any(String),
            name: expect.any(String),
          }),
        ]),
        total: 3,
        page: 1,
        limit: 2,
      });
      expect(response.body.users).toHaveLength(2);
    });
  });

  describe('PUT /users/:id', () => {
    it('should update a user', async () => {
      const user = await prisma.user.create({
        data: {
          email: 'test@example.com',
          name: 'Original Name',
          password: 'hashedPassword',
        },
      });

      const updateDto = { name: 'Updated Name' };

      const response = await request(app.getHttpServer())
        .put(`/users/${user.id}`)
        .send(updateDto)
        .expect(200);

      expect(response.body.name).toBe(updateDto.name);

      // Verify in database
      const updatedUser = await prisma.user.findUnique({
        where: { id: user.id },
      });
      expect(updatedUser?.name).toBe(updateDto.name);
    });

    it('should return 404 when updating non-existent user', async () => {
      await request(app.getHttpServer())
        .put('/users/non-existent-id')
        .send({ name: 'New Name' })
        .expect(404);
    });
  });

  describe('DELETE /users/:id', () => {
    it('should delete a user', async () => {
      const user = await prisma.user.create({
        data: {
          email: 'test@example.com',
          name: 'Test User',
          password: 'hashedPassword',
        },
      });

      await request(app.getHttpServer())
        .delete(`/users/${user.id}`)
        .expect(204);

      // Verify user is deleted
      const deletedUser = await prisma.user.findUnique({
        where: { id: user.id },
      });
      expect(deletedUser).toBeNull();
    });

    it('should return 404 when deleting non-existent user', async () => {
      await request(app.getHttpServer())
        .delete('/users/non-existent-id')
        .expect(404);
    });
  });
});
```

### Running Tests

```json
// package.json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:cov": "jest --coverage",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "test:e2e:watch": "jest --config ./test/jest-e2e.json --watch"
  }
}
```

## Frontend Testing (React)

### Test Structure

```
frontend/
├── src/
│   ├── components/
│   │   └── Button.test.tsx
│   ├── features/
│   │   └── users/
│   │       ├── components/
│   │       │   └── CreateUserForm.test.tsx
│   │       ├── hooks/
│   │       │   └── useUser.test.ts
│   │       └── api/
│   │           └── users.api.test.ts
│   └── lib/
│       └── utils/
│           └── form-errors.test.ts
└── vitest.config.ts
```

### Vitest Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    globals: true,
    environment: 'jsdom',
    setupFiles: ['./src/test/setup.ts'],
    coverage: {
      reporter: ['text', 'json', 'html'],
      exclude: [
        'node_modules/',
        'src/test/',
      ],
    },
  },
  resolve: {
    alias: {
      '@': path.resolve(__dirname, './src'),
    },
  },
});
```

### Test Setup

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom';
import { cleanup } from '@testing-library/react';
import { afterEach } from 'vitest';

// Cleanup after each test
afterEach(() => {
  cleanup();
});

// Mock IntersectionObserver
global.IntersectionObserver = class IntersectionObserver {
  constructor() {}
  disconnect() {}
  observe() {}
  takeRecords() {
    return [];
  }
  unobserve() {}
};
```

### Component Tests

```typescript
// features/users/components/CreateUserForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { CreateUserForm } from './CreateUserForm';
import { createUser } from '../api/users.api';
import { BrowserRouter } from 'react-router-dom';

// Mock API
vi.mock('../api/users.api');

// Wrapper for router context
const Wrapper = ({ children }: { children: React.ReactNode }) => (
  <BrowserRouter>{children}</BrowserRouter>
);

describe('CreateUserForm', () => {
  beforeEach(() => {
    vi.clearAllMocks();
  });

  it('should render form fields', () => {
    render(<CreateUserForm />, { wrapper: Wrapper });

    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/name/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
    expect(screen.getByRole('button', { name: /create/i })).toBeInTheDocument();
  });

  it('should display validation errors on empty submit', async () => {
    const user = userEvent.setup();
    render(<CreateUserForm />, { wrapper: Wrapper });

    const submitButton = screen.getByRole('button', { name: /create/i });
    await user.click(submitButton);

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(await screen.findByText(/name must be at least 2 characters/i)).toBeInTheDocument();
  });

  it('should display validation error for invalid email', async () => {
    const user = userEvent.setup();
    render(<CreateUserForm />, { wrapper: Wrapper });

    const emailInput = screen.getByLabelText(/email/i);
    await user.type(emailInput, 'invalid-email');
    await user.tab(); // Trigger blur validation

    expect(await screen.findByText(/invalid email/i)).toBeInTheDocument();
  });

  it('should submit form with valid data', async () => {
    const user = userEvent.setup();
    const mockUser = {
      id: '1',
      email: 'test@example.com',
      name: 'Test User',
    };

    (createUser as any).mockResolvedValue(mockUser);

    render(<CreateUserForm />, { wrapper: Wrapper });

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/name/i), 'Test User');
    await user.type(screen.getByLabelText(/password/i), 'Password123');

    const submitButton = screen.getByRole('button', { name: /create/i });
    await user.click(submitButton);

    await waitFor(() => {
      expect(createUser).toHaveBeenCalledWith({
        email: 'test@example.com',
        name: 'Test User',
        password: 'Password123',
        title: '',
      });
    });
  });

  it('should display backend validation errors', async () => {
    const user = userEvent.setup();

    (createUser as any).mockRejectedValue({
      response: {
        status: 422,
        data: {
          message: 'Validation failed',
          errors: {
            email: ['Email is already in use'],
          },
        },
      },
    });

    render(<CreateUserForm />, { wrapper: Wrapper });

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/name/i), 'Test User');
    await user.type(screen.getByLabelText(/password/i), 'Password123');

    const submitButton = screen.getByRole('button', { name: /create/i });
    await user.click(submitButton);

    expect(await screen.findByText(/email is already in use/i)).toBeInTheDocument();
  });

  it('should disable submit button while submitting', async () => {
    const user = userEvent.setup();

    (createUser as any).mockImplementation(
      () => new Promise((resolve) => setTimeout(resolve, 100))
    );

    render(<CreateUserForm />, { wrapper: Wrapper });

    await user.type(screen.getByLabelText(/email/i), 'test@example.com');
    await user.type(screen.getByLabelText(/name/i), 'Test User');
    await user.type(screen.getByLabelText(/password/i), 'Password123');

    const submitButton = screen.getByRole('button', { name: /create/i });
    await user.click(submitButton);

    expect(submitButton).toBeDisabled();
    expect(screen.getByText(/creating/i)).toBeInTheDocument();
  });
});
```

### Hook Tests

```typescript
// hooks/useApiMutation.test.ts
import { renderHook, waitFor } from '@testing-library/react';
import { describe, it, expect, vi } from 'vitest';
import { useApiMutation } from './useApiMutation';

describe('useApiMutation', () => {
  it('should handle successful mutation', async () => {
    const mockFn = vi.fn().mockResolvedValue({ id: 1, name: 'Test' });
    const onSuccess = vi.fn();

    const { result } = renderHook(() =>
      useApiMutation(mockFn, { onSuccess })
    );

    await result.current.mutate({ name: 'Test' });

    await waitFor(() => {
      expect(result.current.data).toEqual({ id: 1, name: 'Test' });
      expect(result.current.isLoading).toBe(false);
      expect(result.current.error).toBeNull();
      expect(onSuccess).toHaveBeenCalled();
    });
  });

  it('should handle mutation error', async () => {
    const mockError = new Error('API Error');
    const mockFn = vi.fn().mockRejectedValue(mockError);
    const onError = vi.fn();

    const { result } = renderHook(() =>
      useApiMutation(mockFn, { onError })
    );

    try {
      await result.current.mutate({ name: 'Test' });
    } catch (error) {
      // Expected error
    }

    await waitFor(() => {
      expect(result.current.error).toEqual(mockError);
      expect(result.current.isLoading).toBe(false);
      expect(onError).toHaveBeenCalledWith(mockError);
    });
  });

  it('should handle backend validation errors', async () => {
    const mockError = {
      response: {
        status: 422,
        data: {
          message: 'Validation failed',
          errors: {
            email: ['Email is already in use'],
          },
        },
      },
    };

    const mockFn = vi.fn().mockRejectedValue(mockError);
    const setError = vi.fn();

    const { result } = renderHook(() =>
      useApiMutation(mockFn, { setError })
    );

    try {
      await result.current.mutate({ email: 'test@example.com' });
    } catch (error) {
      // Expected error
    }

    await waitFor(() => {
      expect(setError).toHaveBeenCalledWith('email', {
        type: 'server',
        message: 'Email is already in use',
      });
    });
  });
});
```

### Utility Tests

```typescript
// lib/utils/form-errors.test.ts
import { describe, it, expect, vi } from 'vitest';
import {
  handleBackendValidationErrors,
  isBackendValidationError,
} from './form-errors';

describe('form-errors utils', () => {
  describe('handleBackendValidationErrors', () => {
    it('should set errors for each field', () => {
      const setError = vi.fn();
      const errors = {
        email: ['Email is already in use'],
        password: ['Password is too weak', 'Password must contain numbers'],
      };

      handleBackendValidationErrors(errors, setError);

      expect(setError).toHaveBeenCalledWith('email', {
        type: 'server',
        message: 'Email is already in use',
      });

      expect(setError).toHaveBeenCalledWith('password', {
        type: 'server',
        message: 'Password is too weak',
      });

      expect(setError).toHaveBeenCalledTimes(2);
    });
  });

  describe('isBackendValidationError', () => {
    it('should return true for valid validation error', () => {
      const error = {
        response: {
          status: 422,
          data: {
            message: 'Validation failed',
            errors: {
              email: ['Invalid email'],
            },
          },
        },
      };

      expect(isBackendValidationError(error)).toBe(true);
    });

    it('should return false for non-422 error', () => {
      const error = {
        response: {
          status: 500,
          data: { message: 'Internal server error' },
        },
      };

      expect(isBackendValidationError(error)).toBe(false);
    });

    it('should return false for error without errors object', () => {
      const error = {
        response: {
          status: 422,
          data: { message: 'Validation failed' },
        },
      };

      expect(isBackendValidationError(error)).toBe(false);
    });
  });
});
```

### Running Frontend Tests

```json
// package.json
{
  "scripts": {
    "test": "vitest",
    "test:ui": "vitest --ui",
    "test:coverage": "vitest --coverage"
  }
}
```

## Test Coverage Goals

### Backend
- **Unit Tests**: > 80% coverage
  - Services: 100% (business logic must be fully tested)
  - Controllers: > 70%
  - Repositories: Optional (tested via integration tests)

- **E2E Tests**: All endpoints
  - Happy paths
  - Validation errors (422)
  - Not found errors (404)
  - Conflict errors (409)
  - Auth errors (401, 403)

### Frontend
- **Component Tests**: > 70% coverage
  - Render tests
  - User interaction tests
  - Form validation tests
  - Error handling tests

- **Hook Tests**: > 80% coverage
- **Utility Tests**: 100% coverage

## Testing Checklist

### Backend Unit Tests
- [ ] All service methods tested
- [ ] Repository is mocked
- [ ] No real database connections
- [ ] No external API calls
- [ ] Success cases tested
- [ ] Error cases tested (NotFoundException, ConflictException, etc.)
- [ ] Edge cases tested
- [ ] Mock data is realistic

### Backend E2E Tests
- [ ] Test database is configured
- [ ] Database is cleared before each test
- [ ] All CRUD endpoints tested
- [ ] Validation errors return 422 with correct format
- [ ] Authentication/authorization tested (if applicable)
- [ ] Response format matches expectations
- [ ] Database state is verified after operations
- [ ] Relationships are tested

### Frontend Component Tests
- [ ] Component renders correctly
- [ ] Form fields are present
- [ ] Client-side validation works
- [ ] Backend validation errors are displayed
- [ ] Submit button states (enabled/disabled/loading)
- [ ] Success scenarios
- [ ] Error scenarios
- [ ] User interactions (click, type, etc.)

### Frontend Hook Tests
- [ ] Hook returns expected values
- [ ] State updates correctly
- [ ] Side effects work (API calls, etc.)
- [ ] Error handling works
- [ ] Loading states are correct

## Best Practices

1. **Arrange-Act-Assert**: Structure tests with clear sections
2. **Descriptive test names**: `it('should throw NotFoundException when user not found')`
3. **Test behavior, not implementation**: Focus on what, not how
4. **Mock external dependencies**: Database, APIs, services
5. **Use realistic data**: Don't use overly simple test data
6. **Clean up**: Clear database, reset mocks between tests
7. **Test error cases**: Don't just test happy paths
8. **Keep tests independent**: Each test should work in isolation
9. **Fast tests**: Unit tests should run in milliseconds
10. **Maintainable**: Tests should be easy to understand and update

## Common Testing Patterns

### Testing with Authentication

```typescript
// test/helpers/auth.helper.ts
export async function createAuthenticatedUser(prisma: PrismaClient) {
  const user = await prisma.user.create({
    data: {
      email: 'auth@example.com',
      name: 'Auth User',
      password: 'hashedPassword',
    },
  });

  // Generate JWT token
  const token = jwt.sign({ userId: user.id }, process.env.JWT_SECRET);

  return { user, token };
}

// In E2E test
it('should require authentication', async () => {
  const { token } = await createAuthenticatedUser(prisma);

  await request(app.getHttpServer())
    .get('/users/profile')
    .set('Authorization', `Bearer ${token}`)
    .expect(200);
});
```

### Testing with Relations

```typescript
it('should include author in post response', async () => {
  const user = await prisma.user.create({
    data: { email: 'author@example.com', name: 'Author', password: 'pass' },
  });

  const post = await prisma.post.create({
    data: {
      title: 'Test Post',
      content: 'Content',
      authorId: user.id,
    },
  });

  const response = await request(app.getHttpServer())
    .get(`/posts/${post.id}`)
    .expect(200);

  expect(response.body.author).toMatchObject({
    id: user.id,
    name: user.name,
  });
});
```

### Testing Async Operations

```typescript
it('should handle async validation', async () => {
  // Create existing user
  await prisma.user.create({
    data: { email: 'existing@example.com', name: 'Existing', password: 'pass' },
  });

  // Try to create duplicate
  const response = await request(app.getHttpServer())
    .post('/users')
    .send({
      email: 'existing@example.com',
      name: 'New User',
      password: 'Password123',
    })
    .expect(422);

  expect(response.body.errors.email).toContain('Email is already in use');
});
```

## Continuous Integration

```yaml
# .github/workflows/test.yml
name: Tests

on: [push, pull_request]

jobs:
  backend-tests:
    runs-on: ubuntu-latest

    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        working-directory: ./backend
        run: npm ci

      - name: Run unit tests
        working-directory: ./backend
        run: npm test

      - name: Run e2e tests
        working-directory: ./backend
        run: npm run test:e2e
        env:
          DATABASE_TEST_URL: postgresql://postgres:postgres@localhost:5432/test

  frontend-tests:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: '18'

      - name: Install dependencies
        working-directory: ./frontend
        run: npm ci

      - name: Run tests
        working-directory: ./frontend
        run: npm test
```

See related guides:
- `backend-development-guide.md` for service and controller implementation
- `frontend-development-guide.md` for component implementation
- `repository-pattern-guide.md` for repository testing
