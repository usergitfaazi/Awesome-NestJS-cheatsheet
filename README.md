# NEST\_GUIDE.md — NestJS 11.1.6 Setup & How-To Handbook

## Table of Contents

1. [Project Setup (from zero to running)](#1-project-setup-from-zero-to-running)
   1.1 [Install Nest CLI & create app](#11-install-nest-cli--create-app)
   1.2 [Add TypeORM + Postgres](#12-add-typeorm--postgres)
   1.3 [Global validation pipes (class-validator)](#13-global-validation-pipes-class-validator)
   1.4 [Multiple environments (`.env` per env) + validation](#14-multiple-environments-env-per-env--validation)
   1.5 [API Docs: Swagger (OpenAPI)](#15-api-docs-swagger-openapi)
   1.6 [Architecture Docs: Compodoc](#16-architecture-docs-compodoc)
   1.7 [Recommended scripts](#17-recommended-scripts)
2. [How-to by Use Case](#2-howto-by-use-case)
   2.1 [Create a new module quickly](#21-create-a-new-module-quickly)
   2.2 [When a circular dependency appears](#22-when-a-circular-dependency-appears)
   2.3 [Add a new entity (create → import → use)](#23-add-a-new-entity-create--import--use)
   2.4 [Create and run database migrations (TypeORM 0.3+)](#24-create-and-run-database-migrations-typeorm-03)
3. [Controller / Service / Repository patterns](#3-controller--service--repository-patterns)
4. [Guards, Pipes, Interceptors, Filters (quick ref)](#4-guards-pipes-interceptors-filters-quick-ref)
5. [Caching, Scheduling, Events](#5-caching-scheduling-events)
6. [Common Confusions (fast clarifications)](#6-common-confusions-fast-clarifications)
7. [Best Practices (production-minded)](#7-best-practices-production-minded)
8. [Troubleshooting (frequent gotchas)](#8-troubleshooting-frequent-gotchas)
9. [Contributing](#9-contributing)

---

## 1) Project Setup (from zero to running)

### 1.1 Install Nest CLI & create app

```bash
npm i -g @nestjs/cli
nest new my-app
cd my-app
npm run start:dev
```

### 1.2 Add TypeORM + Postgres

```bash
npm i @nestjs/typeorm typeorm pg
```

**`src/app.module.ts`**

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      url: process.env.DATABASE_URL, // e.g. postgres://user:pass@localhost:5432/db
      autoLoadEntities: true,
      synchronize: false,            // NEVER true in prod. Use migrations.
      logging: process.env.NODE_ENV !== 'production',
    }),
  ],
})
export class AppModule {}
```

### 1.3 Global validation pipes (class-validator)

```bash
npm i class-validator class-transformer
```

**`src/main.ts`**

```ts
import { ValidationPipe } from '@nestjs/common';
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(new ValidationPipe({
    whitelist: true,              // strip unknown props
    forbidNonWhitelisted: true,   // throw if unknown props present
    transform: true,              // auto-transform payloads to DTO types
  }));

  await app.listen(process.env.PORT || 3000);
}
bootstrap();
```

### 1.4 Multiple environments (`.env` per env) + validation

```bash
npm i @nestjs/config joi
```

**Project root:**

```
.env
.env.development
.env.staging
.env.production
```

**`src/app.module.ts`**

```ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi';

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,
      envFilePath: [
        `.env.${process.env.NODE_ENV || 'development'}`,
        '.env', // fallback
      ],
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development','test','staging','production').required(),
        DATABASE_URL: Joi.string().uri().required(),
        PORT: Joi.number().default(3000),
      }),
    }),
  ],
})
export class AppModule {}
```

**Usage in services:**

```ts
import { ConfigService } from '@nestjs/config';
constructor(private readonly config: ConfigService) {}
const dbUrl = this.config.get<string>('DATABASE_URL');
```

### 1.5 API Docs: Swagger (OpenAPI)

```bash
npm i @nestjs/swagger swagger-ui-express
```

**`src/main.ts`**

```ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

// ... after creating the app
const config = new DocumentBuilder()
  .setTitle('My API')
  .setDescription('REST API for My App')
  .setVersion('1.0.0')
  .addBearerAuth() // if JWT
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('docs', app, document); // http://localhost:3000/docs
```

**Decorate DTOs & endpoints:**

```ts
import { ApiProperty, ApiTags, ApiOkResponse } from '@nestjs/swagger';

@ApiTags('users')
@Controller('users')
export class UsersController {
  @ApiOkResponse({ description: 'List users' })
  @Get() findAll() {}
}

export class CreateUserDto {
  @ApiProperty({ example: 'alice@example.com' })
  email: string;
}
```

### 1.6 Architecture Docs: Compodoc

```bash
npm i -D @compodoc/compodoc
```

**Serve docs:**

```bash
npx compodoc -p tsconfig.json -s -r 8081
# open http://localhost:8081
```

**Export static site:**

```bash
npx compodoc -p tsconfig.json -d ./compodoc
```

### 1.7 Recommended scripts

**`package.json`**

```json
{
  "scripts": {
    "start": "nest start",
    "start:dev": "nest start --watch",
    "build": "nest build",
    "lint": "eslint .",
    "test": "jest",
    "test:e2e": "jest --config ./test/jest-e2e.json",
    "docs:openapi": "node ./scripts/export-openapi.js",
    "docs:compodoc": "compodoc -p tsconfig.json -s -r 8081"
  }
}
```

---

## 2) How-to by Use Case

### 2.1 Create a new module quickly

```bash
nest g module users
nest g service users --flat --no-spec
nest g controller users --no-spec
```

**Wire it**

```ts
// users.module.ts
import { Module } from '@nestjs/common';
import { UsersController } from './users.controller';
import { UsersService } from './users.service';

@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // export if other modules need it
})
export class UsersModule {}
```

**Consume from another module**

```ts
// orders.module.ts
import { Module } from '@nestjs/common';
import { UsersModule } from '../users/users.module';
import { OrdersService } from './orders.service';

@Module({ imports: [UsersModule], providers: [OrdersService] })
export class OrdersModule {}
```

### 2.2 When a circular dependency appears

**Symptoms:** `Nest can't resolve dependencies of X (Y, ?)... argument Z at index 1 is available...`

**Fix with `forwardRef` on BOTH sides that reference each other:**

```ts
// users.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { OrdersModule } from '../orders/orders.module';
import { UsersService } from './users.service';

@Module({
  imports: [forwardRef(() => OrdersModule)],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// orders.module.ts
import { Module, forwardRef } from '@nestjs/common';
import { UsersModule } from '../users/users.module';
import { OrdersService } from './orders.service';

@Module({
  imports: [forwardRef(() => UsersModule)],
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

**Inject using `forwardRef`:**

```ts
import { Inject, forwardRef } from '@nestjs/common';
constructor(@Inject(forwardRef(() => OrdersService)) private orders: OrdersService) {}
```

> Prefer refactoring to break cycles (extract a shared “core” provider) when possible.

### 2.3 Add a new entity (create → import → use)

**1) Define entity**

```ts
// src/users/user.entity.ts
import { Entity, PrimaryGeneratedColumn, Column, Index } from 'typeorm';

@Entity('users')
export class UserEntity {
  @PrimaryGeneratedColumn('uuid')
  id: string;

  @Index({ unique: true })
  @Column({ nullable: true }) // Postgres allows multiple NULLs with unique index
  email?: string;

  @Column()
  name: string;
}
```

**2) Register in module**

```ts
// src/users/users.module.ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { UsersService } from './users.service';
import { UsersController } from './users.controller';
import { UserEntity } from './user.entity';

@Module({
  imports: [TypeOrmModule.forFeature([UserEntity])],
  providers: [UsersService],
  controllers: [UsersController],
  exports: [UsersService],
})
export class UsersModule {}
```

**3) Use repository in service**

```ts
// src/users/users.service.ts
import { Injectable } from '@nestjs/common';
import { Repository } from 'typeorm';
import { UserEntity } from './user.entity';
import { InjectRepository } from '@nestjs/typeorm';

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(UserEntity) private readonly users: Repository<UserEntity>,
  ) {}

  create(data: Partial<UserEntity>) {
    return this.users.save(this.users.create(data));
  }

  findAll() {
    return this.users.find();
  }
}
```

### 2.4 Create and run database migrations (TypeORM 0.3+)

**1) Dedicated data source for CLI**

```ts
// src/database/data-source.ts
import { DataSource } from 'typeorm';
import { UserEntity } from '../users/user.entity';

export const AppDataSource = new DataSource({
  type: 'postgres',
  url: process.env.DATABASE_URL,
  entities: [UserEntity],
  migrations: [__dirname + '/migrations/*.{ts,js}'],
  synchronize: false,
  logging: false,
});

export default AppDataSource;
```

**2) Scripts (TypeScript execution)**

```bash
npm i -D ts-node
```

**`package.json`**

```json
{
  "scripts": {
    "typeorm": "node --loader ts-node/esm ./node_modules/typeorm/cli.js",
    "db:gen": "npm run typeorm -- migration:generate src/database/migrations/Auto -d src/database/data-source.ts",
    "db:create": "npm run typeorm -- migration:create src/database/migrations/Manual",
    "db:run": "npm run typeorm -- migration:run -d src/database/data-source.ts",
    "db:revert": "npm run typeorm -- migration:revert -d src/database/data-source.ts"
  }
}
```

> If using CommonJS, replace the loader with `ts-node/register` and set `"module": "commonjs"` in `tsconfig.json`.

**3) Generate after entity change**

```bash
# Update or add entities first, then:
npm run db:gen
npm run db:run
```

**4) Example migration (manual)**

```ts
import { MigrationInterface, QueryRunner } from "typeorm";

export class CreateUsers1730000000000 implements MigrationInterface {
  name = 'CreateUsers1730000000000'

  public async up(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`
      CREATE TABLE "users" (
        "id" uuid PRIMARY KEY DEFAULT gen_random_uuid(),
        "email" varchar UNIQUE,
        "name" varchar NOT NULL
      )
    `);
  }

  public async down(queryRunner: QueryRunner): Promise<void> {
    await queryRunner.query(`DROP TABLE "users"`);
  }
}
```

---

## 3) Controller / Service / Repository patterns

**Controller — thin**

```ts
import { Controller, Get, Post, Body } from '@nestjs/common';
import { UsersService } from './users.service';
import { CreateUserDto } from './dtos/create-user.dto';

@Controller('users')
export class UsersController {
  constructor(private readonly users: UsersService) {}

  @Get()
  findAll() { return this.users.findAll(); }

  @Post()
  create(@Body() dto: CreateUserDto) { return this.users.create(dto); }
}
```

**Service — business logic**

```ts
import { Injectable } from '@nestjs/common';
import { Repository } from 'typeorm';
import { InjectRepository } from '@nestjs/typeorm';
import { UserEntity } from './user.entity';

@Injectable()
export class UsersService {
  constructor(@InjectRepository(UserEntity) private repo: Repository<UserEntity>) {}
  // business logic here
}
```

**DTOs + Validation**

```ts
import { IsEmail, IsOptional, IsString } from 'class-validator';

export class CreateUserDto {
  @IsOptional() @IsEmail() email?: string;
  @IsString() name: string;
}
```

---

## 4) Guards, Pipes, Interceptors, Filters (quick ref)

**Guard (auth/perm)**

```ts
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(ctx: ExecutionContext) {
    const req = ctx.switchToHttp().getRequest();
    return !!req.headers.authorization;
  }
}
```

**Pipe (validate/transform)**

```ts
import { Injectable, PipeTransform, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParseIdPipe implements PipeTransform {
  transform(v: string) {
    if (!/^[0-9a-f-]{36}$/i.test(v)) throw new BadRequestException('Invalid UUID');
    return v;
  }
}
```

**Interceptor (wrap/transform/log)**

```ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { tap } from 'rxjs/operators';
import { Observable } from 'rxjs';

@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler): Observable<any> {
    const started = Date.now();
    return next.handle().pipe(tap(() => console.log('took', Date.now() - started, 'ms')));
  }
}
```

**Exception Filter (centralize errors)**

```ts
import { Catch, ExceptionFilter, ArgumentsHost, HttpException } from '@nestjs/common';

@Catch(HttpException)
export class HttpErrorFilter implements ExceptionFilter {
  catch(e: HttpException, host: ArgumentsHost) {
    const res = host.switchToHttp().getResponse();
    res.status(e.getStatus()).json({ error: e.message });
  }
}
```

---

## 5) Caching, Scheduling, Events

**Cache**

```ts
import { Module } from '@nestjs/common';
import { CacheModule, CacheInterceptor } from '@nestjs/cache-manager';

@Module({
  imports: [CacheModule.register({ ttl: 60 })],
})
export class AppModule {}
```

```ts
import { UseInterceptors } from '@nestjs/common';
@UseInterceptors(CacheInterceptor)
@Get() getHeavy() { /* ... */ }
```

**Scheduling**

```bash
npm i @nestjs/schedule
```

```ts
import { Injectable } from '@nestjs/common';
import { Cron, CronExpression } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  @Cron(CronExpression.EVERY_30_SECONDS)
  handleCron() { /* ... */ }
}
```

**Events**

```bash
npm i @nestjs/event-emitter
```

```ts
import { EventEmitter2 } from '@nestjs/event-emitter';
constructor(private emitter: EventEmitter2) {}
this.emitter.emit('user.created', { id: '...' });
```

---

## 6) Common Confusions (fast clarifications)

* **Service vs Repository**
  Service = business logic/orchestration.
  Repository = persistence (CRUD) injected *into* the service.

* **DTO vs Entity**
  DTO = request/response contract + validation.
  Entity = DB mapping. Don’t return entities directly to clients.

* **Providers vs Exports**
  `providers` are local to the module.
  To reuse elsewhere, add to `exports` and import that module where needed.

* **Middleware vs Interceptor**
  Middleware runs before route handler at framework level (raw req/res).
  Interceptor wraps handler execution (transform/measure/cache).

* **Circular deps**
  Use `forwardRef` as a stopgap. Prefer extracting shared logic into a separate module to break the cycle.

* **`unique` + `nullable` (emails)**
  Postgres allows multiple `NULL` rows under a unique index. “Unique when present” is fine with `nullable: true` + `unique`.

---

## 7) Best Practices (production-minded)

* Disable `synchronize` in prod; use migrations.
* Validate env with `@nestjs/config` + `Joi`.
* Use DTOs + global `ValidationPipe` with `whitelist` and `transform`.
* Keep controllers slim; push logic into services.
* Organize by **domain** (users/, orders/, payments/) not by type.
* Put shared utilities in a `common`/`shared` module (avoid circular deps).
* Add Swagger early; export OpenAPI in CI for clients.
* Use Compodoc for onboarding/architecture visibility.
* Log structured JSON (Pino/Winston) in prod; add request-ID correlation.
* Cover unit + e2e tests; centralize exception filtering.

---

## 8) Troubleshooting (frequent gotchas)

* **“Nest can’t resolve dependencies…”**
  Missing `exports: [X]` or forgot to import the module that provides X, or a circular dep—use `forwardRef` or refactor.

* **Migrations don’t see entities**
  Ensure the CLI **data source** includes correct `entities` and `migrations` paths; pass `-d src/database/data-source.ts`.

* **Validation not running**
  `app.useGlobalPipes(new ValidationPipe(...))` in `main.ts`, and DTOs must be used in controller method params.

* **Swagger not showing models**
  Decorate DTO props with `@ApiProperty()` and tag controllers with `@ApiTags()`.

* **Config not loading**
  Check `envFilePath` ordering and `NODE_ENV`; ensure `validationSchema` matches required vars.

  ## 9) Contributing
Spotted a gap or want to improve examples? Read the [Contributing Guide](CONTRIBUTING.md) and open a PR.

**Quick start:** fork → branch → edit → `pnpm test` (or `npm run test`) → PR with a clear description.

