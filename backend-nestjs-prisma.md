# Backend architecture prompt — NestJS + Prisma + PostgreSQL

Copy everything below the line into your AI coding agent. Fill in **Section 0** first; everything else is the reusable architecture and should not be changed per project.

---

## 0. Project brief (fill this in)

- **Project name:** `<project name>`
- **What it does:** `<one or two sentences>`
- **User roles:** `<e.g. customer, vendor, admin>`
- **Modules / features to build (in order):**
  1. `<module>` — `<what it does>`
  2. `<module>` — `<what it does>`
- **Extra requirements:** `<e.g. email sign-in codes, payments, file uploads, identity verification — or "none">`

You are a senior backend engineer. Build the backend described in Section 0 using **exactly** the architecture below, so every module has the same structure, naming and rules. Do not add domain features that are not listed in Section 0.

## 1. Stack

- Node.js 20+, TypeScript (strict)
- NestJS 11 (`@nestjs/core`, `@nestjs/config`, `@nestjs/swagger`, `@nestjs/passport`, `@nestjs/throttler`)
- Prisma 6 + PostgreSQL
- class-validator / class-transformer for DTO validation
- passport-jwt for access tokens; refresh tokens stored as hashed sessions
- Jest for unit tests (`*.spec.ts` next to the file they test)
- Swagger at `/api/docs`, global prefix `/api/v1`

## 2. Folder structure

```
prisma/
  schema.prisma            # ONLY generator + datasource
  models/<module>.prisma   # one file per module that owns those tables
  migrations/
  seed.ts
src/
  main.ts                  # bootstrap (§5)
  app.module.ts            # imports every module + global interceptors
  config/configuration.ts  # typed config from env
  database/
    prisma.module.ts       # global PrismaService
    prisma.service.ts
    base.repository.ts     # generic typed CRUD every repository extends (§4)
    unit-of-work.ts        # transactions (§4)
  common/
    constants/             # shared constants/enums (e.g. system-roles.ts)
    decorators/            # @CurrentUser, @Roles, @Permissions, @ResponseMessage, @RawResponse
    dto/                   # pagination-query.dto.ts (+ Paginated<T>)
    filters/               # http-exception.filter.ts (error envelope)
    guards/                # jwt-auth.guard, roles.guard, permissions.guard
    http/                  # api-response.ts (envelope types), request-context.ts (requestId)
    interceptors/          # logging.interceptor.ts, transform-response.interceptor.ts
    types/                 # authenticated-user.type.ts
    validation/            # reusable custom validators
  health/                  # GET /health
  jobs/                    # scheduled jobs (cleanup etc.)
  modules/<module>/        # one folder per business module (§3)
```

Path aliases (`tsconfig.json`), used everywhere instead of long relative imports:
`@common/*` → `src/common/*`, `@config/*` → `src/config/*`, `@database/*` → `src/database/*`, `@modules/*` → `src/modules/*`.

## 3. Module layout (every module looks like this)

```
src/modules/<module>/
  <module>.module.ts
  <module>.controller.ts      # HTTP only: routes, guards, DTOs, Swagger
  <module>.service.ts         # business rules
  <model>.repository.ts       # one per model this module OWNS
  dto/<thing>.dto.ts          # class-validator DTOs (+ @ApiProperty)
  <module>.constants.ts       # optional
  <module>.service.spec.ts    # unit tests (repositories mocked)
```

Rules:
- **Controller:** no business logic. Reads `@CurrentUser()`, `@Param`, `@Body`, calls one service method. Protected with `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(...)` or `PermissionsGuard` + `@Permissions(...)`. Has `@ApiTags`, `@ApiBearerAuth`.
- **Service:** owns the rules (ownership checks, state transitions, notifications). Never queries Prisma directly for a model that has a repository.
- **Repository:** the only place that queries its model.
- **Boundaries:** a repository is a provider of its module and is **not exported**. Other modules import the module and call its **exported service**:
  ```ts
  @Module({
    controllers: [OrdersController],
    providers: [OrdersService, OrdersRepository], // repository stays private
    exports: [OrdersService],
  })
  export class OrdersModule {}
  ```
- Register every module in `app.module.ts`.

## 4. Data access: BaseRepository + UnitOfWork

**`src/database/base.repository.ts`** — a generic class typed from Prisma's generated types (no `any`):

```ts
export type Db = PrismaService | Prisma.TransactionClient;

export abstract class BaseRepository<M extends Prisma.ModelName> {
  protected constructor(prisma: PrismaService, delegate: (db: Db) => Delegate<M>, options?: { softDelete?: boolean });
  protected model(tx?: Db): Delegate<M>;   // fully typed Prisma delegate for custom queries
  findById(id, tx?) / findOne(where, tx?) / findMany({ where, orderBy, skip, take }, tx?)
  count(where?, tx?) / exists(where, tx?)
  paginate({ where, orderBy, skip, take }, tx?): Promise<Paginated<Entity<M>>>   // { items, total, skip, take }
  create(data, tx?) / createMany(data[], tx?)
  update(id, data, tx?) / updateMany(where, data, tx?)
  delete(id, tx?) / deleteMany(where, tx?)
}
```

- Derive `Entity<M>`, `WhereInput<M>`, `OrderByInput<M>`, `CreateInput<M>`, `UpdateInput<M>` from `Prisma.TypeMap['model'][M]`.
- Optional `softDelete`: reads skip rows with `deletedAt`; deletes set `deletedAt`. Off by default.
- **Every method accepts an optional `tx`** so it can join a transaction.
- Queries the base doesn't cover (`select`, `include`, `groupBy`, raw SQL) are named methods on the concrete repository using `this.model(tx)`.

```ts
@Injectable()
export class OrdersRepository extends BaseRepository<'Order'> {
  constructor(prisma: PrismaService) {
    super(prisma, (db) => db.order);
  }
  listForCustomer(customerId: string, tx?: Db) {
    return this.model(tx).findMany({ where: { customerId }, orderBy: { createdAt: 'desc' } });
  }
}
```

**`src/database/unit-of-work.ts`** — transactions across repositories:

```ts
@Injectable()
export class UnitOfWork {
  constructor(private readonly prisma: PrismaService) {}
  run<T>(work: (tx: Prisma.TransactionClient) => Promise<T>, options?) {
    return this.prisma.$transaction(work, options);
  }
  batch(operations) {
    return this.prisma.$transaction(operations);
  }
}

// in a service
await this.uow.run(async (tx) => {
  const user = await this.users.create(data, tx);
  await this.userRoles.create({ userId: user.id, roleId }, tx);
});
```

Include a unit test for the base repository.

## 5. HTTP conventions

**`main.ts`:**
- `NestFactory.create(AppModule, { rawBody: true })` (raw body available for webhooks)
- `trust proxy` from env; JSON body limit (e.g. `2mb`)
- middleware that sets `req.startedAt` and a **requestId** (reuse incoming `x-request-id` or generate one; echo it in the response header)
- `app.setGlobalPrefix('api/v1')`
- `ValidationPipe({ whitelist: true, transform: true, forbidNonWhitelisted: true })`
- `app.useGlobalFilters(new HttpExceptionFilter())`
- CORS; Swagger (`DocumentBuilder().addBearerAuth()`) at `api/docs`

**`app.module.ts` providers:**
```ts
{ provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
{ provide: APP_INTERCEPTOR, useClass: TransformResponseInterceptor },
```

**Success envelope** (`TransformResponseInterceptor`) — handlers return plain data; the interceptor wraps it:
```json
{ "statusCode": 200, "message": "Success", "data": { }, "errors": null }
```
- `@ResponseMessage('Order created')` sets `message` (default `"Success"`).
- `@RawResponse()` skips the envelope (files, webhooks, health). `StreamableFile` passes through.

**Error envelope** (`HttpExceptionFilter`; types in `common/http/api-response.ts`):
```json
{ "statusCode": 400, "message": "Validation failed", "data": null,
  "errors": ["email must be an email"], "code": "OPTIONAL_MACHINE_CODE",
  "details": {}, "timestamp": "…", "path": "/api/v1/…", "requestId": "…" }
```
- Validation errors → `message: "Validation failed"` and every reason in `errors`.
- Business errors: `throw new ForbiddenException({ code: 'SOME_CODE', message: '…' })` — `code` is passed through for clients to branch on.
- Unknown errors → 500 `"Internal server error"`; never leak stack traces or internals.

**`LoggingInterceptor`:** one line per request — method, URL, status, duration, requestId.

> Clients must unwrap `data` from the success envelope and read `errors[]` for messages — once, in their API layer.

## 6. Prisma schema (multi-file)

- `package.json`: `"prisma": { "schema": "prisma" }` so the CLI reads the folder.
- `prisma/schema.prisma` holds only `generator client` + `datasource db`.
- Each module's models live in `prisma/models/<module>.prisma`; models can reference models in other files.
- Every table: `id` (`cuid()`/`uuid()`), `createdAt`, `updatedAt`.
- New tables: edit `prisma/models/<module>.prisma` → `npx prisma migrate dev --name <change>` (local only) → commit the migration.
- Shared/deployed databases: `npx prisma migrate deploy` (never `migrate dev`), then `npx prisma generate`.

## 7. Auth & access control

- **Roles as data** (`Role`, `UserRole`), not a hard-coded enum. System roles seeded with `isSystem = true` and referenced in code via a `SystemRole` constant.
- **Permissions as data** (`Permission`, `RolePermission`); `PermissionsGuard` + `@Permissions('resource.action')`.
- Endpoints: `POST /auth/signup`, `POST /auth/login`, `POST /auth/refresh`, `POST /auth/logout`, `GET /auth/me`.
- Optional (if Section 0 asks): second factor by emailed one-time code — login returns a `challengeToken`, `POST /auth/2fa/verify` returns tokens, `POST /auth/2fa/resend`; store only hashes, with expiry and an attempt limit.
- `Session` table stores a **hash** of each refresh token; logout deletes it; a scheduled job cleans expired sessions/challenges.
- `JwtAuthGuard` + `RolesGuard` + `@CurrentUser()` → `AuthenticatedUser`.
- Rate-limit auth endpoints (`@nestjs/throttler`).
- Any rule that blocks sign-in (unverified email, pending verification, suspended account) is enforced in `AuthService.login()` **before** tokens or codes are issued, and returns `403` with a `code`.

## 8. Config & secrets

- All env vars are read in `src/config/configuration.ts` (typed) and accessed through `ConfigService` — no `process.env` scattered in services.
- `.env` is never committed; `.env.example` documents every variable.
- Secrets (JWT secrets, third-party private keys, SMTP passwords) stay on the backend only.

## 9. Quality rules

- TypeScript strict, no `any` (use Prisma's generated types).
- Every service has a spec that mocks its repositories and tests the business rules.
- DTOs validate every input (`whitelist` strips unknown fields).
- One responsibility per file; comments explain *why*.
- Keep `docs/ARCHITECTURE.md` updated with these rules and the module list.

## 10. Build order

1. **Foundation:** `main.ts`, config, multi-file Prisma, `BaseRepository`, `UnitOfWork`, envelope interceptor, exception filter, logging, requestId, health check, Swagger.
2. **Users, Roles, Permissions** (+ seed).
3. **Auth** (signup, login, refresh, sessions, `/auth/me`, plus anything from Section 0).
4. **The modules in Section 0**, one at a time.

For each module: Prisma model file → migration → repository → service (+ spec) → DTOs → controller → register in `app.module.ts` → update `docs/ARCHITECTURE.md`. Run `npm run build` and `npm test` after each module and fix everything before moving on. Ask before adding any dependency not listed in §1.
