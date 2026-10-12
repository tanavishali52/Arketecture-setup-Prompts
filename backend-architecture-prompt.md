# Prompt: Build a NestJS + Prisma backend (PETS-BE architecture)

Copy everything below the line into Claude (or any AI coding assistant). Replace the `<…>` placeholders first.

---

You are a senior backend engineer. Build the backend for **<PROJECT NAME>** — <one-line description, e.g. "a pet-services marketplace with clients, providers (groomer / carer / vet), sellers and admins">.

Follow the architecture below exactly. It is the architecture of an existing production codebase (PETS-BE); keep its structure, naming and rules so every module looks the same.

## 1. Stack

- **Node.js 20+, TypeScript (strict)**
- **NestJS 11** (`@nestjs/core`, `@nestjs/config`, `@nestjs/swagger`, `@nestjs/passport`, `@nestjs/throttler`)
- **Prisma 6** + **PostgreSQL**
- **class-validator / class-transformer** for DTO validation
- **passport-jwt** for access tokens; refresh tokens stored as hashed sessions
- **Jest** for unit tests (`*.spec.ts` next to the file they test)
- Swagger at `/api/docs`, global prefix `/api/v1`

## 2. Folder structure

```
prisma/
  schema.prisma            # ONLY generator + datasource
  models/<module>.prisma   # one file per module that owns those tables
  migrations/
  seed.ts
src/
  main.ts                  # bootstrap (see §5)
  app.module.ts            # imports every module + global interceptors
  config/configuration.ts  # typed config from env (ConfigModule.forRoot({ isGlobal, load }))
  database/
    prisma.module.ts       # global PrismaService
    prisma.service.ts
    base.repository.ts     # generic typed CRUD every repository extends (§4)
    unit-of-work.ts        # transactions (§4)
  common/
    constants/             # e.g. system-roles.ts, enums shared across modules
    decorators/            # @CurrentUser, @Roles, @Permissions, @ResponseMessage, @RawResponse
    dto/                   # pagination-query.dto.ts (+ Paginated<T> type)
    filters/               # http-exception.filter.ts (error envelope)
    guards/                # jwt-auth.guard, roles.guard, permissions.guard
    http/                  # api-response.ts (envelope types), request-context.ts (requestId)
    interceptors/          # logging.interceptor.ts, transform-response.interceptor.ts
    types/                 # authenticated-user.type.ts
    validation/            # reusable validators (e.g. UK postcode/phone)
  health/                  # GET /health
  jobs/                    # scheduled jobs (cleanup etc.)
  modules/<module>/        # one folder per business module (§3)
```

Path aliases in `tsconfig.json` (use them everywhere, no long relative imports across folders):
`@common/*` → `src/common/*`, `@config/*` → `src/config/*`, `@database/*` → `src/database/*`, `@modules/*` → `src/modules/*`.

## 3. Module layout (every module looks like this)

```
src/modules/<module>/
  <module>.module.ts
  <module>.controller.ts      # HTTP only: routes, guards, DTOs, Swagger tags
  <module>.service.ts         # business rules
  <model>.repository.ts       # one per model this module OWNS
  dto/<thing>.dto.ts          # class-validator DTOs (+ @ApiProperty)
  <module>.constants.ts       # optional
  <module>.service.spec.ts    # unit tests (mock the repository)
```

Rules:
- **Controller** does no business logic: it reads `@CurrentUser()`, `@Param`, `@Body`, and calls one service method. Protect with `@UseGuards(JwtAuthGuard, RolesGuard)` + `@Roles(...)` (or `PermissionsGuard` + `@Permissions(...)`). Add `@ApiTags`, `@ApiBearerAuth`.
- **Service** holds the rules (ownership checks, state changes, emails). It never calls `PrismaService` for a model that has a repository — it calls the repository.
- **Repository** is the only place that queries its model.
- **Module boundaries:** a repository is a provider of its module and is **NOT exported**. Other modules import the module and call its **exported service**. Example:
  ```ts
  @Module({
    controllers: [PetsController],
    providers: [PetsService, PetsRepository], // repository stays private
    exports: [PetsService],
  })
  export class PetsModule {}
  ```
- Register every module in `app.module.ts`.

## 4. Data access: BaseRepository + UnitOfWork

**`src/database/base.repository.ts`** — a generic class typed from Prisma's generated types (no `any`):

```ts
export type Db = PrismaService | Prisma.TransactionClient;

export abstract class BaseRepository<M extends Prisma.ModelName> {
  protected constructor(prisma: PrismaService, delegate: (db: Db) => Delegate<M>, options?: { softDelete?: boolean });
  protected model(tx?: Db): Delegate<M>;          // fully typed Prisma delegate, for custom queries
  findById(id, tx?) / findOne(where, tx?) / findMany({ where, orderBy, skip, take }, tx?)
  count(where?, tx?) / exists(where, tx?)
  paginate({ where, orderBy, skip, take }, tx?): Promise<Paginated<Entity<M>>>  // { items, total, skip, take }
  create(data, tx?) / createMany(data[], tx?)
  update(id, data, tx?) / updateMany(where, data, tx?)
  delete(id, tx?) / deleteMany(where, tx?)
}
```

- Derive `Entity<M>`, `WhereInput<M>`, `OrderByInput<M>`, `CreateInput<M>`, `UpdateInput<M>` from `Prisma.TypeMap['model'][M]`.
- Optional `softDelete`: reads skip rows with `deletedAt`, deletes set `deletedAt` (off by default).
- **Every method takes an optional `tx`** so it can join a transaction.
- Queries the base doesn't cover (`select`, `include`, `groupBy`, raw SQL) are **named methods on the concrete repository** using `this.model(tx)`.

Concrete repository:
```ts
@Injectable()
export class PetsRepository extends BaseRepository<'Pet'> {
  constructor(prisma: PrismaService) {
    super(prisma, (db) => db.pet);
  }
  listForOwner(ownerId: string, tx?: Db) {
    return this.model(tx).findMany({ where: { ownerId }, orderBy: { createdAt: 'desc' } });
  }
}
```

**`src/database/unit-of-work.ts`** — transactions across repositories:
```ts
@Injectable()
export class UnitOfWork {
  constructor(private readonly prisma: PrismaService) {}
  run<T>(work: (tx: Prisma.TransactionClient) => Promise<T>, options?) { return this.prisma.$transaction(work, options); }
  batch(operations) { return this.prisma.$transaction(operations); }
}
// usage in a service:
await this.uow.run(async (tx) => {
  const user = await this.users.create(data, tx);
  await this.userRoles.create({ userId: user.id, roleId }, tx);
});
```

Add a unit test for the base repository (`base.repository.spec.ts`).

## 5. HTTP conventions (response envelope, errors, logging)

**`main.ts`**:
- `NestFactory.create(AppModule, { rawBody: true })` (raw body for webhooks)
- `trust proxy` from env, JSON body limit `2mb`
- a middleware that sets `req.startedAt` and a **requestId** (reuse `x-request-id` header or generate one, echo it back in the response header)
- `app.setGlobalPrefix('api/v1')`
- `ValidationPipe({ whitelist: true, transform: true, forbidNonWhitelisted: true })`
- `app.useGlobalFilters(new HttpExceptionFilter())`
- CORS, Swagger (`DocumentBuilder().addBearerAuth()`, served at `api/docs`)

**`app.module.ts`** providers:
```ts
{ provide: APP_INTERCEPTOR, useClass: LoggingInterceptor },
{ provide: APP_INTERCEPTOR, useClass: TransformResponseInterceptor },
```

**Success envelope** (`TransformResponseInterceptor`) — handlers return plain data, the interceptor wraps it:
```json
{ "statusCode": 200, "message": "Success", "data": { ... }, "errors": null }
```
- `@ResponseMessage('Pet created')` sets `message` (default "Success").
- `@RawResponse()` skips the envelope (files, webhooks, health); `StreamableFile` passes through.

**Error envelope** (`HttpExceptionFilter`, types in `common/http/api-response.ts`):
```json
{ "statusCode": 400, "message": "Validation failed", "data": null,
  "errors": ["email must be an email"], "code": "OPTIONAL_MACHINE_CODE",
  "details": {}, "timestamp": "...", "path": "/api/v1/...", "requestId": "..." }
```
- Validation errors → `message: "Validation failed"`, `errors: [...]` with each reason.
- Business errors: `throw new ForbiddenException({ code: 'SOME_CODE', message: '...' })` → `code` is passed through.
- Unknown errors → 500 `"Internal server error"`; never leak stack traces or internal details in production.

**`LoggingInterceptor`**: one line per request — method, URL, status, duration (ms), requestId. Errors are logged by the filter.

> Frontend note: clients must unwrap `data` from the success envelope and read `errors[]` for messages — do it once in the frontend's API layer.

## 6. Prisma schema (multi-file)

- `package.json`: `"prisma": { "schema": "prisma" }` so the CLI reads the whole folder.
- `prisma/schema.prisma` holds only `generator client` + `datasource db`.
- Each module's models live in `prisma/models/<module>.prisma` (users, roles, permissions, auth, account, billing, …). Models can reference models in other files without imports.
- New tables: add/extend `prisma/models/<module>.prisma` → `npx prisma migrate dev --name <change>` (local only) → commit the migration.
- Deploy / other machines: `npx prisma migrate deploy` (never `migrate dev` against shared DBs), then `npx prisma generate`.
- Use `@map`/`@@map` for snake_case table/column names if wanted; ids as `cuid()`/`uuid()`; `createdAt`/`updatedAt` on every table.

## 7. Auth & access control

- **Roles are data** (`Role`, `UserRole` tables), not a hard-coded enum; system roles (e.g. CLIENT, PROVIDER roles, ADMIN) seeded with `isSystem = true` and referenced in code via `SystemRole` constants.
- **Permissions are data** (`Permission`, `RolePermission`); `PermissionsGuard` + `@Permissions('users.read')`.
- **Login flow**: `POST /auth/login` (password check) → email a 6-digit code (store only a hash, with expiry + attempt limit) → `POST /auth/2fa/verify` returns `{ accessToken, refreshToken }`; `POST /auth/2fa/resend`; `POST /auth/refresh`; `POST /auth/logout`; `GET /auth/me`.
- Sessions table stores a **hash** of each refresh token; logout/revoke deletes it; a cleanup job removes expired sessions/challenges.
- `JwtAuthGuard` (passport-jwt) + `RolesGuard` + `@CurrentUser()` decorator returning `AuthenticatedUser`.
- Rate-limit auth endpoints with `@nestjs/throttler`.
- If identity verification (KYC/KYB) is required: decide per role whether sign-in is blocked until verified, enforce it in `AuthService.login()` **before** the code email is sent, and return `403` with a `code` (e.g. `IDENTITY_VERIFICATION_PENDING`).

## 8. Config & secrets

- All env vars read in `src/config/configuration.ts` (typed), accessed via `ConfigService` — never `process.env` scattered in services.
- `.env` is never committed; keep `.env.example` with every variable documented.
- Secrets (JWT secrets, API private keys, SMTP passwords) only on the backend, never in frontend env vars.

## 9. Quality rules

- TypeScript strict, no `any` (use Prisma's generated types).
- Every service has a `*.spec.ts` that mocks its repositories; test the business rules, not Prisma.
- DTOs validate everything coming in (`whitelist` strips unknown fields).
- One responsibility per file; comments explain *why*, not *what*.
- Keep `docs/ARCHITECTURE.md` updated with these rules and the module list.

## 10. What to build now

<List the modules/features for this project, e.g.:>
1. Foundation: `main.ts`, config, Prisma (multi-file), BaseRepository, UnitOfWork, envelope interceptor, exception filter, logging, requestId, health check, Swagger.
2. Users, Roles, Permissions (with seed).
3. Auth (login + email code + refresh + sessions + /auth/me).
4. <feature module 1> …
5. <feature module 2> …

Work module by module. For each one: Prisma model file → migration → repository → service (+ spec) → DTOs → controller → register in `app.module.ts` → update `docs/ARCHITECTURE.md`. After each module, run `npm run build` and `npm test` and fix everything before moving on. Ask me before adding any dependency not listed in §1.
