Here’s a drop-in **`NEST_GUIDE.md`**—dense, practical, and ToC-friendly. It starts with setup (install → DB → validation → environments → docs) and then “How-to by use-case” so you can jump straight to what you want.

---

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

`src/app.module.ts`

```ts
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      url: process.env.DATABASE_URL, // e.g. postgres://user:pass@localhost:5432/db
      autoLoadEntities: true,
      synchronize: false, // NEVER true in prod. Use migrations.
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

`src/main.ts`

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

Project root:

```
.env            # default / dev
.env.development
.env.staging
.env.production
```

`src/app.module.ts`

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

Usage in services:

```ts
import { ConfigService } from '@nestjs/config';

constructor(private readonly config: ConfigService) {}
const dbUrl = this.config.get<string>('DATABASE_URL');
```

### 1.5 API Docs: Swagger (OpenAPI)

```bash
npm i @nestjs/swagger swagger-ui-express
```

`src/main.ts`

```ts
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

const config = new DocumentBuilder()
  .setTitle('My API')
  .setDescription('REST API for My App')
  .setVersion('1.0.0')
  .addBearerAuth() // if JWT
  .build();

const document = SwaggerModule.createDocument(app, config);
SwaggerModule.setup('docs', app, document); // http://localhost:3000/docs
```

Decorate DTOs & endpoints for better docs:

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

Generate & serve:

```bash
npx compodoc -p tsconfig.json -s -r 8081
# open http://localhost:8081
```

Generate static docs:

```bash
npx compodoc -p tsconfig.json -d ./compodoc
```

### 1.7 Recommended scripts

`package.json`

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

Wire it:

```ts
// users.module.ts
@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService], // export if other modules need it
})
export class UsersModule {}
```

Use from another module:

```ts
@Module({ imports: [UsersModule] })
export class OrdersModule {
  constructor(private readonly users: UsersService) {}
}
```

### 2.2 When a circular dependency appears

Symptoms: `Nest can't resolve dependencies of X (Y, ?). Please make sure that the argument Z at index 1 is available...`

Fix with `forwardRef` on BOTH sides that reference each other:

```ts
// users.module.ts
@Module({
  imports: [forwardRef(() => OrdersModule)],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

// orders.module.ts
@Module({
  imports: [forwardRef(() => UsersModule)],
  providers: [OrdersService],
  exports: [OrdersService],
})
export class OrdersModule {}
```

And inject the opposite using `forwardRef`:

```ts
constructor(@Inject(forwardRef(() => OrdersService)) private orders: OrdersService) {}
```

**Prefer refactoring** to break cycles (extract a shared “core” provider) if possible.

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
  @Column({ nullable: true }) // unique + nullable is fine in Postgres; multiple NULLs allowed
  email?: string;

  @Column()
  name: string;
}
```

**2) Register in module**

```ts
// src/users/users.module.ts
import { TypeOrmModule } from '@nestjs/typeorm';
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

**1) Create a dedicated data source for CLI**

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

**2) Scripts (TS exec)**

```bash
npm i -D ts-node
```

`package.json`

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
# Update/新增 entities first, then:
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
@Injectable()
export class UsersService {
  constructor(@InjectRepository(UserEntity) private repo: Repository<UserEntity>) {}
  // do validation/coordination, not raw HTTP concerns
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
@Injectable()
export class LoggingInterceptor implements NestInterceptor {
  intercept(ctx: ExecutionContext, next: CallHandler) {
    const started = Date.now();
    return next.handle().pipe(tap(() => console.log('took', Date.now() - started, 'ms')));
  }
}
```

**Exception Filter (centralize errors)**

```ts
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
import { CacheModule } from '@nestjs/cache-manager';

@Module({
  imports: [CacheModule.register({ ttl: 60 })],
})
export class AppModule {}
```

```ts
@UseInterceptors(CacheInterceptor)
@Get() getHeavy() { /* ... */ }
```

**Scheduling**

```bash
npm i @nestjs/schedule
```

```ts
import { Cron, CronExpression, SchedulerRegistry } from '@nestjs/schedule';

@Cron(CronExpression.EVERY_30_SECONDS)
handleCron() { /* ... */ }
```

**Events**

```bash
npm i @nestjs/event-emitter
```

```ts
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
  To reuse elsewhere, add to `exports` and import the module there.

* **Middleware vs Interceptor**
  Middleware runs before the route handler at the framework level (raw req/res).
  Interceptor wraps handler execution (can map/transform responses, measure timing, cache).

* **Circular deps**
  Use `forwardRef` as a stopgap. Prefer extracting shared logic into a separate module to break the cycle cleanly.

* **`unique` + `nullable` (emails)**
  Postgres allows multiple `NULL` with a unique index. If you need “unique when present,” it’s fine to use `nullable: true` + `unique`.

---

## 7) Best Practices (production-minded)

* Disable `synchronize` in prod; use migrations.
* Validate env with `@nestjs/config` + `Joi`.
* Use DTOs + global `ValidationPipe` with `whitelist` and `transform`.
* Keep controllers slim; push logic into services.
* Organize by **domain** (users/, orders/, payments/) not by type.
* Put shared utilities into a `shared`/`common` module (no circular deps).
* Add Swagger early; CI can export OpenAPI JSON for clients.
* Compodoc for team onboarding/architecture visibility.
* Log structured JSON in prod (Pino/Winston).
* Add request-ID correlation (e.g., header middleware) if you care about tracing.

---

## 8) Troubleshooting (frequent gotchas)

* **“Nest can’t resolve dependencies…”**
  You forgot to `exports: [X]` or to import the module that provides X.
  Or you have a circular dep—use `forwardRef` or refactor.

* **Migrations don’t see my entities**
  Make sure the CLI **data source** includes the correct `entities` paths and that the `-d` option points to it.

* **Validation not running**
  Ensure `app.useGlobalPipes(new ValidationPipe(...))` is in `main.ts` and DTOs are used as method params.

* **Swagger not showing models**
  Decorate DTO props with `@ApiProperty()` and ensure the controller has `@ApiTags()`.

* **Config not loading**
  Confirm `envFilePath` ordering, `NODE_ENV`, and validation schema in `ConfigModule.forRoot`.

---

### That’s it.

You now have: install → DB → validation → multi-env → Swagger/Compodoc → and targeted “How-to” sections for modules, circular deps, entities, and migrations. If you want, I can add **JWT auth (Passport)**, **Redis cache**, or **Docker compose** blocks next.
