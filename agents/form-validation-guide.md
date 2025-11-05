# Form Validation Guide

## Overview
This guide defines the standards for form validation across the full stack, ensuring consistent error handling between backend (NestJS) and frontend (React + Zod).

## Core Requirement

**Backend must return validation errors in a structure that frontend can easily map to individual form fields, similar to how Zod displays errors.**

### Standard Error Response Format

All validation errors (422 status) must follow this structure:

```json
{
  "message": "Validation failed",
  "errors": {
    "email": ["Email is already in use", "Email format is invalid"],
    "password": ["Password must be at least 8 characters"],
    "name": ["Name is required"]
  }
}
```

**Key Requirements:**
- HTTP Status: `422 Unprocessable Entity`
- `message`: General error message (string)
- `errors`: Object where keys are field names, values are arrays of error messages (string[])
- Each field under `errors` must be explicitly defined in the DTO
- Frontend can directly map these errors to form fields

## Backend Validation (NestJS)

### Step 1: Create DTOs with Validation

Use class-validator decorators:

```typescript
// features/users/dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  Matches,
  IsOptional,
} from 'class-validator';
import { ApiProperty } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'user@example.com' })
  @IsEmail({}, { message: 'Invalid email format' })
  email: string;

  @ApiProperty({ example: 'John Doe' })
  @IsString({ message: 'Name must be a string' })
  @MinLength(2, { message: 'Name must be at least 2 characters' })
  @MaxLength(100, { message: 'Name cannot exceed 100 characters' })
  name: string;

  @ApiProperty({ example: 'SecurePass123!' })
  @IsString({ message: 'Password must be a string' })
  @MinLength(8, { message: 'Password must be at least 8 characters' })
  @MaxLength(100, { message: 'Password cannot exceed 100 characters' })
  @Matches(/^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/, {
    message: 'Password must contain uppercase, lowercase, and number',
  })
  password: string;

  @ApiProperty({ example: 'Software Engineer', required: false })
  @IsOptional()
  @IsString({ message: 'Title must be a string' })
  @MaxLength(200, { message: 'Title cannot exceed 200 characters' })
  title?: string;
}
```

### Step 2: Create Custom Validation Exception Filter

Create a filter to format validation errors correctly:

```typescript
// common/filters/validation-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  BadRequestException,
  HttpStatus,
} from '@nestjs/common';
import { Response } from 'express';
import { ValidationError } from 'class-validator';

export interface ValidationErrorResponse {
  message: string;
  errors: Record<string, string[]>;
}

@Catch(BadRequestException)
export class ValidationExceptionFilter implements ExceptionFilter {
  catch(exception: BadRequestException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const exceptionResponse = exception.getResponse();

    // Check if this is a validation error
    if (
      typeof exceptionResponse === 'object' &&
      'message' in exceptionResponse &&
      Array.isArray((exceptionResponse as any).message)
    ) {
      const validationErrors = (exceptionResponse as any).message;
      const formattedErrors = this.formatValidationErrors(validationErrors);

      const errorResponse: ValidationErrorResponse = {
        message: 'Validation failed',
        errors: formattedErrors,
      };

      return response.status(HttpStatus.UNPROCESSABLE_ENTITY).json(errorResponse);
    }

    // Not a validation error, return original response
    response.status(exception.getStatus()).json(exceptionResponse);
  }

  private formatValidationErrors(errors: any[]): Record<string, string[]> {
    const formattedErrors: Record<string, string[]> = {};

    errors.forEach((error) => {
      if (typeof error === 'string') {
        // Simple string error
        formattedErrors['_general'] = formattedErrors['_general'] || [];
        formattedErrors['_general'].push(error);
      } else if (error.property && error.constraints) {
        // class-validator error format
        const field = error.property;
        const messages = Object.values(error.constraints) as string[];
        formattedErrors[field] = messages;
      }
    });

    return formattedErrors;
  }
}
```

### Step 3: Register Filter Globally

```typescript
// main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe } from '@nestjs/common';
import { AppModule } from './app.module';
import { ValidationExceptionFilter } from './common/filters/validation-exception.filter';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Enable validation globally
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true, // Strip properties not in DTO
      forbidNonWhitelisted: true, // Throw error for non-whitelisted properties
      transform: true, // Transform payloads to DTO instances
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );

  // Apply validation exception filter
  app.useGlobalFilters(new ValidationExceptionFilter());

  await app.listen(3000);
}
bootstrap();
```

### Step 4: Add Business Logic Validation

For custom validation (e.g., unique email check):

```typescript
// features/users/services/users.service.ts
import { Injectable, Inject, ConflictException } from '@nestjs/common';
import { IUserRepository, USER_REPOSITORY } from '@/domain/repositories/user-repository.interface';

@Injectable()
export class UsersService {
  constructor(
    @Inject(USER_REPOSITORY)
    private readonly userRepository: IUserRepository,
  ) {}

  async create(data: CreateUserDto): Promise<User> {
    // Check if email is already in use
    const existingUser = await this.userRepository.findByEmail(data.email);
    if (existingUser) {
      // Throw structured validation error
      throw new ValidationException({
        email: ['Email is already in use'],
      });
    }

    // Hash password
    const hashedPassword = await bcrypt.hash(data.password, 10);

    return this.userRepository.create({
      ...data,
      password: hashedPassword,
    });
  }
}
```

### Step 5: Create Custom ValidationException

```typescript
// common/exceptions/validation.exception.ts
import { HttpException, HttpStatus } from '@nestjs/common';

export class ValidationException extends HttpException {
  constructor(errors: Record<string, string[]>) {
    super(
      {
        message: 'Validation failed',
        errors,
      },
      HttpStatus.UNPROCESSABLE_ENTITY,
    );
  }
}
```

### Example: Complete Validation Flow

```typescript
// Controller
@Post()
@HttpCode(HttpStatus.CREATED)
@ApiResponse({ status: 201, description: 'User created' })
@ApiResponse({
  status: 422,
  description: 'Validation error',
  schema: {
    example: {
      message: 'Validation failed',
      errors: {
        email: ['Email is already in use'],
        password: ['Password must be at least 8 characters']
      }
    }
  }
})
async create(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
  const user = await this.usersService.create(createUserDto);
  return plainToInstance(UserResponseDto, user);
}
```

## Frontend Validation (React + Zod)

### Step 1: Define Zod Schema

Mirror backend validations:

```typescript
// features/users/schemas/create-user.schema.ts
import { z } from 'zod';

export const createUserSchema = z.object({
  email: z
    .string()
    .min(1, 'Email is required')
    .email('Invalid email format'),

  name: z
    .string()
    .min(2, 'Name must be at least 2 characters')
    .max(100, 'Name cannot exceed 100 characters'),

  password: z
    .string()
    .min(8, 'Password must be at least 8 characters')
    .max(100, 'Password cannot exceed 100 characters')
    .regex(
      /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)/,
      'Password must contain uppercase, lowercase, and number'
    ),

  title: z
    .string()
    .max(200, 'Title cannot exceed 200 characters')
    .optional(),
});

export type CreateUserFormData = z.infer<typeof createUserSchema>;
```

### Step 2: Create Backend Error Handler

```typescript
// lib/utils/form-errors.ts
import { UseFormSetError, FieldValues, Path } from 'react-hook-form';

export interface BackendValidationErrors {
  [field: string]: string[];
}

export interface BackendErrorResponse {
  message: string;
  errors: BackendValidationErrors;
}

/**
 * Maps backend validation errors (422) to React Hook Form errors
 *
 * Backend format:
 * {
 *   "message": "Validation failed",
 *   "errors": {
 *     "email": ["Email is already in use"],
 *     "password": ["Password is too weak"]
 *   }
 * }
 */
export function handleBackendValidationErrors<T extends FieldValues>(
  errors: BackendValidationErrors,
  setError: UseFormSetError<T>,
): void {
  Object.entries(errors).forEach(([field, messages]) => {
    if (messages && messages.length > 0) {
      setError(field as Path<T>, {
        type: 'server',
        message: messages[0], // Show first error message
      });

      // Optionally, you can show all messages
      // message: messages.join(', ')
    }
  });
}

/**
 * Type guard to check if error is a backend validation error
 */
export function isBackendValidationError(
  error: any
): error is { response: { status: number; data: BackendErrorResponse } } {
  return (
    error?.response?.status === 422 &&
    error?.response?.data?.errors &&
    typeof error.response.data.errors === 'object'
  );
}
```

### Step 3: Create API Client with Error Handling

```typescript
// lib/api/client.ts
import axios, { AxiosError } from 'axios';

export const apiClient = axios.create({
  baseURL: import.meta.env.VITE_API_URL || 'http://localhost:3000',
  headers: {
    'Content-Type': 'application/json',
  },
});

// Add request interceptor for auth
apiClient.interceptors.request.use((config) => {
  const token = localStorage.getItem('auth_token');
  if (token) {
    config.headers.Authorization = `Bearer ${token}`;
  }
  return config;
});

// Add response interceptor for error handling
apiClient.interceptors.response.use(
  (response) => response,
  (error: AxiosError) => {
    // Log errors in development
    if (import.meta.env.DEV) {
      console.error('API Error:', error.response?.data);
    }
    return Promise.reject(error);
  }
);
```

### Step 4: Create Form with Error Handling

```typescript
// features/users/components/CreateUserForm.tsx
import { FC } from 'react';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createUserSchema, CreateUserFormData } from '../schemas/create-user.schema';
import { createUser } from '../api/users.api';
import {
  handleBackendValidationErrors,
  isBackendValidationError,
} from '@/lib/utils/form-errors';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import {
  Form,
  FormControl,
  FormField,
  FormItem,
  FormLabel,
  FormMessage,
  FormDescription,
} from '@/components/ui/form';
import { useToast } from '@/components/ui/use-toast';
import { useNavigate } from 'react-router-dom';

export const CreateUserForm: FC = () => {
  const navigate = useNavigate();
  const { toast } = useToast();

  const form = useForm<CreateUserFormData>({
    resolver: zodResolver(createUserSchema),
    defaultValues: {
      email: '',
      name: '',
      password: '',
      title: '',
    },
  });

  const onSubmit = async (data: CreateUserFormData) => {
    try {
      await createUser(data);

      toast({
        title: 'Success',
        description: 'User created successfully',
      });

      navigate('/users');
    } catch (error: any) {
      // Handle backend validation errors (422)
      if (isBackendValidationError(error)) {
        handleBackendValidationErrors(
          error.response.data.errors,
          form.setError
        );

        toast({
          title: 'Validation Error',
          description: error.response.data.message,
          variant: 'destructive',
        });
      } else {
        // Handle other errors
        toast({
          title: 'Error',
          description: 'An unexpected error occurred',
          variant: 'destructive',
        });
      }
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <FormField
          control={form.control}
          name="email"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Email</FormLabel>
              <FormControl>
                <Input
                  type="email"
                  placeholder="user@example.com"
                  {...field}
                />
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
              <FormDescription>
                Must contain uppercase, lowercase, and number
              </FormDescription>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="title"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Title (Optional)</FormLabel>
              <FormControl>
                <Input placeholder="Software Engineer" {...field} />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button
          type="submit"
          disabled={form.formState.isSubmitting}
          className="w-full"
        >
          {form.formState.isSubmitting ? 'Creating...' : 'Create User'}
        </Button>
      </form>
    </Form>
  );
};
```

### Step 5: Create Reusable Hook

```typescript
// hooks/useApiMutation.ts
import { useState } from 'react';
import { UseFormSetError, FieldValues } from 'react-hook-form';
import {
  handleBackendValidationErrors,
  isBackendValidationError,
} from '@/lib/utils/form-errors';

interface UseApiMutationOptions<T extends FieldValues> {
  setError?: UseFormSetError<T>;
  onSuccess?: () => void;
  onError?: (error: any) => void;
}

export function useApiMutation<TData, TVariables, TFormData extends FieldValues = any>(
  mutationFn: (variables: TVariables) => Promise<TData>,
  options?: UseApiMutationOptions<TFormData>
) {
  const [isLoading, setIsLoading] = useState(false);
  const [error, setError] = useState<Error | null>(null);
  const [data, setData] = useState<TData | null>(null);

  const mutate = async (variables: TVariables) => {
    setIsLoading(true);
    setError(null);

    try {
      const result = await mutationFn(variables);
      setData(result);
      options?.onSuccess?.();
      return result;
    } catch (err: any) {
      setError(err);

      // Automatically handle backend validation errors
      if (options?.setError && isBackendValidationError(err)) {
        handleBackendValidationErrors(
          err.response.data.errors,
          options.setError
        );
      }

      options?.onError?.(err);
      throw err;
    } finally {
      setIsLoading(false);
    }
  };

  const reset = () => {
    setData(null);
    setError(null);
  };

  return {
    mutate,
    isLoading,
    error,
    data,
    reset,
  };
}
```

### Simplified Form with Hook

```typescript
// Using the custom hook
export const CreateUserFormSimplified: FC = () => {
  const navigate = useNavigate();
  const { toast } = useToast();

  const form = useForm<CreateUserFormData>({
    resolver: zodResolver(createUserSchema),
  });

  const { mutate, isLoading } = useApiMutation(createUser, {
    setError: form.setError, // Automatically handle validation errors
    onSuccess: () => {
      toast({ title: 'Success', description: 'User created' });
      navigate('/users');
    },
    onError: (error) => {
      if (isBackendValidationError(error)) {
        toast({
          title: 'Validation Error',
          description: error.response.data.message,
          variant: 'destructive',
        });
      } else {
        toast({
          title: 'Error',
          description: 'An unexpected error occurred',
          variant: 'destructive',
        });
      }
    },
  });

  const onSubmit = async (data: CreateUserFormData) => {
    await mutate(data);
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)}>
        {/* Form fields */}
        <Button type="submit" disabled={isLoading}>
          {isLoading ? 'Creating...' : 'Create User'}
        </Button>
      </form>
    </Form>
  );
};
```

## Advanced Validation Patterns

### 1. Nested Object Validation

Backend:
```typescript
import { ValidateNested, Type } from 'class-validator';

class AddressDto {
  @IsString()
  street: string;

  @IsString()
  city: string;

  @IsString()
  @Length(5, 5)
  zipCode: string;
}

class CreateUserDto {
  @IsEmail()
  email: string;

  @ValidateNested()
  @Type(() => AddressDto)
  address: AddressDto;
}
```

Frontend:
```typescript
const createUserSchema = z.object({
  email: z.string().email(),
  address: z.object({
    street: z.string(),
    city: z.string(),
    zipCode: z.string().length(5, 'Zip code must be 5 characters'),
  }),
});
```

### 2. Array Validation

Backend:
```typescript
import { IsArray, ArrayMinSize, ValidateNested } from 'class-validator';

class CreatePostDto {
  @IsString()
  title: string;

  @IsArray()
  @ArrayMinSize(1, { message: 'At least one tag is required' })
  @IsString({ each: true })
  tags: string[];
}
```

Frontend:
```typescript
const createPostSchema = z.object({
  title: z.string(),
  tags: z
    .array(z.string())
    .min(1, 'At least one tag is required'),
});
```

### 3. Custom Validation

Backend:
```typescript
import { registerDecorator, ValidationOptions } from 'class-validator';

export function IsStrongPassword(validationOptions?: ValidationOptions) {
  return function (object: Object, propertyName: string) {
    registerDecorator({
      name: 'isStrongPassword',
      target: object.constructor,
      propertyName: propertyName,
      options: validationOptions,
      validator: {
        validate(value: any) {
          return (
            typeof value === 'string' &&
            /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/.test(value)
          );
        },
        defaultMessage: () =>
          'Password must contain uppercase, lowercase, number, and special character',
      },
    });
  };
}

class CreateUserDto {
  @IsStrongPassword()
  password: string;
}
```

Frontend:
```typescript
const passwordValidation = z
  .string()
  .regex(
    /^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[@$!%*?&])/,
    'Password must contain uppercase, lowercase, number, and special character'
  );

const createUserSchema = z.object({
  password: passwordValidation,
});
```

### 4. Conditional Validation

Backend:
```typescript
import { ValidateIf } from 'class-validator';

class UpdateUserDto {
  @IsOptional()
  @IsEmail()
  email?: string;

  @ValidateIf(o => o.email !== undefined)
  @IsString()
  @MinLength(1)
  emailConfirmation?: string;
}
```

Frontend:
```typescript
const updateUserSchema = z.object({
  email: z.string().email().optional(),
  emailConfirmation: z.string().optional(),
}).refine(
  (data) => {
    if (data.email && !data.emailConfirmation) {
      return false;
    }
    return true;
  },
  {
    message: 'Email confirmation is required when updating email',
    path: ['emailConfirmation'],
  }
);
```

## Testing Validation

### Backend Tests

```typescript
// features/users/controllers/users.controller.spec.ts
describe('UsersController - Validation', () => {
  it('should return 422 for invalid email', async () => {
    const response = await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'invalid-email',
        name: 'John Doe',
        password: 'Password123',
      })
      .expect(422);

    expect(response.body).toEqual({
      message: 'Validation failed',
      errors: {
        email: expect.arrayContaining([
          expect.stringContaining('Invalid email'),
        ]),
      },
    });
  });

  it('should return 422 for duplicate email', async () => {
    // Create first user
    await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'test@example.com',
        name: 'John Doe',
        password: 'Password123',
      });

    // Try to create duplicate
    const response = await request(app.getHttpServer())
      .post('/users')
      .send({
        email: 'test@example.com',
        name: 'Jane Doe',
        password: 'Password456',
      })
      .expect(422);

    expect(response.body).toEqual({
      message: 'Validation failed',
      errors: {
        email: ['Email is already in use'],
      },
    });
  });
});
```

### Frontend Tests

```typescript
// features/users/components/CreateUserForm.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { CreateUserForm } from './CreateUserForm';
import { createUser } from '../api/users.api';

jest.mock('../api/users.api');

describe('CreateUserForm', () => {
  it('should display client-side validation errors', async () => {
    render(<CreateUserForm />);

    const submitButton = screen.getByRole('button', { name: /create/i });
    await userEvent.click(submitButton);

    expect(await screen.findByText(/email is required/i)).toBeInTheDocument();
    expect(screen.getByText(/name must be at least 2 characters/i)).toBeInTheDocument();
  });

  it('should display backend validation errors', async () => {
    (createUser as jest.Mock).mockRejectedValue({
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

    render(<CreateUserForm />);

    await userEvent.type(screen.getByLabelText(/email/i), 'test@example.com');
    await userEvent.type(screen.getByLabelText(/name/i), 'John Doe');
    await userEvent.type(screen.getByLabelText(/password/i), 'Password123');
    await userEvent.click(screen.getByRole('button', { name: /create/i }));

    await waitFor(() => {
      expect(screen.getByText(/email is already in use/i)).toBeInTheDocument();
    });
  });
});
```

## Checklist

### Backend
- [ ] Create DTO with class-validator decorators
- [ ] Each field has appropriate validation
- [ ] Custom ValidationExceptionFilter is registered
- [ ] Global ValidationPipe is enabled
- [ ] Business logic throws ValidationException for custom errors
- [ ] API documentation includes 422 response examples
- [ ] Validation tests are written

### Frontend
- [ ] Zod schema mirrors backend validations
- [ ] Backend error handler utility is created
- [ ] Form uses React Hook Form with Zod resolver
- [ ] Backend validation errors are mapped to form fields
- [ ] Error messages are displayed under each field
- [ ] Toast/notification shows general error message
- [ ] Loading states are handled
- [ ] Form validation tests are written

## Best Practices

1. **Keep validation in sync** between frontend and backend
2. **Client-side validation** for UX (immediate feedback)
3. **Server-side validation** for security (never trust client)
4. **Consistent error messages** across frontend and backend
5. **User-friendly messages** (not technical jargon)
6. **Test both client and server validation**
7. **Document validation rules** in API docs

See related guides:
- `frontend-development-guide.md` for form implementation
- `backend-development-guide.md` for DTO creation
- `testing-guide.md` for validation testing
