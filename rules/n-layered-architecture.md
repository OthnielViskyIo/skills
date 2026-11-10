---
paths:
  - "**/*.ts"
---

# N-Layered Architecture Rules

These rules govern the layering of NestJS backend services. Follow them precisely unless the user explicitly overrides a specific rule.

The layering is pragmatic: **the database sits behind a port, everything else is injected directly.** The reference implementation is the LHS `contact-details` endpoint in `b2c-online-customer-onboarding-service` (`src/contexts/luminus-home-services`). When in doubt, read it and follow its shape.

- Dependency injection, error handling and testing follow `rules/nestjs.md`.
- Database code follows `rules/mongodb.md`.

---

## 1. Layers and Folders

Each bounded context lives in `src/contexts/<context>/`, with its Nest module at the root of that folder.

| Folder                                | Layer          | Contains                                                              |
|---------------------------------------|----------------|-----------------------------------------------------------------------|
| `presentation/http/controllers/`      | Presentation   | Controllers                                                           |
| `presentation/http/dtos/`             | Presentation   | Request and response DTOs (classes, `class-validator`, Swagger)       |
| `application/use-cases/`              | Application    | One use case per user intent                                          |
| `application/ports/`                  | Application    | Database repository ports (abstract classes) and their response types |
| `application/types/`                  | Application    | Typed use case inputs and queries                                     |
| `domain/types/`, `domain/enums/`      | Domain         | Entity shapes and enums                                               |
| `domain/policies/`                    | Domain         | Pure business rules that need no I/O                                  |
| `infrastructure/<store>/`             | Infrastructure | Repository adapters, schemas/models                                   |
| `<context>.module.ts`                 | Wiring         | Registers controllers, use cases and port → adapter bindings          |

Shared code lives outside the contexts: `src/shared/` (including shared service contracts in `src/shared/contracts/`), `src/enums/`, `src/interfaces/`, `src/exceptions/`, `src/interceptors/`.

---

## 2. Dependency Direction

```
presentation  →  application  →  domain
                     ↑
               infrastructure (implements application ports)
```

- **Domain** imports only other domain code and shared types/exceptions. No I/O, no NestJS, no Mongoose.
- **Application** may import domain code, its own ports and types, NestJS DI (`@Injectable`), and the existing service classes/tokens of non-database collaborators. It must **not** import repository adapters, Mongoose models or schemas.
- **Presentation** imports use cases and application types. It must **not** import infrastructure.
- **Infrastructure** implements application ports and may import domain types. Nothing imports a repository adapter except the module.
- Only the module binds a port to its adapter. Never instantiate a concrete adapter in a use case.

---

## 3. Building an Endpoint

Follow this flow for every new endpoint.

### 3.1 Controller

- Define the HTTP route and its Swagger, auth, error and response decorators (`@ApiOperation`, `@UseGuards`, `@UseFilters`, `@ResponseValidator`, ...).
- Validate request DTOs at the boundary (`ValidationPipe` with `transform`, `whitelist`, `forbidNonWhitelisted`).
- Derive trusted identity and context from the request (e.g. `@VerifiedBpClaim()`, `request.sessionID`), never from the body.
- Delegate to **exactly one** use case, passing typed plain inputs, never DTO instances with framework behavior or ORM documents.
- Return the use case result unchanged. No persistence, no business rules, no rebuilding or mutating the response.

```ts
public async saveContactDetails(
  @Query(validation) { type, segment }: LhsSessionQueryDto,
  @VerifiedBpClaim() customerNumber: string,
  @Body(validation) { contactDetails }: LhsContactDetailsRequestDto,
  @AcceptLanguage() language: Language,
  @Req() request: Request,
): Promise<LhsSessionResponseDto> {
  return this.updateContactDetails.execute(
    { sessionId: request.sessionID, type, segment, customerNumber },
    contactDetails,
    language,
  );
}
```

### 3.2 Use Case

- One class per user intent, named `<Verb><Noun>UseCase`, with a single public `execute()` method.
- Takes typed inputs from `application/types/`.
- Uses the **repository port** for every database operation. Never bypass it.
- Injects other collaborators (upstream clients, validators, CMS, maintenance services) **directly**, using their existing service classes/tokens. Do **not** create application ports for non-database dependencies.
- Owns feature-specific decisions and derived values (e.g. "a newly created session is returned without applying the update", "a to-be-planned date is moved to the start of the month").
- Extract a rule to `domain/policies/` as a pure function when it needs no I/O and is worth testing on its own (e.g. `validateAppointmentWindow`).
- Do not copy another feature's branching where the feature contract differs.

```ts
@Injectable()
export class UpdateLhsContactDetailsUseCase {
  constructor(
    private readonly repository: LhsSessionsRepository,
    private readonly cmsService: CmsService,
  ) {}

  public async execute(
    query: LhsSessionQuery,
    contactDetails: UpdateLhsContactDetailsInput,
    language: Language,
  ): Promise<LhsSessionResponse> {
    const { session, newlyCreatedSession } = await this.repository.findOneOrCreate(query);
    if (newlyCreatedSession) {
      return session;
    }

    const content = await this.cmsService.getBoilerMaintenanceProduct({
      language,
      segment: session.segment,
    });

    return this.repository.updateContactDetails(session.id, { ...contactDetails, errors: [] }, content);
  }
}
```

### 3.3 Repository Port and Adapter

- Declare database operations on the **port**: an abstract class in `application/ports/`, named after the entity by intent (`LhsSessionsRepository`), never after the technology. The response shape the application returns is defined next to it (`LhsSessionResponse`).
- Implement the port in the **adapter** in `infrastructure/<store>/`, named `<Tech><Port>` (`MongoLhsSessionsRepository`).
- The adapter owns:
  - database queries,
  - schema validation (`runValidators: true` on updates),
  - persistence-specific errors,
  - mapping stored documents to the application response shape (hide `_id`, `__v`, timestamps and other storage metadata).
- For updates: use the persisted entity id supplied by the use case, return the updated entity, and handle a missing record explicitly by throwing.

```ts
export abstract class LhsSessionsRepository {
  abstract findOneOrCreate(query: LhsSessionQuery): Promise<LhsSessionWithCreationStatus>;
  abstract updateContactDetails(id: string, contactDetails: LhsContactDetails, content: MappedBoilerMaintenanceProduct): Promise<LhsSessionResponse>;
}
```

### 3.4 Return Path

- Return the repository result through the use case to the controller.
- Let the controller's response validator and HTTP exception filters enforce the API response contract.

### 3.5 Wiring

- Register new use cases and providers in the owning Nest module.
- Bind ports with `{ provide: Port, useClass: Adapter }`, and existing contracts with `useExisting` (e.g. `{ provide: CmsService, useExisting: SanityService }`).
- Keep Swagger and response DTOs aligned with the actual payload.

---

## 4. Errors

- Follow `rules/nestjs.md` (§Exception Filters, §Error Handling in Services): throw exceptions and let the exception filters map them to HTTP responses.
- Do not return `Result<T, E>` / `Either` types.
- The repository adapter translates database errors and missing records (`rules/mongodb.md` §11).

---

## 5. Tests per Layer

Mocking follows `rules/nestjs.md` §Testing: every injected dependency is mocked.

| Layer      | Cover                                                       |
|------------|-------------------------------------------------------------|
| Controller | DTO/HTTP validation and auth                                |
| Use case   | Branching and mapping                                       |
| Repository | Persistence calls, defaults and missing-record behavior     |
| Policy     | Every rule outcome, as a pure function                      |

---

## 6. Naming

| Concept              | Convention                     | Example                         |
|----------------------|--------------------------------|---------------------------------|
| Use case             | `<Verb><Noun>UseCase`          | `UpdateLhsContactDetailsUseCase`|
| Use case input       | `<Verb><Noun>Input`            | `UpdateLhsContactDetailsInput`  |
| Repository port      | `<Entity>Repository`           | `LhsSessionsRepository`         |
| Repository adapter   | `<Tech><Entity>Repository`     | `MongoLhsSessionsRepository`    |
| Port response type   | `<Entity>Response`             | `LhsSessionResponse`            |
| Request DTO          | `<Noun>RequestDto`             | `LhsContactDetailsRequestDto`   |
| Response DTO         | `<Noun>ResponseDto`            | `LhsSessionResponseDto`         |
| Schema model         | `<Noun>Model`                  | `LhsSessionModel`               |

Files use kebab-case with a type suffix: `update-lhs-contact-details.use-case.ts`, `lhs-sessions.repository.ts`, `mongo-lhs-sessions.repository.ts`, `lhs-contact-details-request.dto.ts`.

---

## 7. Not Adopted from Clean Architecture

Do **not** apply these, even if another rules file suggests them:

- Application ports for non-database dependencies (notifiers, gateways, clocks, id generators, CMS).
- `Result<T, E>` return types for expected failures.
- Aggregates that own business decisions instead of use cases.
- Separate CQRS query ports and read models.

---

## 8. Common Violations — Flag These

- A Mongoose model, schema or repository adapter imported outside `infrastructure/` or the module.
- A controller with business rules or persistence, or one that calls more than one use case.
- A controller that rebuilds or mutates the use case result.
- A use case that queries the database without going through the repository port.
- A new application port created for a non-database dependency.
- `new ConcreteRepository()` anywhere outside the DI container.
- An update that ignores a missing record or omits `runValidators: true`.
- A repository that leaks `_id`, `__v` or other storage metadata to the response.
- A new use case or provider that is not registered in the owning module.
