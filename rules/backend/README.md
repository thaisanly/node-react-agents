# Backend Rules Documentation

This folder contains all documentation and guides for backend development using NestJS, PostgreSQL, and clean architecture patterns.

## Documentation Files

- **[backend-development-guide.md](./backend-development-guide.md)** - Complete guide for NestJS backend development with clean architecture
- **[repository-pattern-guide.md](./repository-pattern-guide.md)** - Detailed guide on implementing the Repository Pattern
- **[testing-guide.md](./testing-guide.md)** - Testing strategies and best practices for backend

## Quick Start

### Technology Stack

- **Framework**: NestJS
- **Database**: PostgreSQL
- **ORM**: Prisma
- **Language**: TypeScript (strict mode)
- **Validation**: class-validator, class-transformer
- **Testing**: Jest

### Project Setup

1. **Initialize NestJS Project**
   ```bash
   npm i -g @nestjs/cli
   nest new backend
   cd backend
   ```

2. **Install Dependencies**
   ```bash
   npm install @nestjs/config @prisma/client class-validator class-transformer
   npm install -D prisma
   ```

3. **Setup Prisma**
   ```bash
   npx prisma init
   ```

4. **Configure Database**
   - Update `.env` with your PostgreSQL connection string
   - Define your schema in `prisma/schema.prisma`
   - Run migrations: `npx prisma migrate dev`

5. **Project Structure**
   Follow the clean architecture structure defined in [backend-development-guide.md](./backend-development-guide.md)

## Key Principles

- **Clean Architecture**: Separation of concerns with Controllers, Services, and Repositories
- **Database Agnostic**: Services never import Prisma directly
- **Type Safety**: Strict TypeScript with domain types
- **Repository Pattern**: Data access abstraction for swappable implementations

## Architecture Overview

```
backend/
├── src/
│   ├── common/              # Shared utilities
│   ├── domain/              # Business logic layer
│   │   ├── entities/        # Domain entities (DB-agnostic)
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

## See Also

- [Frontend Documentation](../frontend/README.md)
- [Main Project README](../../README.md)
