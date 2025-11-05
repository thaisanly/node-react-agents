# Frontend Development Guide

## Overview
This guide defines the standards and patterns for frontend development using React, Vite, TypeScript, Zod, Tailwind CSS, and shadcn/ui components.

## Technology Stack

- **Framework**: React 18+
- **Build Tool**: Vite
- **Language**: TypeScript (strict mode)
- **Form Validation**: Zod
- **Styling**: Tailwind CSS
- **UI Components**: shadcn/ui
- **State Management**: React Context API / Zustand (for complex state)

## Project Structure

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
│   │   ├── utils/           # Utility functions
│   │   └── validations/     # Shared validation schemas
│   ├── hooks/               # Shared custom hooks
│   ├── types/               # Shared TypeScript types
│   └── App.tsx
├── tailwind.config.js
├── tsconfig.json
└── vite.config.ts
```

## Development Standards

### 1. TypeScript Configuration

Always use strict TypeScript configuration:

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "strictFunctionTypes": true
  }
}
```

### 2. Component Development

#### Functional Components
Use functional components with TypeScript:

```typescript
import { FC } from 'react';

interface UserProfileProps {
  userId: string;
  onUpdate?: (user: User) => void;
}

export const UserProfile: FC<UserProfileProps> = ({ userId, onUpdate }) => {
  // Component logic
  return <div>...</div>;
};
```

#### Component Organization
- One component per file
- Co-locate related components in feature directories
- Export components using named exports
- Use index.ts for clean imports

### 3. Form Validation with Zod

#### Define Zod Schemas

Create validation schemas in `schemas/` directory:

```typescript
// features/auth/schemas/login.schema.ts
import { z } from 'zod';

export const loginSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email address'),
  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .max(100, 'Password is too long'),
});

export type LoginFormData = z.infer<typeof loginSchema>;
```

#### Form Implementation with React Hook Form

```typescript
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { loginSchema, LoginFormData } from './schemas/login.schema';

export const LoginForm: FC = () => {
  const {
    register,
    handleSubmit,
    formState: { errors },
    setError,
  } = useForm<LoginFormData>({
    resolver: zodResolver(loginSchema),
  });

  const onSubmit = async (data: LoginFormData) => {
    try {
      await loginUser(data);
    } catch (error) {
      // Handle backend validation errors (422)
      if (error.status === 422) {
        handleBackendValidationErrors(error.data.errors, setError);
      }
    }
  };

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <div>
        <input {...register('email')} />
        {errors.email && <span>{errors.email.message}</span>}
      </div>
      {/* More fields */}
    </form>
  );
};
```

### 4. Backend Validation Error Handling

Backend returns 422 errors in this format:
```json
{
  "message": "Validation failed",
  "errors": {
    "email": ["Email is already in use", "Email format is invalid"],
    "password": ["Password is too weak"]
  }
}
```

Handle these errors and map them to form fields:

```typescript
// lib/utils/form-errors.ts
import { UseFormSetError, FieldValues, Path } from 'react-hook-form';

export interface BackendValidationErrors {
  [key: string]: string[];
}

export interface BackendErrorResponse {
  message: string;
  errors: BackendValidationErrors;
}

export function handleBackendValidationErrors<T extends FieldValues>(
  errors: BackendValidationErrors,
  setError: UseFormSetError<T>
) {
  Object.entries(errors).forEach(([field, messages]) => {
    setError(field as Path<T>, {
      type: 'server',
      message: messages[0], // Show first error message
    });
  });
}
```

### 5. API Client Setup

Create typed API client functions:

```typescript
// lib/api/client.ts
import axios, { AxiosError } from 'axios';
import { BackendErrorResponse } from '../utils/form-errors';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:3000',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add interceptors for auth tokens
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Type-safe error handling
export function isBackendValidationError(
  error: unknown
): error is AxiosError<BackendErrorResponse> {
  return (
    axios.isAxiosError(error) &&
    error.response?.status === 422 &&
    typeof error.response?.data === 'object' &&
    'errors' in error.response.data
  );
}
```

```typescript
// lib/api/auth.api.ts
import { apiClient } from './client';
import { LoginFormData } from '@/features/auth/schemas/login.schema';

export interface LoginResponse {
  token: string;
  user: {
    id: string;
    email: string;
    name: string;
  };
}

export async function loginUser(data: LoginFormData): Promise<LoginResponse> {
  const response = await apiClient.post<LoginResponse>('/auth/login', data);
  return response.data;
}
```

### 6. shadcn/ui Components

#### Installation
```bash
npx shadcn-ui@latest init
npx shadcn-ui@latest add button input form label
```

#### Usage with Forms

```typescript
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Label } from '@/components/ui/label';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';

export const UserForm: FC = () => {
  const form = useForm<FormData>({
    resolver: zodResolver(schema),
  });

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input placeholder="email@example.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />
        <Button type="submit">Submit</Button>
      </form>
    </Form>
  );
};
```

### 7. Tailwind CSS Standards

#### Configuration
```javascript
// tailwind.config.js
module.exports = {
  darkMode: ['class'],
  content: ['./index.html', './src/**/*.{ts,tsx}'],
  theme: {
    extend: {
      colors: {
        // Use CSS variables for theming
      },
    },
  },
  plugins: [require('tailwindcss-animate')],
};
```

#### Best Practices
- Use Tailwind utility classes for styling
- Create custom components in shadcn/ui style for reusable elements
- Use `cn()` utility for conditional classes:

```typescript
import { cn } from '@/lib/utils';

<div className={cn(
  'base-class',
  isActive && 'active-class',
  className // Allow override from props
)} />
```

### 8. Custom Hooks

Create reusable hooks for common patterns:

```typescript
// hooks/useApiMutation.ts
import { useState } from 'react';
import { isBackendValidationError } from '@/lib/api/client';
import { UseFormSetError, FieldValues } from 'react-hook-form';
import { handleBackendValidationErrors } from '@/lib/utils/form-errors';

interface UseApiMutationOptions<T extends FieldValues> {
  setError?: UseFormSetError<T>;
  onSuccess?: () => void;
  onError?: (error: unknown) => void;
}

export function useApiMutation<TData, TVariables, TFormData extends FieldValues>(
  mutationFn: (variables: TVariables) => Promise<TData>,
  options?: UseApiMutationOptions<TFormData>
) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);

  const mutate = async (variables: TVariables) => {
    setIsLoading(true);
    setError(null);

    try {
      const data = await mutationFn(variables);
      options?.onSuccess?.();
      return data;
    } catch (err) {
      setError(err as Error);

      // Handle backend validation errors
      if (options?.setError && isBackendValidationError(err)) {
        handleBackendValidationErrors(err.response.data.errors, options.setError);
      }

      options?.onError?.(err);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };

  return { mutate, isLoading, error };
}
```

### 9. Environment Variables

Use `.env` files with VITE_ prefix:

```env
# .env
VITE_API_URL=http://localhost:3000
VITE_APP_NAME=MyApp
```

Access in code:
```typescript
const apiUrl = import.meta.env.VITE_API_URL;
```

### 10. File Naming Conventions

- Components: PascalCase (e.g., `UserProfile.tsx`)
- Utilities: camelCase (e.g., `formatDate.ts`)
- Hooks: camelCase with 'use' prefix (e.g., `useAuth.ts`)
- Types: PascalCase (e.g., `User.types.ts`)
- Schemas: camelCase with .schema suffix (e.g., `login.schema.ts`)

## Checklist for New Features

When implementing a new feature:

- [ ] Create feature directory under `features/[feature-name]`
- [ ] Define TypeScript types in `types/` subdirectory
- [ ] Create Zod validation schemas in `schemas/` subdirectory
- [ ] Implement form components using shadcn/ui and React Hook Form
- [ ] Create API client functions with proper typing
- [ ] Handle backend validation errors (422 responses)
- [ ] Implement loading and error states
- [ ] Use Tailwind CSS for styling
- [ ] Create custom hooks for reusable logic
- [ ] Add proper TypeScript types for all props and functions
- [ ] Ensure accessibility (ARIA labels, keyboard navigation)
- [ ] Test form validation (both client-side Zod and server-side errors)

## Example: Complete Feature Implementation

```typescript
// features/users/schemas/create-user.schema.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z.string().email('Invalid email'),
  name: z.string().min(2, 'Name must be at least 2 characters'),
  password: z.string().min(8, 'Password must be at least 8 characters'),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ['confirmPassword'],
});

export type CreateUserFormData = z.infer<typeof createUserSchema>;
```

```typescript
// features/users/api/users.api.ts
import { apiClient } from '@/lib/api/client';
import { CreateUserFormData } from '../schemas/create-user.schema';

export interface User {
  id: string;
  email: string;
  name: string;
}

export async function createUser(data: CreateUserFormData): Promise<User> {
  const response = await apiClient.post<User>('/users', data);
  return response.data;
}
```

```typescript
// features/users/components/CreateUserForm.tsx
import { FC } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useNavigate } from 'react-router-dom';
import { createUserSchema, CreateUserFormData } from '../schemas/create-user.schema';
import { createUser } from '../api/users.api';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
} from '@/components/ui/form';
import { useApiMutation } from '@/hooks/useApiMutation';
import { useToast } from '@/components/ui/use-toast';

export const CreateUserForm: FC = () => {
  const navigate = useNavigate();
  const { toast } = useToast();

  const form = useForm<CreateUserFormData>({
    resolver: zodResolver(createUserSchema),
    defaultValues: {
      email: '',
      name: '',
      password: '',
      confirmPassword: '',
    },
  });

  const { mutate, isLoading } = useApiMutation(createUser, {
    setError: form.setError,
    onSuccess: () => {
      toast({
        title: 'Success',
        description: 'User created successfully',
      });
      navigate('/users');
    },
    onError: (error) => {
      toast({
        title: 'Error',
        description: 'Failed to create user',
        variant: 'destructive',
      });
    },
  });

  const onSubmit = async (data: CreateUserFormData) => {
    await mutate(data);
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-4">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input type="email" placeholder="user@example.com" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="name"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Name</FormLabel>
              <FormControl>
                <Input placeholder="John Doe" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="password"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Password</FormLabel>
              <FormControl>
                <Input type="password" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="confirmPassword"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Confirm Password</FormLabel>
              <FormControl>
                <Input type="password" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={isLoading}>
          {isLoading ? 'Creating...' : 'Create User'}
        </Button>
      </form>
    </Form>
  );
};
```

## Common Patterns

### Loading States
```typescript
{isLoading && <Spinner />}
{!isLoading && data && <DataDisplay data={data} />}
```

### Error Boundaries
Create error boundaries for graceful error handling:
```typescript
import { Component, ErrorInfo, ReactNode } from 'react';

interface Props {
  children: ReactNode;
}

interface State {
  hasError: boolean;
}

export class ErrorBoundary extends Component<Props, State> {
  public state: State = { hasError: false };

  public static getDerivedStateFromError(): State {
    return { hasError: true };
  }

  public componentDidCatch(error: Error, errorInfo: ErrorInfo) {
    console.error('Uncaught error:', error, errorInfo);
  }

  public render() {
    if (this.state.hasError) {
      return <h1>Something went wrong.</h1>;
    }

    return this.props.children;
  }
}
```

## Testing Frontend Components

See `testing-guide.md` for detailed testing instructions including:
- Component testing with React Testing Library
- Form validation testing
- API mocking
- E2E testing with Playwright/Cypress
