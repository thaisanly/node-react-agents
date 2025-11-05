# Frontend Rules Documentation

This folder contains all documentation and guides for frontend development using React, Vite, TypeScript, Zod, TanStack Query, and Tailwind CSS, including end-to-end testing with Playwright.

## ⚠️ MANDATORY REQUIREMENTS

Before starting development, understand these critical requirements:

1. **E2E Testing**: All Playwright tests MUST use Page Object Model (POM)
   - Direct page interactions in test files are prohibited
   - See [Playwright E2E Guide](./playwright-e2e-guide.md) for details

2. **Modal & Dropdown Management**: All modals and dropdowns MUST support ESC key with modal stack
   - ESC key must close the last opened modal/dropdown (LIFO)
   - All select dropdowns must register in the modal stack
   - See [Modal Stack Guide](./modal-stack-guide.md) for implementation

## Documentation Files

- **[frontend-development-guide.md](./frontend-development-guide.md)** - Complete guide for React + Vite + TypeScript development
- **[form-validation-guide.md](./form-validation-guide.md)** - Form handling and validation with Zod
- **[playwright-e2e-guide.md](./playwright-e2e-guide.md)** - End-to-end testing with Playwright (Page Object Model required)
- **[modal-stack-guide.md](./modal-stack-guide.md)** - Modal and dropdown ESC key handling with stack management

## Quick Start

### Technology Stack

- **Framework**: React 18+
- **Build Tool**: Vite
- **Language**: TypeScript (strict mode)
- **Form Validation**: Zod with react-hook-form
- **Styling**: Tailwind CSS
- **UI Components**: shadcn/ui
- **Data Fetching**: TanStack Query (React Query)
- **Client State**: React Context API (for simple state)
- **E2E Testing**: Playwright

### Project Setup

1. **Initialize Vite + React Project**
   ```bash
   npm create vite@latest frontend -- --template react-ts
   cd frontend
   ```

2. **Install Core Dependencies**
   ```bash
   npm install react-router-dom zod @hookform/resolvers react-hook-form @tanstack/react-query
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
- **Server State Management**: TanStack Query for API data fetching, caching, and synchronization
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

## Data Fetching with TanStack Query

TanStack Query (formerly React Query) is the recommended solution for managing server state:

### Key Features
- **Automatic Caching**: Intelligent caching and cache invalidation
- **Background Refetching**: Keep data fresh automatically
- **Optimistic Updates**: Instant UI updates before server confirmation
- **Request Deduplication**: Automatic deduplication of requests
- **Pagination & Infinite Scroll**: Built-in support
- **DevTools**: Excellent debugging experience

### Basic Setup

```typescript
// src/main.tsx
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';

const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 1000 * 60 * 5, // 5 minutes
      retry: 1,
    },
  },
});

ReactDOM.createRoot(document.getElementById('root')!).render(
  <QueryClientProvider client={queryClient}>
    <App />
  </QueryClientProvider>
);
```

### Usage Example

```typescript
// src/features/users/api/useUsers.ts
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export const useUsers = () => {
  return useQuery({
    queryKey: ['users'],
    queryFn: async () => {
      const response = await fetch('/api/users');
      if (!response.ok) throw new Error('Failed to fetch users');
      return response.json();
    },
  });
};

export const useCreateUser = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: CreateUserData) => {
      const response = await fetch('/api/users', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data),
      });
      if (!response.ok) throw new Error('Failed to create user');
      return response.json();
    },
    onSuccess: () => {
      // Invalidate and refetch users list
      queryClient.invalidateQueries({ queryKey: ['users'] });
    },
  });
};
```

## Testing Strategy

### Unit Tests (Vitest + Testing Library)
- Component testing
- Hook testing
- Utility function testing
- TanStack Query hooks testing

### E2E Tests (Playwright)
- User workflow testing
- Cross-browser testing
- Mobile viewport testing
- Visual regression testing

See [playwright-e2e-guide.md](./playwright-e2e-guide.md) for comprehensive E2E testing documentation.

## See Also

- [Backend Documentation](../backend/README.md)
- [Main Project README](../../README.md)
