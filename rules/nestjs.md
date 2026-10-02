# NestJS Standards & Conventions

## Project Structure

- Organize code by **feature modules**, not by type. Each domain feature gets its own folder (e.g., `cats/`) with its own module, controller, service, DTOs, interfaces, and filters co-located.
- Every module folder should follow this structure:
  ```
  feature/
    dto/
      create-feature.dto.ts
      update-feature.dto.ts
    interfaces/
      feature.interface.ts
    feature.controller.ts
    feature.module.ts
    feature.service.ts
  ```
- Use the Nest CLI to scaffold: `nest g module`, `nest g controller`, `nest g service`, `nest g resource`.

---

## Modules

- Every feature must be encapsulated in a `@Module()` class.
- `providers`, `controllers`, `imports`, and `exports` must be explicit — never rely on magic discovery.
- Only export providers that need to be consumed by other modules.
- Use `@Global()` sparingly (only for infrastructure-level concerns like config or logging) — it is not a good design decision for feature providers.
- Use `forRoot()` / `forRootAsync()` patterns on dynamic modules to accept configuration. Use `ConfigurableModuleBuilder` for standard config patterns.
- Register global-scoped providers (filters, guards, interceptors, pipes) via the module's `providers` array using `APP_FILTER`, `APP_GUARD`, `APP_INTERCEPTOR`, `APP_PIPE` tokens — **not** via `useGlobalX()` in `main.ts` when dependency injection is needed.
- Avoid circular module dependencies; when unavoidable between providers, use `forwardRef()` both at module import and injection sites (`@Inject(forwardRef(() => Service))`) to prevent bootstrap failures.

---

## Configuration

- Use `@nestjs/config` for centralized, environment-aware configuration across the application.
- Always validate environment variables at startup using a validation schema (via Joi or `class-validator` + `plainToInstance`) to fail fast on invalid deployments.
- Inject `ConfigService` or define strongly typed configuration namespaces using `registerAs()` to maximize type safety and prevent runtime property typos.
- Avoid accessing `process.env` directly in services or controllers — route all environment access through the configuration module.

---

## Middleware

- Implement `NestMiddleware` for pre-routing concerns requiring raw HTTP request/response manipulation (e.g., request correlation IDs, logging raw payloads, header sanitation).
- Configure middleware in module classes implementing `NestModule` via the `configure(consumer: MiddlewareConsumer)` method.
- Scope middleware intentionally using `consumer.apply(...).forRoutes(...)` or `consumer.apply(...).exclude(...)`.
- Prefer **Guards** for authentication/authorization and **Interceptors** for response transformation/lifecycle hooks, as middleware lacks Nest's DI-rich `ExecutionContext`.

---

## Controllers

- Controllers handle HTTP routing only. No business logic.
- Use `@Controller('resource')` with a route prefix. Route paths should be kebab-case.
- Always use the **standard (recommended) response approach** — return values from handlers directly. Avoid `@Res()` unless you need full low-level control; if you must use it, set `passthrough: true`.
- Use `@Get()`, `@Post()`, `@Put()`, `@Patch()`, `@Delete()` — never `@All()` unless intentionally handling all methods.
- Use `@Param()`, `@Query()`, `@Body()`, `@Headers()` decorators for accessing request data — never inject the raw `@Req()` object into handlers unless absolutely necessary.
- **Route ordering matters on Express** (the default adapter). Declare specific/static routes before parametric ones (e.g., `@Get('me')` before `@Get(':id')`). Enable `routeConflictPolicy` and `routeResolutionStrategy: 'specificity'` in `NestApplicationOptions` to catch conflicts at bootstrap.
- Use `@HttpCode()` to override the default status code where the default (200/201) is not appropriate.
- Define DTOs as **classes** (not interfaces or plain types) so they are available at runtime for pipes and validators.
- Use `async/await` on all async handlers. Return `Promise<T>` or an RxJS `Observable<T>`.

---

## Providers & Dependency Injection

- Mark all injectable classes with `@Injectable()`.
- Always use **constructor-based injection**. Avoid property-based injection (`@Inject()` on a property) except when extending a base class where constructor injection is not possible.
- Inject by **type** wherever possible. Use `@Inject(TOKEN)` only for non-class-based tokens (strings/Symbols).
- Define string/Symbol injection tokens as named constants in a `constants.ts` file — never use inline string literals at the injection site.
- Prefer **abstract classes** over TypeScript interfaces as DI tokens — abstract classes survive compilation and avoid the need for `@Inject()`.
- Use `useClass`, `useValue`, `useFactory`, or `useExisting` for custom provider registration when the default `@Injectable()` class pattern is insufficient.
- Providers are **singletons** by default. Only change scope to `REQUEST` or `TRANSIENT` when there is a concrete, justified need (e.g., per-request caching, multi-tenancy).
- Do not inject module classes into providers — this creates circular dependency risk.

---

## Lifecycle Hooks

- Implement standard lifecycle interfaces (`OnModuleInit`, `OnApplicationBootstrap`, `OnModuleDestroy`, `BeforeApplicationShutdown`, `OnApplicationShutdown`) on providers managing external connections (databases, message brokers, caching tiers).
- Always clean up open handles, sockets, timers, and active connections in teardown hooks (`OnModuleDestroy`, `OnApplicationShutdown`) to prevent connection leaks during rolling updates.
- Enable shutdown hooks in `main.ts` via `app.enableShutdownHooks()` so lifecycle destruction signals (`SIGTERM`, `SIGINT`) are handled gracefully by container orchestrators.

---

## Custom Decorators

- Use `createParamDecorator()` to extract and transform request data into clean, reusable parameter decorators (e.g., `@CurrentUser()`, `@ClientIp()`, `@Tenant()`).
- Combine `createParamDecorator()` with validation pipes when validating extracted data dynamically at the route parameter level.
- Keep custom param decorator functions pure and focused exclusively on request extraction; do not execute side effects or database lookups inside them.
- Use `applyDecorators()` to compose multiple common decorators (e.g., bundling Auth guards, role requirements, and OpenAPI responses into a single `@Auth(...)` decorator) to reduce duplication and cognitive load.

---

## DTOs & Validation

- All DTOs must be **classes** annotated with `class-validator` decorators.
- Enable global validation via `ValidationPipe` in `main.ts`:

```ts
app.useGlobalPipes(
  new ValidationPipe({
    whitelist: true,        // strip unknown properties
    forbidNonWhitelisted: true, // throw on unknown properties
    transform: true,        // auto-transform payloads to DTO instances
  }),
);
```

- Always set `whitelist: true` and `forbidNonWhitelisted: true` to prevent mass-assignment vulnerabilities.
- Use `transform: true` so that route params and query strings are automatically coerced to their declared types.
- Use `@nestjs/mapped-types` utilities (`PartialType`, `PickType`, `OmitType`, `IntersectionType`) to derive update/partial DTOs from create DTOs — do not manually duplicate fields.
- For array body payloads, use `ParseArrayPipe` with `items: Dto` instead of raw `T[]`.
- Prefer `class-validator` + `ValidationPipe` for class-based DTOs. Prefer `StandardSchemaValidationPipe` with Zod/Valibot when schemas already exist outside class-based DTOs.

---

## Serialization

- Use `ClassSerializerInterceptor` to standardize response payload transformation across the application.
- Annotate entity and response DTO classes with `class-transformer` decorators (`@Exclude()`, `@Expose()`, `@Transform()`) to prevent sensitive data (passwords, tokens, internal flags) from accidentally leaking to clients.
- Register `ClassSerializerInterceptor` globally via `APP_INTERCEPTOR` or bind it to specific controllers/handlers returning sensitive models.

---

## OpenAPI & Swagger

- Decorate DTO properties with `@ApiProperty()` or `@ApiPropertyOptional()` to provide clear, schema-accurate API documentation.
- Enable the `@nestjs/swagger` CLI plugin (`@nestjs/swagger/plugin`) in `nest-cli.json` to automatically generate OpenAPI metadata from TypeScript AST and eliminate repetitive decorator boilerplate.
- Use `@ApiOperation()`, `@ApiResponse()`, and `@ApiTags()` on controllers to ensure contract clarity and seamless integration for client SDK generators and frontend consumers.

---

## Pipes

- Use built-in `Parse*` pipes (`ParseIntPipe`, `ParseUUIDPipe`, `ParseBoolPipe`, etc.) on individual parameters before writing custom parse logic.
- Apply `DefaultValuePipe` before `Parse*` pipes when a query parameter is optional with a known default.
- Custom pipes must implement `PipeTransform` and be decorated with `@Injectable()`.
- Pipe errors should always throw a `BadRequestException` (or an appropriate `HttpException`).
- Bind pipes at the narrowest scope needed: `parameter` > `method` > `controller` > `global`.

---

## Guards

- Guards are primarily for authorization; when using strategy-based authentication (e.g., Passport), it's idiomatic for the guard to delegate to the strategy rather than implement auth logic inline.
- Guards must implement `CanActivate` and be decorated with `@Injectable()`.
- Use `Reflector.createDecorator<T>()` to define custom metadata decorators (e.g., `@Roles(['admin'])`), then read them inside the guard via `Reflector`.
- Explicitly throw `UnauthorizedException` from guards/strategies handling authentication (401), and rely on the framework's default `ForbiddenException` — or throw it explicitly — for authorization failures (403).
- Register global guards via `APP_GUARD` in the module `providers` array to preserve DI.

---

## Interceptors

- Interceptors handle cross-cutting concerns: logging, response transformation, caching, timeout, error mapping.
- Interceptors must implement `NestInterceptor` and be decorated with `@Injectable()`.
- Always call and return `next.handle()` (the Observable) — use RxJS operators (`tap`, `map`, `catchError`, `timeout`) to transform or observe it.
- Use `map()` to normalise response shapes (e.g., wrap all responses in `{ data: T }`).
- Use `timeout()` + `catchError()` for enforcing per-route timeouts.
- Register global interceptors via `APP_INTERCEPTOR` in the module `providers` array to preserve DI.
- Do not use `@Res()` on a handler decorated with an interceptor that maps the response — they are mutually exclusive unless `passthrough: true` is set.

---

## Exception Filters

- Throw built-in `HttpException` subclasses (`NotFoundException`, `BadRequestException`, `UnauthorizedException`, etc.) from services and controllers — never raw `Error` objects into HTTP routes.
- Pass a machine-readable `errorCode` via the third options argument for client-consumable error codes:

```ts
throw new BadRequestException('Password too weak', { errorCode: 'WEAK_PASSWORD' });
```

- Custom exception filters must implement `ExceptionFilter` and be decorated with `@Catch(ExceptionType)`.
- For a catch-all filter, use `@Catch()` with no arguments and use `HttpAdapterHost` to send the response in a platform-agnostic way.
- Register global filters via `APP_FILTER` in the module `providers` array to preserve DI.
- Extend `BaseExceptionFilter` when you want to delegate handling back to the built-in filter after custom logic.

---

## Error Handling in Services

- Services should throw domain-meaningful exceptions (`NotFoundException`, `ConflictException`, etc.) — do not return `null` for "not found" and handle it in the controller.
- Pass `{ cause: originalError }` in the third argument of `HttpException` constructors to preserve the error chain for logging without exposing internals to clients.

---

## Logging

- Use the built-in `Logger` class (instantiated with a dedicated context, e.g., `new Logger(CatsService.name)`) in services and controllers to provide searchable, contextual log traces.
- For high-load or distributed production environments, replace the default logger in `NestFactory.create()` with a high-throughput structured logging library (e.g., `nestjs-pino` or Winston) to optimize JSON log indexing and reduce I/O overhead.
- Explicitly pass error objects and stack traces to `logger.error(message, error.stack, context)` to preserve diagnosability during incident investigation.

---

## Microservices & Background Processing

- Use `@nestjs/microservices` when decomposing monolithic operations into event-driven or RPC-based architectures (via transports such as Redis, NATS, Kafka, or RabbitMQ).
- Use `@nestjs/bullmq` (or `@nestjs/bull`) for background jobs, asynchronous batch processing, and rate-limited tasks to keep HTTP request-response cycles fast and resilient.
- Encapsulate message patterns (`@MessagePattern()`), event subscribers (`@EventPattern()`), and queue workers (`@Processor()`) in isolated worker modules distinct from HTTP controllers.

---

## TypeScript

- Always use strict TypeScript. Enable `strict: true` and `strictNullChecks: true`.
- Use `readonly` on injected dependencies in constructors: `constructor(private readonly catsService: CatsService)`.
- Never use `any` — use `unknown` and narrow the type, or define an explicit interface/type.
- Import types with `import type` where the value is only used as a type, to avoid unintended runtime imports.
- Avoid `emitDecoratorMetadata: false` — it must be true for Nest's DI system to function.
- Use `as const` assertions on tuple literals (e.g., in `PickType`/`OmitType` field arrays).

---

## Testing

- Unit test services in isolation by mocking all dependencies using custom providers with `useValue: mockService`.
- Use `Test.createTestingModule()` from `@nestjs/testing` for all Nest-aware tests.
- End-to-end test controllers via `supertest` on the full Nest app, not by calling controller methods directly.
- Prefer injecting mock providers over patching module internals at test time.
