# Repository Pattern Guide

## Overview
This guide provides detailed instructions for implementing the Repository Pattern with swappable database implementations in NestJS applications.

## Core Principles

### Why Repository Pattern?

1. **Database Abstraction**: Services don't know about the underlying database
2. **Swappable Implementation**: Easy to switch from Prisma to TypeORM, MongoDB, etc.
3. **Testability**: Services can be tested with mock repositories
4. **Single Responsibility**: Data access logic is separated from business logic
5. **Domain-Driven Design**: Work with domain entities, not ORM models

### Architecture Rules

1. **Services NEVER import Prisma/TypeORM/etc. directly**
2. **Services ONLY depend on repository interfaces**
3. **Repositories implement interfaces and handle DB operations**
4. **Domain entities are database-agnostic**
5. **Each feature has its own repository**

## Implementation Steps

### Step 1: Define Domain Entity

Create database-agnostic domain entities:

```typescript
// domain/entities/post.entity.ts
export class Post {
  id: string;
  title: string;
  content: string;
  published: boolean;
  authorId: string;
  createdAt: Date;
  updatedAt: Date;
}

export class CreatePostData {
  title: string;
  content: string;
  published?: boolean;
  authorId: string;
}

export class UpdatePostData {
  title?: string;
  content?: string;
  published?: boolean;
}

export class PostFilters {
  authorId?: string;
  published?: boolean;
  search?: string;
}
```

### Step 2: Define Repository Interface

Create an interface that defines all data operations:

```typescript
// domain/repositories/post-repository.interface.ts
import { Post, CreatePostData, UpdatePostData, PostFilters } from '../entities/post.entity';

export interface IPostRepository {
  // Read operations
  findById(id: string): Promise<Post | null>;
  findAll(filters?: PostFilters, skip?: number, take?: number): Promise<Post[]>;
  findByAuthor(authorId: string): Promise<Post[]>;

  // Write operations
  create(data: CreatePostData): Promise<Post>;
  update(id: string, data: UpdatePostData): Promise<Post>;
  delete(id: string): Promise<void>;

  // Utility operations
  count(filters?: PostFilters): Promise<number>;
  exists(id: string): Promise<boolean>;
}

// Dependency Injection token
export const POST_REPOSITORY = 'PostRepository';
```

### Step 3: Implement Prisma Repository

Implement the interface using Prisma:

```typescript
// infrastructure/database/repositories/prisma-post-repository.ts
import { Injectable } from '@nestjs/common';
import { PrismaService } from '../prisma/prisma.service';
import { IPostRepository } from '@/domain/repositories/post-repository.interface';
import {
  Post,
  CreatePostData,
  UpdatePostData,
  PostFilters,
} from '@/domain/entities/post.entity';

@Injectable()
export class PrismaPostRepository implements IPostRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<Post | null> {
    const post = await this.prisma.post.findUnique({
      where: { id },
    });
    return post ? this.toDomain(post) : null;
  }

  async findAll(
    filters?: PostFilters,
    skip: number = 0,
    take: number = 10,
  ): Promise<Post[]> {
    const where = this.buildWhereClause(filters);
    const posts = await this.prisma.post.findMany({
      where,
      skip,
      take,
      orderBy: { createdAt: 'desc' },
    });
    return posts.map(this.toDomain);
  }

  async findByAuthor(authorId: string): Promise<Post[]> {
    const posts = await this.prisma.post.findMany({
      where: { authorId },
      orderBy: { createdAt: 'desc' },
    });
    return posts.map(this.toDomain);
  }

  async create(data: CreatePostData): Promise<Post> {
    const post = await this.prisma.post.create({
      data: {
        title: data.title,
        content: data.content,
        published: data.published ?? false,
        authorId: data.authorId,
      },
    });
    return this.toDomain(post);
  }

  async update(id: string, data: UpdatePostData): Promise<Post> {
    const post = await this.prisma.post.update({
      where: { id },
      data,
    });
    return this.toDomain(post);
  }

  async delete(id: string): Promise<void> {
    await this.prisma.post.delete({
      where: { id },
    });
  }

  async count(filters?: PostFilters): Promise<number> {
    const where = this.buildWhereClause(filters);
    return this.prisma.post.count({ where });
  }

  async exists(id: string): Promise<boolean> {
    const count = await this.prisma.post.count({
      where: { id },
    });
    return count > 0;
  }

  // Private helper methods
  private toDomain(prismaPost: any): Post {
    return {
      id: prismaPost.id,
      title: prismaPost.title,
      content: prismaPost.content,
      published: prismaPost.published,
      authorId: prismaPost.authorId,
      createdAt: prismaPost.createdAt,
      updatedAt: prismaPost.updatedAt,
    };
  }

  private buildWhereClause(filters?: PostFilters): any {
    if (!filters) return {};

    const where: any = {};

    if (filters.authorId) {
      where.authorId = filters.authorId;
    }

    if (filters.published !== undefined) {
      where.published = filters.published;
    }

    if (filters.search) {
      where.OR = [
        { title: { contains: filters.search, mode: 'insensitive' } },
        { content: { contains: filters.search, mode: 'insensitive' } },
      ];
    }

    return where;
  }
}
```

### Step 4: Create Service Using Repository

Services depend on the repository interface, not the implementation:

```typescript
// features/posts/services/posts.service.ts
import { Injectable, Inject, NotFoundException } from '@nestjs/common';
import {
  IPostRepository,
  POST_REPOSITORY,
} from '@/domain/repositories/post-repository.interface';
import {
  Post,
  CreatePostData,
  UpdatePostData,
  PostFilters,
} from '@/domain/entities/post.entity';

@Injectable()
export class PostsService {
  constructor(
    @Inject(POST_REPOSITORY)
    private readonly postRepository: IPostRepository,
  ) {}

  async findById(id: string): Promise<Post> {
    const post = await this.postRepository.findById(id);
    if (!post) {
      throw new NotFoundException(`Post with ID ${id} not found`);
    }
    return post;
  }

  async findAll(
    filters?: PostFilters,
    page: number = 1,
    limit: number = 10,
  ): Promise<{ posts: Post[]; total: number }> {
    const skip = (page - 1) * limit;
    const [posts, total] = await Promise.all([
      this.postRepository.findAll(filters, skip, limit),
      this.postRepository.count(filters),
    ]);
    return { posts, total };
  }

  async findByAuthor(authorId: string): Promise<Post[]> {
    return this.postRepository.findByAuthor(authorId);
  }

  async create(data: CreatePostData): Promise<Post> {
    // Business logic here (validation, etc.)
    return this.postRepository.create(data);
  }

  async update(id: string, data: UpdatePostData): Promise<Post> {
    // Verify post exists
    await this.findById(id);

    // Business logic here
    return this.postRepository.update(id, data);
  }

  async delete(id: string): Promise<void> {
    // Verify post exists
    await this.findById(id);

    await this.postRepository.delete(id);
  }

  async publish(id: string): Promise<Post> {
    // Business logic: publish a post
    return this.update(id, { published: true });
  }

  async unpublish(id: string): Promise<Post> {
    // Business logic: unpublish a post
    return this.update(id, { published: false });
  }
}
```

### Step 5: Register in Module

Use dependency injection to bind interface to implementation:

```typescript
// features/posts/posts.module.ts
import { Module } from '@nestjs/common';
import { PostsController } from './controllers/posts.controller';
import { PostsService } from './services/posts.service';
import { PrismaPostRepository } from '@/infrastructure/database/repositories/prisma-post-repository';
import { PrismaService } from '@/infrastructure/database/prisma/prisma.service';
import { POST_REPOSITORY } from '@/domain/repositories/post-repository.interface';

@Module({
  controllers: [PostsController],
  providers: [
    PostsService,
    PrismaService,
    {
      provide: POST_REPOSITORY,
      useClass: PrismaPostRepository,
    },
  ],
  exports: [PostsService],
})
export class PostsModule {}
```

## Swapping Database Implementation

### Example: TypeORM Implementation

To switch to TypeORM, create a new repository implementation:

```typescript
// infrastructure/database/repositories/typeorm-post-repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { IPostRepository } from '@/domain/repositories/post-repository.interface';
import {
  Post,
  CreatePostData,
  UpdatePostData,
  PostFilters,
} from '@/domain/entities/post.entity';
import { PostEntity } from '../typeorm/entities/post.entity'; // TypeORM entity

@Injectable()
export class TypeOrmPostRepository implements IPostRepository {
  constructor(
    @InjectRepository(PostEntity)
    private repository: Repository<PostEntity>,
  ) {}

  async findById(id: string): Promise<Post | null> {
    const post = await this.repository.findOne({ where: { id } });
    return post ? this.toDomain(post) : null;
  }

  async findAll(
    filters?: PostFilters,
    skip: number = 0,
    take: number = 10,
  ): Promise<Post[]> {
    const query = this.repository.createQueryBuilder('post');

    if (filters?.authorId) {
      query.andWhere('post.authorId = :authorId', { authorId: filters.authorId });
    }

    if (filters?.published !== undefined) {
      query.andWhere('post.published = :published', { published: filters.published });
    }

    if (filters?.search) {
      query.andWhere(
        '(post.title ILIKE :search OR post.content ILIKE :search)',
        { search: `%${filters.search}%` },
      );
    }

    const posts = await query.skip(skip).take(take).orderBy('post.createdAt', 'DESC').getMany();
    return posts.map(this.toDomain);
  }

  async findByAuthor(authorId: string): Promise<Post[]> {
    const posts = await this.repository.find({
      where: { authorId },
      order: { createdAt: 'DESC' },
    });
    return posts.map(this.toDomain);
  }

  async create(data: CreatePostData): Promise<Post> {
    const post = this.repository.create({
      title: data.title,
      content: data.content,
      published: data.published ?? false,
      authorId: data.authorId,
    });
    const saved = await this.repository.save(post);
    return this.toDomain(saved);
  }

  async update(id: string, data: UpdatePostData): Promise<Post> {
    await this.repository.update(id, data);
    const post = await this.repository.findOne({ where: { id } });
    return this.toDomain(post!);
  }

  async delete(id: string): Promise<void> {
    await this.repository.delete(id);
  }

  async count(filters?: PostFilters): Promise<number> {
    const query = this.repository.createQueryBuilder('post');

    if (filters?.authorId) {
      query.andWhere('post.authorId = :authorId', { authorId: filters.authorId });
    }

    if (filters?.published !== undefined) {
      query.andWhere('post.published = :published', { published: filters.published });
    }

    if (filters?.search) {
      query.andWhere(
        '(post.title ILIKE :search OR post.content ILIKE :search)',
        { search: `%${filters.search}%` },
      );
    }

    return query.getCount();
  }

  async exists(id: string): Promise<boolean> {
    const count = await this.repository.count({ where: { id } });
    return count > 0;
  }

  private toDomain(entity: PostEntity): Post {
    return {
      id: entity.id,
      title: entity.title,
      content: entity.content,
      published: entity.published,
      authorId: entity.authorId,
      createdAt: entity.createdAt,
      updatedAt: entity.updatedAt,
    };
  }
}
```

### Switching Implementation

To switch databases, just change the module configuration:

```typescript
// posts.module.ts
import { TypeOrmPostRepository } from '@/infrastructure/database/repositories/typeorm-post-repository';

@Module({
  providers: [
    PostsService,
    {
      provide: POST_REPOSITORY,
      useClass: TypeOrmPostRepository, // Changed from PrismaPostRepository
    },
  ],
})
export class PostsModule {}
```

**The service code remains unchanged!** This is the power of the repository pattern.

## Advanced Patterns

### 1. Transactions

Add transaction support to repository interface:

```typescript
// domain/repositories/base-repository.interface.ts
export interface ITransactionManager {
  runInTransaction<T>(work: () => Promise<T>): Promise<T>;
}

// Prisma implementation
async runInTransaction<T>(work: () => Promise<T>): Promise<T> {
  return this.prisma.$transaction(async () => {
    return work();
  });
}
```

### 2. Specification Pattern

For complex queries:

```typescript
// domain/specifications/post-specification.ts
export interface Specification<T> {
  isSatisfiedBy(entity: T): boolean;
}

export class PublishedPostSpecification implements Specification<Post> {
  isSatisfiedBy(post: Post): boolean {
    return post.published === true;
  }
}

export class AuthorPostSpecification implements Specification<Post> {
  constructor(private authorId: string) {}

  isSatisfiedBy(post: Post): boolean {
    return post.authorId === this.authorId;
  }
}
```

### 3. Base Repository

Create a base repository for common operations:

```typescript
// domain/repositories/base-repository.interface.ts
export interface IBaseRepository<T, TCreate, TUpdate, TFilters = any> {
  findById(id: string): Promise<T | null>;
  findAll(filters?: TFilters, skip?: number, take?: number): Promise<T[]>;
  create(data: TCreate): Promise<T>;
  update(id: string, data: TUpdate): Promise<T>;
  delete(id: string): Promise<void>;
  count(filters?: TFilters): Promise<number>;
  exists(id: string): Promise<boolean>;
}

// Extend for specific repositories
export interface IPostRepository extends IBaseRepository<
  Post,
  CreatePostData,
  UpdatePostData,
  PostFilters
> {
  findByAuthor(authorId: string): Promise<Post[]>;
  // Additional post-specific methods
}
```

### 4. Repository with Relations

Handle relationships in repositories:

```typescript
// domain/entities/post.entity.ts
export class PostWithAuthor extends Post {
  author: {
    id: string;
    name: string;
    email: string;
  };
}

// domain/repositories/post-repository.interface.ts
export interface IPostRepository {
  // ... other methods
  findByIdWithAuthor(id: string): Promise<PostWithAuthor | null>;
  findAllWithAuthors(filters?: PostFilters): Promise<PostWithAuthor[]>;
}

// Prisma implementation
async findByIdWithAuthor(id: string): Promise<PostWithAuthor | null> {
  const post = await this.prisma.post.findUnique({
    where: { id },
    include: {
      author: {
        select: {
          id: true,
          name: true,
          email: true,
        },
      },
    },
  });

  if (!post) return null;

  return {
    ...this.toDomain(post),
    author: post.author,
  };
}
```

### 5. Caching Layer

Add caching to repositories:

```typescript
// infrastructure/database/repositories/cached-post-repository.ts
import { Injectable, Inject } from '@nestjs/common';
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Cache } from 'cache-manager';
import { IPostRepository } from '@/domain/repositories/post-repository.interface';
import { Post } from '@/domain/entities/post.entity';

@Injectable()
export class CachedPostRepository implements IPostRepository {
  constructor(
    @Inject('BASE_POST_REPOSITORY')
    private baseRepository: IPostRepository,
    @Inject(CACHE_MANAGER)
    private cacheManager: Cache,
  ) {}

  async findById(id: string): Promise<Post | null> {
    const cacheKey = `post:${id}`;
    const cached = await this.cacheManager.get<Post>(cacheKey);

    if (cached) {
      return cached;
    }

    const post = await this.baseRepository.findById(id);

    if (post) {
      await this.cacheManager.set(cacheKey, post, 300); // 5 minutes
    }

    return post;
  }

  async create(data: CreatePostData): Promise<Post> {
    const post = await this.baseRepository.create(data);
    await this.cacheManager.set(`post:${post.id}`, post, 300);
    return post;
  }

  async update(id: string, data: UpdatePostData): Promise<Post> {
    const post = await this.baseRepository.update(id, data);
    await this.cacheManager.del(`post:${id}`); // Invalidate cache
    return post;
  }

  // ... implement other methods
}
```

## Testing with Repository Pattern

The repository pattern makes testing easy:

```typescript
// features/posts/services/posts.service.spec.ts
import { Test } from '@nestjs/testing';
import { PostsService } from './posts.service';
import { IPostRepository, POST_REPOSITORY } from '@/domain/repositories/post-repository.interface';
import { NotFoundException } from '@nestjs/common';

describe('PostsService', () => {
  let service: PostsService;
  let repository: jest.Mocked<IPostRepository>;

  beforeEach(async () => {
    // Create mock repository
    const mockRepository: jest.Mocked<IPostRepository> = {
      findById: jest.fn(),
      findAll: jest.fn(),
      findByAuthor: jest.fn(),
      create: jest.fn(),
      update: jest.fn(),
      delete: jest.fn(),
      count: jest.fn(),
      exists: jest.fn(),
    };

    const module = await Test.createTestingModule({
      providers: [
        PostsService,
        {
          provide: POST_REPOSITORY,
          useValue: mockRepository,
        },
      ],
    }).compile();

    service = module.get<PostsService>(PostsService);
    repository = module.get(POST_REPOSITORY);
  });

  it('should find post by id', async () => {
    const mockPost = {
      id: '1',
      title: 'Test Post',
      content: 'Content',
      published: true,
      authorId: 'user1',
      createdAt: new Date(),
      updatedAt: new Date(),
    };

    repository.findById.mockResolvedValue(mockPost);

    const result = await service.findById('1');

    expect(result).toEqual(mockPost);
    expect(repository.findById).toHaveBeenCalledWith('1');
  });

  it('should throw NotFoundException when post not found', async () => {
    repository.findById.mockResolvedValue(null);

    await expect(service.findById('999')).rejects.toThrow(NotFoundException);
  });

  // More tests...
});
```

## Checklist for Implementing Repository Pattern

- [ ] Define domain entity (database-agnostic)
- [ ] Create repository interface in `domain/repositories/`
- [ ] Define DI token constant (e.g., `POST_REPOSITORY`)
- [ ] Implement repository in `infrastructure/database/repositories/prisma-[feature]-repository.ts`
- [ ] Implement `toDomain()` mapper method
- [ ] Implement all interface methods
- [ ] Handle filters and search in `buildWhereClause()` method
- [ ] Inject repository interface (not implementation) in service
- [ ] Register repository in module using DI token
- [ ] Write unit tests using mock repository
- [ ] Document repository methods

## Common Mistakes to Avoid

1. **DON'T**: Import Prisma in services
   ```typescript
   // ❌ WRONG
   constructor(private prisma: PrismaService) {}
   ```

2. **DO**: Inject repository interface
   ```typescript
   // ✅ CORRECT
   constructor(@Inject(POST_REPOSITORY) private repo: IPostRepository) {}
   ```

3. **DON'T**: Return Prisma models from repository
   ```typescript
   // ❌ WRONG
   async findById(id: string): Promise<PrismaPost> {
     return this.prisma.post.findUnique({ where: { id } });
   }
   ```

4. **DO**: Map to domain entities
   ```typescript
   // ✅ CORRECT
   async findById(id: string): Promise<Post | null> {
     const post = await this.prisma.post.findUnique({ where: { id } });
     return post ? this.toDomain(post) : null;
   }
   ```

5. **DON'T**: Put business logic in repositories
   ```typescript
   // ❌ WRONG - business logic in repository
   async createPost(data: CreatePostData): Promise<Post> {
     if (data.content.length < 100) {
       throw new Error('Content too short');
     }
     // ...
   }
   ```

6. **DO**: Keep repositories focused on data access
   ```typescript
   // ✅ CORRECT - pure data access
   async create(data: CreatePostData): Promise<Post> {
     const post = await this.prisma.post.create({ data });
     return this.toDomain(post);
   }
   ```

## Benefits Summary

1. **Testability**: Easy to mock repositories for unit tests
2. **Flexibility**: Switch databases without changing business logic
3. **Maintainability**: Clear separation of concerns
4. **Domain-Driven**: Work with domain concepts, not database details
5. **Refactoring**: Safe to refactor data layer without affecting services

See related guides:
- `backend-development-guide.md` for overall backend architecture
- `testing-guide.md` for testing repository implementations
