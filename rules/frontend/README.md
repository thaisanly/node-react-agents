# Frontend Rules Documentation

This folder contains all documentation and guides for frontend development using React, Vite, TypeScript, Zod, and Tailwind CSS, including end-to-end testing with Playwright.

## Documentation Files

- **[frontend-development-guide.md](./frontend-development-guide.md)** - Complete guide for React + Vite + TypeScript development
- **[form-validation-guide.md](./form-validation-guide.md)** - Form handling and validation with Zod
- **[playwright-e2e-guide.md](./playwright-e2e-guide.md)** - End-to-end testing with Playwright

## Quick Start

### Technology Stack

- **Framework**: React 18+
- **Build Tool**: Vite
- **Language**: TypeScript (strict mode)
- **Form Validation**: Zod with react-hook-form
- **Styling**: Tailwind CSS
- **UI Components**: shadcn/ui
- **State Management**: React Context API / Zustand
- **E2E Testing**: Playwright

### Project Setup

1. **Initialize Vite + React Project**
   ```bash
   npm create vite@latest frontend -- --template react-ts
   cd frontend
   ```

2. **Install Core Dependencies**
   ```bash
   npm install react-router-dom zod @hookform/resolvers react-hook-form zustand
   ```

3. **Setup Tailwind CSS**
   ```bash
   npm install -D tailwindcss postcss autoprefixer
   npx tailwindcss init -p
   ```

4. **Setup Playwright for E2E Testing**
   ```bash
   npm init playwright@latest
   ```

   See [playwright-e2e-guide.md](./playwright-e2e-guide.md) for detailed setup and testing patterns.

5. **Project Structure**
   Follow the structure defined in [frontend-development-guide.md](./frontend-development-guide.md)

## Key Principles

- **Type Safety**: Strict TypeScript with proper typing
- **Form Validation**: Zod schemas with type inference
- **Component Composition**: Reusable, testable components
- **Feature-based Organization**: Code organized by features
- **E2E Testing**: Comprehensive Playwright tests for user workflows

## Architecture Overview

```
frontend/
├── src/
│   ├── components/
│   │   ├── ui/              # shadcn/ui components
│   │   ├── forms/           # Form components
│   │   └── layouts/         # Layout components
│   ├── features/            # Feature-based modules
│   │   └── [feature-name]/
│   │       ├── components/  # Feature-specific components
│   │       ├── hooks/       # Feature-specific hooks
│   │       ├── schemas/     # Zod validation schemas
│   │       └── types/       # TypeScript types
│   ├── lib/
│   │   ├── api/             # API client functions
│   │   └── utils/           # Utility functions
│   ├── hooks/               # Shared custom hooks
│   └── types/               # Shared TypeScript types
├── e2e/                     # Playwright E2E tests
│   ├── fixtures/            # Test fixtures
│   ├── pages/               # Page Object Models
│   └── tests/               # Test files
└── playwright.config.ts     # Playwright configuration
```

## Testing Strategy

### Unit Tests (Vitest + Testing Library)
- Component testing
- Hook testing
- Utility function testing

### E2E Tests (Playwright)
- User workflow testing
- Cross-browser testing
- Mobile viewport testing
- Visual regression testing

See [playwright-e2e-guide.md](./playwright-e2e-guide.md) for comprehensive E2E testing documentation.

## See Also

- [Backend Documentation](../backend/README.md)
- [Main Project README](../../README.md)
