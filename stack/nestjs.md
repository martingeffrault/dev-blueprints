# NestJS (2025)

> **Last updated**: January 2026
> **Versions covered**: NestJS 10+, 11.x
> **Purpose**: Enterprise-grade Node.js backend framework with TypeScript

---

## Philosophy (2025-2026)

NestJS is the **enterprise Node.js framework** — Angular-inspired architecture with dependency injection, decorators, and modules. It enforces structure that scales.

**Key principles:**
- **Modular architecture** — Feature-based modules
- **Dependency injection** — IoC container for testability
- **Decorator-based** — Clean, declarative code
- **TypeScript-first** — Full type safety
- **Layered design** — Controllers → Services → Repositories
- **SOLID principles** — Single responsibility, dependency inversion

---

## TL;DR

- Structure by feature modules, not by type
- Use DTOs with class-validator for all inputs
- Services for business logic, not controllers
- Use Guards for auth, Interceptors for transforms
- Custom exceptions with filters
- Repository pattern for data access
- Config module for environment variables
- Always use strict TypeScript

---

## Best Practices

### Project Structure (Feature-Based)

```
src/
├── app.module.ts
├── main.ts
├── common/                     # Shared code
│   ├── decorators/
│   │   └── current-user.decorator.ts
│   ├── filters/
│   │   └── http-exception.filter.ts
│   ├── guards/
│   │   └── auth.guard.ts
│   ├── interceptors/
│   │   ├── logging.interceptor.ts
│   │   └── transform.interceptor.ts
│   ├── pipes/
│   │   └── validation.pipe.ts
│   └── dto/
│       └── pagination.dto.ts
├── config/                     # Configuration
│   ├── config.module.ts
│   ├── database.config.ts
│   └── app.config.ts
├── modules/                    # Feature modules
│   ├── auth/
│   │   ├── auth.module.ts
│   │   ├── auth.controller.ts
│   │   ├── auth.service.ts
│   │   ├── strategies/
│   │   │   └── jwt.strategy.ts
│   │   ├── guards/
│   │   │   └── jwt-auth.guard.ts
│   │   └── dto/
│   │       ├── login.dto.ts
│   │       └── register.dto.ts
│   ├── users/
│   │   ├── users.module.ts
│   │   ├── users.controller.ts
│   │   ├── users.service.ts
│   │   ├── users.repository.ts
│   │   ├── entities/
│   │   │   └── user.entity.ts
│   │   └── dto/
│   │       ├── create-user.dto.ts
│   │       └── update-user.dto.ts
│   └── posts/
│       ├── posts.module.ts
│       ├── posts.controller.ts
│       ├── posts.service.ts
│       ├── posts.repository.ts
│       ├── entities/
│       │   └── post.entity.ts
│       └── dto/
│           ├── create-post.dto.ts
│           └── update-post.dto.ts
├── database/
│   ├── database.module.ts
│   └── migrations/
└── health/
    ├── health.module.ts
    └── health.controller.ts
```

### Main Entry Point

```typescript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { ValidationPipe, VersioningType } from '@nestjs/common';
import { ConfigService } from '@nestjs/config';
import { DocumentBuilder, SwaggerModule } from '@nestjs/swagger';
import helmet from 'helmet';
import { AppModule } from './app.module';
import { HttpExceptionFilter } from './common/filters/http-exception.filter';
import { TransformInterceptor } from './common/interceptors/transform.interceptor';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: ['error', 'warn', 'log'],
  });

  const configService = app.get(ConfigService);

  // Security
  app.use(helmet());
  app.enableCors({
    origin: configService.get('CORS_ORIGINS')?.split(',') || [],
    credentials: true,
  });

  // API versioning
  app.enableVersioning({
    type: VersioningType.URI,
    defaultVersion: '1',
  });

  // Global prefix
  app.setGlobalPrefix('api');

  // Global pipes, filters, interceptors
  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: {
        enableImplicitConversion: true,
      },
    }),
  );
  app.useGlobalFilters(new HttpExceptionFilter());
  app.useGlobalInterceptors(new TransformInterceptor());

  // Swagger documentation
  if (configService.get('NODE_ENV') !== 'production') {
    const config = new DocumentBuilder()
      .setTitle('My API')
      .setDescription('API Documentation')
      .setVersion('1.0')
      .addBearerAuth()
      .build();
    const document = SwaggerModule.createDocument(app, config);
    SwaggerModule.setup('docs', app, document);
  }

  // Graceful shutdown
  app.enableShutdownHooks();

  const port = configService.get('PORT') || 3000;
  await app.listen(port);
  console.log(`Application is running on: ${await app.getUrl()}`);
}
bootstrap();
```

### App Module

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';
import { DatabaseModule } from './database/database.module';
import { AuthModule } from './modules/auth/auth.module';
import { UsersModule } from './modules/users/users.module';
import { PostsModule } from './modules/posts/posts.module';
import { HealthModule } from './health/health.module';
import { appConfig, databaseConfig } from './config';

@Module({
  imports: [
    // Configuration
    ConfigModule.forRoot({
      isGlobal: true,
      load: [appConfig, databaseConfig],
      envFilePath: ['.env.local', '.env'],
    }),

    // Rate limiting
    ThrottlerModule.forRoot([
      {
        ttl: 60000,
        limit: 100,
      },
    ]),

    // Database
    DatabaseModule,

    // Feature modules
    AuthModule,
    UsersModule,
    PostsModule,
    HealthModule,
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard,
    },
  ],
})
export class AppModule {}
```

### Configuration

```typescript
// src/config/app.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('app', () => ({
  nodeEnv: process.env.NODE_ENV || 'development',
  port: parseInt(process.env.PORT, 10) || 3000,
  jwtSecret: process.env.JWT_SECRET,
  jwtExpiresIn: process.env.JWT_EXPIRES_IN || '1d',
  corsOrigins: process.env.CORS_ORIGINS?.split(',') || ['http://localhost:3000'],
}));

// src/config/database.config.ts
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DB_HOST || 'localhost',
  port: parseInt(process.env.DB_PORT, 10) || 5432,
  username: process.env.DB_USERNAME,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_DATABASE,
}));
```

### Feature Module

```typescript
// src/modules/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';
import { User } from './entities/user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService, UsersRepository],
  exports: [UsersService],
})
export class UsersModule {}
```

### Entity

```typescript
// src/modules/users/entities/user.entity.ts
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  OneToMany,
  Index,
} from 'typeorm';
import { Exclude } from 'class-transformer';
import { Post } from '../../posts/entities/post.entity';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Column({ length: 100 })
  name: string;

  @Index({ unique: true })
  @Column({ length: 255 })
  email: string;

  @Exclude()
  @Column()
  password: string;

  @Column({ default: true })
  isActive: boolean;

  @Column({ type: 'enum', enum: ['user', 'admin'], default: 'user' })
  role: 'user' | 'admin';

  @OneToMany(() => Post, (post) => post.author)
  posts: Post[];

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;
}
```

### DTOs with Validation

```typescript
// src/modules/users/dto/create-user.dto.ts
import {
  IsEmail,
  IsString,
  MinLength,
  MaxLength,
  IsOptional,
  IsEnum,
} from 'class-validator';
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'John Doe' })
  @IsString()
  @MinLength(2)
  @MaxLength(100)
  name: string;

  @ApiProperty({ example: 'john@example.com' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'password123', minLength: 8 })
  @IsString()
  @MinLength(8)
  password: string;

  @ApiPropertyOptional({ enum: ['user', 'admin'], default: 'user' })
  @IsOptional()
  @IsEnum(['user', 'admin'])
  role?: 'user' | 'admin';
}

// src/modules/users/dto/update-user.dto.ts
import { PartialType, OmitType } from '@nestjs/swagger';
import { CreateUserDto } from './create-user.dto';

export class UpdateUserDto extends PartialType(
  OmitType(CreateUserDto, ['password'] as const),
) {}

// src/common/dto/pagination.dto.ts
import { IsOptional, IsInt, Min, Max } from 'class-validator';
import { Type } from 'class-transformer';
import { ApiPropertyOptional } from '@nestjs/swagger';

export class PaginationDto {
  @ApiPropertyOptional({ default: 1 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  page?: number = 1;

  @ApiPropertyOptional({ default: 20, maximum: 100 })
  @IsOptional()
  @Type(() => Number)
  @IsInt()
  @Min(1)
  @Max(100)
  limit?: number = 20;
}
```

### Controller

```typescript
// src/modules/users/users.controller.ts
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  ParseUUIDPipe,
  HttpCode,
  HttpStatus,
  UseGuards,
} from '@nestjs/common';
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
} from '@nestjs/swagger';
import { UsersService } from './users.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { PaginationDto } from '../../common/dto/pagination.dto';
import { JwtAuthGuard } from '../auth/guards/jwt-auth.guard';
import { RolesGuard } from '../auth/guards/roles.guard';
import { Roles } from '../auth/decorators/roles.decorator';
import { CurrentUser } from '../../common/decorators/current-user.decorator';

@ApiTags('users')
@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Post()
  @ApiOperation({ summary: 'Create a new user' })
  @ApiResponse({ status: 201, description: 'User created successfully' })
  @ApiResponse({ status: 409, description: 'Email already exists' })
  async create(@Body() createUserDto: CreateUserDto) {
    return this.usersService.create(createUserDto);
  }

  @Get()
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @ApiOperation({ summary: 'Get all users' })
  async findAll(@Query() pagination: PaginationDto) {
    return this.usersService.findAll(pagination);
  }

  @Get(':id')
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @ApiOperation({ summary: 'Get user by ID' })
  @ApiResponse({ status: 404, description: 'User not found' })
  async findOne(@Param('id', ParseUUIDPipe) id: string) {
    return this.usersService.findOne(id);
  }

  @Put(':id')
  @UseGuards(JwtAuthGuard)
  @ApiBearerAuth()
  @ApiOperation({ summary: 'Update user' })
  async update(
    @Param('id', ParseUUIDPipe) id: string,
    @Body() updateUserDto: UpdateUserDto,
  ) {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(':id')
  @UseGuards(JwtAuthGuard, RolesGuard)
  @Roles('admin')
  @ApiBearerAuth()
  @HttpCode(HttpStatus.NO_CONTENT)
  @ApiOperation({ summary: 'Delete user (admin only)' })
  async remove(@Param('id', ParseUUIDPipe) id: string) {
    await this.usersService.remove(id);
  }
}
```

### Service

```typescript
// src/modules/users/users.service.ts
import {
  Injectable,
  NotFoundException,
  ConflictException,
} from '@nestjs/common';
import * as bcrypt from 'bcrypt';
import { UsersRepository } from './users.repository';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';
import { PaginationDto } from '../../common/dto/pagination.dto';

@Injectable()
export class UsersService {
  constructor(private readonly usersRepository: UsersRepository) {}

  async create(createUserDto: CreateUserDto) {
    const existingUser = await this.usersRepository.findByEmail(
      createUserDto.email,
    );
    if (existingUser) {
      throw new ConflictException('Email already exists');
    }

    const hashedPassword = await bcrypt.hash(createUserDto.password, 10);

    const user = await this.usersRepository.create({
      ...createUserDto,
      password: hashedPassword,
    });

    const { password, ...result } = user;
    return result;
  }

  async findAll(pagination: PaginationDto) {
    const { page, limit } = pagination;
    const skip = (page - 1) * limit;

    const [users, total] = await this.usersRepository.findAll(skip, limit);

    return {
      data: users,
      meta: {
        page,
        limit,
        total,
        totalPages: Math.ceil(total / limit),
      },
    };
  }

  async findOne(id: string) {
    const user = await this.usersRepository.findById(id);
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }

  async findByEmail(email: string) {
    return this.usersRepository.findByEmail(email);
  }

  async update(id: string, updateUserDto: UpdateUserDto) {
    const user = await this.findOne(id);
    Object.assign(user, updateUserDto);
    return this.usersRepository.save(user);
  }

  async remove(id: string) {
    const user = await this.findOne(id);
    await this.usersRepository.remove(user);
  }
}
```

### Repository

```typescript
// src/modules/users/users.repository.ts
import { Injectable } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './entities/user.entity';

@Injectable()
export class UsersRepository {
  constructor(
    @InjectRepository(User)
    private readonly repository: Repository<User>,
  ) {}

  async create(data: Partial<User>): Promise<User> {
    const user = this.repository.create(data);
    return this.repository.save(user);
  }

  async findAll(skip: number, take: number): Promise<[User[], number]> {
    return this.repository.findAndCount({
      skip,
      take,
      order: { createdAt: 'DESC' },
      select: ['id', 'name', 'email', 'role', 'createdAt'],
    });
  }

  async findById(id: string): Promise<User | null> {
    return this.repository.findOne({
      where: { id },
      select: ['id', 'name', 'email', 'role', 'createdAt'],
    });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.repository.findOne({ where: { email } });
  }

  async save(user: User): Promise<User> {
    return this.repository.save(user);
  }

  async remove(user: User): Promise<void> {
    await this.repository.remove(user);
  }
}
```

### Exception Filter

```typescript
// src/common/filters/http-exception.filter.ts
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
  Logger,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger(HttpExceptionFilter.name);

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    let status = HttpStatus.INTERNAL_SERVER_ERROR;
    let message = 'Internal server error';
    let code = 'INTERNAL_ERROR';

    if (exception instanceof HttpException) {
      status = exception.getStatus();
      const exceptionResponse = exception.getResponse();

      if (typeof exceptionResponse === 'object') {
        message = (exceptionResponse as any).message || exception.message;
        code = (exceptionResponse as any).error || 'ERROR';
      } else {
        message = exceptionResponse as string;
      }
    }

    // Log internal errors
    if (status >= 500) {
      this.logger.error(
        `${request.method} ${request.url}`,
        exception instanceof Error ? exception.stack : exception,
      );
    }

    response.status(status).json({
      success: false,
      error: {
        code,
        message,
        timestamp: new Date().toISOString(),
        path: request.url,
      },
    });
  }
}
```

### Transform Interceptor

```typescript
// src/common/interceptors/transform.interceptor.ts
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface Response<T> {
  success: boolean;
  data: T;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, Response<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<Response<T>> {
    return next.handle().pipe(
      map((data) => ({
        success: true,
        data,
      })),
    );
  }
}
```

### Custom Decorator

```typescript
// src/common/decorators/current-user.decorator.ts
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    return data ? user?.[data] : user;
  },
);

// Usage in controller
@Get('me')
getProfile(@CurrentUser() user: User) {
  return user;
}

@Get('my-id')
getMyId(@CurrentUser('id') userId: string) {
  return { userId };
}
```

### Testing

```typescript
// src/modules/users/users.service.spec.ts
import { Test, TestingModule } from '@nestjs/testing';
import { ConflictException, NotFoundException } from '@nestjs/common';
import { UsersService } from './users.service';
import { UsersRepository } from './users.repository';

describe('UsersService', () => {
  let service: UsersService;
  let repository: jest.Mocked<UsersRepository>;

  beforeEach(async () => {
    const mockRepository = {
      create: jest.fn(),
      findAll: jest.fn(),
      findById: jest.fn(),
      findByEmail: jest.fn(),
      save: jest.fn(),
      remove: jest.fn(),
    };

    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UsersService,
        { provide: UsersRepository, useValue: mockRepository },
      ],
    }).compile();

    service = module.get<UsersService>(UsersService);
    repository = module.get(UsersRepository);
  });

  describe('create', () => {
    it('should create a user', async () => {
      const createDto = {
        name: 'John Doe',
        email: 'john@example.com',
        password: 'password123',
      };

      repository.findByEmail.mockResolvedValue(null);
      repository.create.mockResolvedValue({
        id: '1',
        ...createDto,
        password: 'hashed',
        role: 'user',
        createdAt: new Date(),
        updatedAt: new Date(),
      } as any);

      const result = await service.create(createDto);

      expect(result).not.toHaveProperty('password');
      expect(result.email).toBe(createDto.email);
    });

    it('should throw ConflictException if email exists', async () => {
      repository.findByEmail.mockResolvedValue({ id: '1' } as any);

      await expect(
        service.create({
          name: 'John',
          email: 'existing@example.com',
          password: 'password123',
        }),
      ).rejects.toThrow(ConflictException);
    });
  });

  describe('findOne', () => {
    it('should throw NotFoundException if user not found', async () => {
      repository.findById.mockResolvedValue(null);

      await expect(service.findOne('non-existent')).rejects.toThrow(
        NotFoundException,
      );
    });
  });
});
```

---

## Anti-Patterns

### ❌ Business Logic in Controllers

```typescript
// ❌ DON'T — Logic in controller
@Post()
async create(@Body() dto: CreateUserDto) {
  const hashedPassword = await bcrypt.hash(dto.password, 10);
  return this.userRepository.save({ ...dto, password: hashedPassword });
}

// ✅ DO — Logic in service
@Post()
async create(@Body() dto: CreateUserDto) {
  return this.usersService.create(dto);
}
```

### ❌ No Input Validation

```typescript
// ❌ DON'T — No validation
@Post()
async create(@Body() body: any) {
  return this.service.create(body);
}

// ✅ DO — DTO with validation
@Post()
async create(@Body() dto: CreateUserDto) {
  return this.service.create(dto);
}
```

### ❌ Exposing Sensitive Data

```typescript
// ❌ DON'T — Return password
return user;

// ✅ DO — Exclude sensitive fields
const { password, ...result } = user;
return result;

// Or use class-transformer
@Exclude()
@Column()
password: string;
```

### ❌ Not Using Dependency Injection

```typescript
// ❌ DON'T — Direct instantiation
export class UsersService {
  private repository = new UsersRepository();
}

// ✅ DO — Inject dependencies
@Injectable()
export class UsersService {
  constructor(private readonly repository: UsersRepository) {}
}
```

---

## 2025-2026 Changelog

| Version | Date | Key Changes |
|---------|------|-------------|
| **11.0** | Jan 2025 | Express v5 default, JSON logging, faster startup |
| **11.1** | Mar 2025 | Reflector changes, Apollo Server v4 support |
| 10.x | 2024 | ESM support, standalone applications |

### NestJS 11 Breaking Changes

**Express v5 Default — Wildcard Route Naming:**
```typescript
// ❌ OLD — Express v4 unnamed wildcards
@Get('*')  // Worked in NestJS 10
async catchAll() {}

@Get('files/*')  // Worked in NestJS 10
async serveFile() {}

// ✅ NEW — Express v5 requires named wildcards
@Get('*splat')  // Named wildcard parameter
async catchAll(@Param('splat') path: string) {}

@Get('files/*filepath')  // Named wildcard
async serveFile(@Param('filepath') filepath: string) {}

// Or use explicit regex for flexibility
@Get(/^\/files\/(.*)$/)
async serveFileRegex(@Param('0') filepath: string) {}

// Note: This is NOT a NestJS change — it's Express v5
// You can still use Express v4 by explicitly installing it
```

**Reflector.getAllAndMerge Returns Object (Breaking):**
```typescript
// ❌ OLD — Returned array in NestJS 10
const roles = this.reflector.getAllAndMerge('roles', [
  context.getHandler(),
  context.getClass(),
]);
// roles was: ['admin', 'user'] (array)

// ✅ NEW — Returns merged object in NestJS 11
const metadata = this.reflector.getAllAndMerge('config', [
  context.getHandler(),
  context.getClass(),
]);
// metadata is now: { roles: ['admin', 'user'], cache: true } (object)

// If you need array behavior, use getAllAndOverride instead
const roles = this.reflector.getAllAndOverride<string[]>('roles', [
  context.getHandler(),
  context.getClass(),
]);
```

**Faster Startup — Object References for Opaque Keys:**
```typescript
// INTERNAL CHANGE — Faster module initialization
// NestJS 11 uses object references instead of string-based lookups
// No code changes needed, just faster cold starts

// Benefits:
// - 15-20% faster application bootstrap
// - Better memory efficiency during startup
// - Improved module resolution performance
```

### NestJS 11 New Features

**Enhanced ConsoleLogger with JSON Output:**
```typescript
// NEW — JSON logging for production (structured logging)
import { NestFactory } from '@nestjs/core';
import { ConsoleLogger } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule, {
    logger: new ConsoleLogger({
      // NEW in NestJS 11
      json: process.env.NODE_ENV === 'production',
      colors: process.env.NODE_ENV !== 'production',
      timestamp: true,
    }),
  });
  await app.listen(3000);
}

// Production output (JSON):
// {"level":"log","message":"Application started","timestamp":"2025-01-21T10:00:00.000Z","context":"NestApplication"}

// Development output (colored):
// [Nest] 12345  - 01/21/2025, 10:00:00 AM     LOG [NestApplication] Application started
```

**Apollo Server v4 Integration:**
```typescript
// UPDATED — @nestjs/graphql now uses Apollo Server v4
// package.json
{
  "@nestjs/graphql": "^13.0.0",
  "@apollo/server": "^4.0.0",  // Replaces apollo-server-express
  "graphql": "^16.0.0"
}

// ❌ OLD — Apollo Server v3 setup (NestJS 10)
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';

GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: 'schema.gql',
  playground: true,  // Deprecated option
});

// ✅ NEW — Apollo Server v4 setup (NestJS 11)
import { ApolloDriver, ApolloDriverConfig } from '@nestjs/apollo';
import { ApolloServerPluginLandingPageLocalDefault } from '@apollo/server/plugin/landingPage/default';

GraphQLModule.forRoot<ApolloDriverConfig>({
  driver: ApolloDriver,
  autoSchemaFile: 'schema.gql',
  // Playground replaced with Apollo Sandbox
  plugins: [ApolloServerPluginLandingPageLocalDefault()],
  // Or disable in production
  // plugins: process.env.NODE_ENV === 'production' ? [] : [ApolloServerPluginLandingPageLocalDefault()],
});
```

**Improved TypeORM Integration:**
```typescript
// Better support for TypeORM 0.3.x in NestJS 11
import { TypeOrmModule } from '@nestjs/typeorm';
import { DataSource } from 'typeorm';

@Module({
  imports: [
    TypeOrmModule.forRootAsync({
      useFactory: () => ({
        type: 'postgres',
        host: process.env.DB_HOST,
        // ... other options
      }),
      // NEW — Access to DataSource for migrations
      dataSourceFactory: async (options) => {
        const dataSource = new DataSource(options);
        return dataSource.initialize();
      },
    }),
  ],
})
export class AppModule {}
```

**WebSocket Gateway Enhancements:**
```typescript
// Improved WebSocket gateway with better typing
import {
  WebSocketGateway,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  WsResponse,
} from '@nestjs/websockets';
import { Socket } from 'socket.io';

@WebSocketGateway({
  cors: {
    origin: '*',
  },
  // NEW — Better namespace handling
  namespace: /\/ws-.+/,  // Regex namespace support
})
export class EventsGateway {
  @SubscribeMessage('events')
  handleEvent(
    @MessageBody() data: string,
    @ConnectedSocket() client: Socket,
  ): WsResponse<string> {
    return { event: 'events', data: `Received: ${data}` };
  }
}
```

### Migration Checklist v10 → v11

```typescript
// 1. Update wildcard routes
// Before: @Get('*')
// After:  @Get('*splat')

// 2. Check Reflector usage
// getAllAndMerge now returns object, not array
// Use getAllAndOverride if you need array

// 3. Update Apollo Server (if using GraphQL)
// Replace apollo-server-express with @apollo/server
// Update plugins configuration

// 4. Review custom decorators using Reflector
// Test metadata retrieval logic

// 5. Update logging configuration for production
// Consider switching to JSON logging
```

---

## Quick Reference

| Decorator | Purpose |
|-----------|---------|
| `@Module()` | Define module |
| `@Controller()` | Define controller |
| `@Injectable()` | Mark as provider |
| `@Get/@Post/@Put/@Delete` | HTTP methods |
| `@Body/@Param/@Query` | Request data |
| `@UseGuards()` | Apply guards |
| `@UseInterceptors()` | Apply interceptors |

| Command | Purpose |
|---------|---------|
| `nest new project-name` | Create project |
| `nest g module users` | Generate module |
| `nest g controller users` | Generate controller |
| `nest g service users` | Generate service |
| `npm run start:dev` | Dev server |
| `npm run test` | Run tests |

---

## Resources

- [NestJS Documentation](https://docs.nestjs.com/)
- [NestJS Best Practices](https://docs.nestjs.com/fundamentals/testing)
- [Clean Architecture with NestJS](https://github.com/wesleey/nest-clean-architecture)
- [NestJS Architecture Patterns](https://learn.nestjs.com/p/architecture-and-advanced-patterns)
