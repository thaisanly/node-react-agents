# Node React Rules - Full Stack Development Guides

Comprehensive documentation and development guides for building full-stack applications with Node.js (NestJS) backend and React frontend.

## 📁 Project Structure

```
node-react-rules/
├── rules/
│   ├── backend/          # Backend development documentation
│   │   ├── README.md
│   │   ├── backend-development-guide.md
│   │   ├── repository-pattern-guide.md
│   │   └── testing-guide.md
│   └── frontend/         # Frontend development documentation
│       ├── README.md
│       ├── frontend-development-guide.md
│       ├── form-validation-guide.md
│       └── playwright-e2e-guide.md
└── README.md
```

## 🎯 Overview

This repository contains structured documentation for developing modern full-stack web applications using:

### Backend Stack
- **Framework**: NestJS
- **Database**: PostgreSQL
- **ORM**: Prisma
- **Architecture**: Clean Architecture with Repository Pattern
- **Language**: TypeScript (strict mode)
- **Testing**: Jest

### Frontend Stack
- **Framework**: React 18+
- **Build Tool**: Vite
- **Language**: TypeScript (strict mode)
- **Styling**: Tailwind CSS
- **Form Handling**: React Hook Form + Zod
- **Data Fetching**: TanStack Query (React Query)
- **Client State**: React Context API
- **E2E Testing**: Playwright

## 📚 Documentation

### Backend Documentation

Navigate to [`rules/backend/`](./rules/backend/) for complete backend development guides:

- **[Backend Development Guide](./rules/backend/backend-development-guide.md)** - NestJS best practices, project structure, and clean architecture patterns
- **[Repository Pattern Guide](./rules/backend/repository-pattern-guide.md)** - Implementing data access layer abstraction
- **[Testing Guide](./rules/backend/testing-guide.md)** - Unit testing, integration testing, and E2E testing strategies

[View Backend README →](./rules/backend/README.md)

### Frontend Documentation

Navigate to [`rules/frontend/`](./rules/frontend/) for complete frontend development guides:

- **[Frontend Development Guide](./rules/frontend/frontend-development-guide.md)** - React, Vite, TypeScript, and component architecture
- **[Form Validation Guide](./rules/frontend/form-validation-guide.md)** - Form handling with Zod validation schemas
- **[Playwright E2E Guide](./rules/frontend/playwright-e2e-guide.md)** - End-to-end testing with Playwright

[View Frontend README →](./rules/frontend/README.md)

## 🚀 Quick Start

### Backend Setup

```bash
# Initialize NestJS project
npm i -g @nestjs/cli
nest new backend
cd backend

# Install dependencies
npm install @nestjs/config @prisma/client class-validator class-transformer
npm install -D prisma

# Setup Prisma
npx prisma init
```

See [Backend README](./rules/backend/README.md) for detailed setup instructions.

### Frontend Setup

```bash
# Initialize Vite + React project
npm create vite@latest frontend -- --template react-ts
cd frontend

# Install dependencies
npm install react-router-dom zod @hookform/resolvers react-hook-form @tanstack/react-query

# Setup Tailwind CSS
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p

# Setup Playwright
npm init playwright@latest
```

See [Frontend README](./rules/frontend/README.md) for detailed setup instructions.

## 🏗️ Architecture Principles

### Backend (Clean Architecture)

```
backend/
├── src/
│   ├── common/              # Shared utilities
│   ├── domain/              # Business logic layer
│   │   ├── entities/        # Domain entities
│   │   ├── repositories/    # Repository interfaces
│   │   └── types/          # Shared types
│   ├── infrastructure/      # External services
│   │   └── database/       # Database implementations
│   └── modules/            # Feature modules
│       └── [feature]/
│           ├── controllers/
│           ├── services/
│           └── repositories/
```

**Key Principles:**
- Services are database-agnostic
- Repository pattern for data access
- Dependency injection
- Strict TypeScript typing

### Frontend (Feature-based)

```
frontend/
├── src/
│   ├── components/          # Shared components
│   │   ├── ui/             # Base UI components
│   │   ├── forms/          # Form components
│   │   └── layouts/        # Layout components
│   ├── features/           # Feature modules
│   │   └── [feature]/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── schemas/    # Zod schemas
│   │       └── types/
│   ├── lib/                # Utilities
│   │   ├── api/           # API clients
│   │   └── utils/         # Helper functions
│   └── hooks/             # Shared hooks
└── e2e/                   # Playwright tests
    ├── pages/             # Page Object Models
    └── tests/             # Test specs
```

**Key Principles:**
- Feature-based code organization
- Type-safe forms with Zod
- Reusable component composition
- Comprehensive E2E testing

## 🧪 Testing Strategy

### Backend Testing
- **Unit Tests**: Test services and utilities in isolation
- **Integration Tests**: Test repository implementations with database
- **E2E Tests**: Test complete API workflows

### Frontend Testing
- **Component Tests**: Test React components with Vitest + Testing Library
- **E2E Tests**: Test user workflows with Playwright across browsers
- **Visual Regression**: Screenshot testing with Playwright

See the dedicated testing guides:
- [Backend Testing Guide](./rules/backend/testing-guide.md)
- [Frontend Playwright E2E Guide](./rules/frontend/playwright-e2e-guide.md)

## 📖 Key Concepts

### Clean Architecture
Separation of concerns with clear boundaries between layers, making code maintainable and testable.

### Repository Pattern
Abstraction over data access, allowing services to remain database-agnostic and easily testable.

### Type Safety
Strict TypeScript configuration with end-to-end type safety from database to frontend.

### Form Validation
Schema-based validation with Zod, providing both runtime validation and TypeScript type inference.

### E2E Testing
Comprehensive user workflow testing with Playwright, ensuring application reliability across browsers.

## 🔗 Full Stack Integration

### API Communication

**Backend (NestJS Controller):**
```typescript
@Post('users')
async createUser(@Body() createUserDto: CreateUserDto) {
  return this.userService.createUser(createUserDto);
}
```

**Frontend (TanStack Query Hook):**
```typescript
import { useMutation, useQueryClient } from '@tanstack/react-query';

const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: CreateUserSchema) => {
      const response = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      if (!response.ok) throw new Error('Failed to create user');
      return response.json();
    },
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
};
```

**E2E Test (Playwright):**
```typescript
test('should create user', async ({ page }) => {
  await page.goto('/users/new');
  await page.getByLabel('Name').fill('John Doe');
  await page.getByLabel('Email').fill('john@example.com');
  await page.getByRole('button', { name: 'Create' }).click();
  await expect(page.getByText('User created')).toBeVisible();
});
```

## 🛠️ Development Workflow

1. **Design API** - Define endpoints and DTOs in backend
2. **Implement Backend** - Controllers → Services → Repositories
3. **Test Backend** - Unit and integration tests
4. **Create Zod Schemas** - Define validation schemas in frontend
5. **Build UI Components** - Create forms and views
6. **Write E2E Tests** - Test complete user workflows with Playwright
7. **Integration Testing** - Test full stack together

## 📝 Contributing

When adding new guides or updating existing documentation:

1. Follow the existing structure and formatting
2. Include practical code examples
3. Add table of contents for long documents
4. Link related documentation
5. Keep examples up-to-date with latest versions

## 📄 License

MIT

## 🤝 Support

For questions or issues with the documentation, please open an issue in the repository.

---

**Happy Coding! 🚀**
