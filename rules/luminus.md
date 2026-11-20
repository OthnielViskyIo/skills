# Luminus Rules

These rules apply **only to luminus projects**: the repositories under `/Users/othnielvisky/work/luminus/`. Ignore this file everywhere else.

In a luminus project, this file **overrides** the technology rules files (`typescript.md`, `nestjs.md`, `mongodb.md`, `react-next.md`, `solid.md`). Those files hold the standard rules from the official docs and common best practice. Each section names the rules it overrides. Rules that this file doesn't mention still apply as written.

**Before applying an override, ask the user which way to go.** Explain the standard way and the luminus way, each with a code example, plus the tradeoffs. See "Luminus overrides" in `~/.claude/CLAUDE.md`.

---

## 1. No Optional Properties

Overrides: `typescript.md` §10, `nestjs.md` §DTOs & Validation (`PartialType`), §OpenAPI & Swagger (`@ApiPropertyOptional()`) and §Pipes (`DefaultValuePipe`), `mongodb.md` §3, §12 and §14 (migrating a new discriminator key), and `react-next.md` §3 (optional props).

### 1.1 TypeScript

- **Do not write optional properties (`prop?: T`).** Give every property a sensible default instead (`''`, `[]`, `0`, `false`; for a string enum or literal union too, `''`, e.g. `Segment | ''`), and make the type required.
- Don't get around this with `prop: T | undefined`, `Partial<T>` or a type that only adds the field in some cases. Those are optionals too.
- When variants differ (e.g. one segment has fields the other lacks), prefer one shape that always has every field, with the field defaulted where it doesn't apply, over separate variants with optional fields. This is the case even if the field has no meaning for some variants.
- Set the default **once, at the boundary** where data comes in (request DTO, response of an external API, database read). From there inward the type is required and the code needs no `??` or `?.` for that field.
- When an external source may leave a field out, type only the raw response as partial, and map it straight away to the required internal type. For an upstream API response, do this with a validated response DTO (§5):

```typescript
// ❌ optional leaks into the domain, every reader needs ?? '' or ?.
interface ContactDetails {
  firstName: string;
  companyName?: string; // professional only
}

// ✅ always present, '' when it doesn't apply
interface ContactDetails {
  firstName: string;
  companyName: string; // '' when not entered
}

// ✅ the external API may leave contactPersons out: default it at the boundary
const raw = await fetchCustomer(customerNumber); // Omit<Customer, 'contactPersons'> & Partial<Pick<Customer, 'contactPersons'>>
return { ...raw, contactPersons: raw.contactPersons ?? [] };
```

- The same goes for parameters. Use a default value (`limit = 20`) instead of `limit?: number`.
- An optional property is only acceptable when no sensible default exists, and you can't restructure the type to avoid it. **Ask the user before adding one**, and explain why a default doesn't work.
- Not covered: a function that returns `T | undefined` for "nothing found" (e.g. `array.find`), types from libraries you don't control, and React props that are destructured with a default (§1.4).

### 1.2 NestJS DTOs

- §1.1 applies to request and response DTOs too: no `prop?: T`, no `@IsOptional()`, no `@ApiPropertyOptional()`, no `@ValidateIf` to skip a missing field. Declare the property as required, accept it being left out or empty, and fill in a default with `@Transform`. Document the default in `@ApiProperty({ default })`:

```ts
// ❌
@ApiPropertyOptional()
@IsOptional()
@IsString()
companyName?: string;

// ✅ always present on the DTO, '' when the client doesn't send it
@ApiProperty({ default: '' })
@Expose()
@Transform(({ value }: { value: unknown }) => value ?? '')
@IsString()
@MaxLength(80)
companyName!: string;
```

- Use `@Expose()` only when needed: to apply a default when a value may be `undefined`, `''` or `null` (with `@Transform`), to map an incoming field name to a different internal name, or when the serializer runs with `excludeAll`. Otherwise, leave it off.
- When DTO variants differ (e.g. per segment), make every variant accept every field with the same default. Tighten validation only in the variant that needs the field (e.g. `@Length(10)` for professionals) by redeclaring it with `declare`. Don't leave the field out of the other variants.
- Response DTOs always contain every field, with the default where it doesn't apply.
- Don't derive DTOs with `PartialType` (it makes every field optional) without asking the user first. `PickType`, `OmitType` and `IntersectionType` are fine.
- Every query parameter that may be left out gets a default with `DefaultValuePipe`.
- An optional DTO property is only acceptable when no sensible default exists, e.g. a PATCH where a missing field must mean "don't change" and no default value can express that. Ask the user first.

### 1.3 MongoDB

- Give each field that may be left out a schema `default` (`''`, `[]`, `false`, ...), so that it is required in the document interface.
- When documents written before the field existed don't have it, default it in the repository on read.
- For a discriminator key (`mongodb.md` §14) that documents written before it existed don't have, put the default on the base schema (`default: <legacy variant>`) instead of in the repository. Mongoose picks the variant schema while it loads the document, before the repository sees it. Overrides the migration of `mongodb.md` §14 and §12.
- Ask the user before leaving a field optional.

### 1.4 React props

- A prop of our own components may be optional (`variant?:`) **only** if the component destructures it with a default value (`{ variant = "primary" }`). An optional prop without a default needs the user's approval first.
- Native props inherited from `ComponentPropsWithRef` are not covered.

---

## 2. DI Contracts Are Abstract Classes

Overrides: `typescript.md` §5 (interfaces for repository contracts) and §16 (abstract classes only for template methods).

- Contracts used as dependency injection tokens (repository ports, service contracts) are abstract classes with only abstract members, not interfaces. They survive compilation, so NestJS can inject them by type (`nestjs.md` §Providers & Dependency Injection).
- Everything else follows `typescript.md` §5 and §16 as written.

---

## 3. Unit Tests Mock Every Dependency

Overrides: `nestjs.md` §Testing (adds to it) and `mongodb.md` §13 (testing against a real MongoDB).

- Unit test each provider (controllers, use cases, services, repositories, guards, pipes) in isolation by mocking **all** injected dependencies using custom providers with `useValue: mockService`. This includes Mongoose models, provided via `getModelToken(Model.name)`.
- Set up mock behavior with `jest.spyOn(mockService, 'method').mockResolvedValue(...)`. Never call `.mockResolvedValue()` directly on the mock object. Never type mocks as `jest.Mocked<T>`.
- Assert the full call payloads sent to mocked dependencies where feasible, including query filters and nested objects.

Repositories:

- Unit test repositories with the Mongoose model mocked, not against a real MongoDB.
- Assert the exact query filter, update document and options (e.g. `runValidators: true`, `new: true`) passed to the model, not only the return value.
- Build the documents a mock returns with the real schema (`new SessionModel({...})`), so mapping runs against real hydrated documents.
- Cover the missing-record path (the model resolves `null`) for every update.
- Isolate tests: reset mocks between tests.

---

## 4. N-Layered Architecture

Overrides: `nestjs.md` §Project Structure (feature folders with `dto/` and `interfaces/`). Never import Mongoose or the MongoDB driver outside `infrastructure/`.

These rules govern the layering of NestJS backend services. The layering is pragmatic: **the database sits behind a port, everything else is injected directly.** The reference implementation is the LHS `contact-details` endpoint in `b2c-online-customer-onboarding-service` (`src/contexts/luminus-home-services`). When in doubt, read it and follow its shape.

- Dependency injection, error handling and testing follow `rules/nestjs.md` and §3 of this file.
- Database code follows `rules/mongodb.md`.

### 4.1 Layers and Folders

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

### 4.2 Dependency Direction

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

### 4.3 Building an Endpoint

Follow this flow for every new endpoint.

#### 4.3.1 Controller

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

#### 4.3.2 Use Case

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

#### 4.3.3 Repository Port and Adapter

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

#### 4.3.4 Return Path

- Return the repository result through the use case to the controller.
- Let the controller's response validator and HTTP exception filters enforce the API response contract.

#### 4.3.5 Wiring

- Register new use cases and providers in the owning Nest module.
- Bind ports with `{ provide: Port, useClass: Adapter }`, and existing contracts with `useExisting` (e.g. `{ provide: CmsService, useExisting: SanityService }`).
- Keep Swagger and response DTOs aligned with the actual payload.

---

### 4.4 Errors

- Follow `rules/nestjs.md` (§Exception Filters, §Error Handling in Services): throw exceptions and let the exception filters map them to HTTP responses.
- Do not return `Result<T, E>` / `Either` types.
- The repository adapter translates database errors and missing records (`rules/mongodb.md` §11).

---

### 4.5 Tests per Layer

Mocking follows §3: every injected dependency is mocked.

| Layer      | Cover                                                       |
|------------|-------------------------------------------------------------|
| Controller | DTO/HTTP validation and auth                                |
| Use case   | Branching and mapping                                       |
| Repository | Persistence calls, defaults and missing-record behavior     |
| Policy     | Every rule outcome, as a pure function                      |

---

### 4.6 Naming

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

### 4.7 Not Adopted from Clean Architecture

Do **not** apply these, even if another rules file suggests them:

- Application ports for non-database dependencies (notifiers, gateways, clocks, id generators, CMS).
- `Result<T, E>` return types for expected failures.
- Aggregates that own business decisions instead of use cases.
- Separate CQRS query ports and read models.

---

### 4.8 Common Violations — Flag These

- A Mongoose model, schema or repository adapter imported outside `infrastructure/` or the module.
- A controller with business rules or persistence, or one that calls more than one use case.
- A controller that rebuilds or mutates the use case result.
- A use case that queries the database without going through the repository port.
- A new application port created for a non-database dependency.
- `new ConcreteRepository()` anywhere outside the DI container.
- An update that ignores a missing record or omits `runValidators: true`.
- A repository that leaks `_id`, `__v` or other storage metadata to the response.
- A new use case or provider that is not registered in the owning module.

---

## 5. Validate Upstream Responses

Adds to: `nestjs.md` §DTOs & Validation (which covers only incoming requests) and `typescript.md` §12 (no `as` cast on an upstream response).

Overrides: `nestjs.md` §Exception Filters ("throw built-in `HttpException` subclasses, never raw `Error` objects into HTTP routes") and §Error Handling in Services (pass `{ cause }` to the `HttpException`). An invalid upstream response is thrown as a `CustomError` instead, which extends `Error`, so that every upstream failure is logged in one structured, queryable shape and answered with the same 502 body.

The response of an upstream API (customer service, ...) is untrusted input, just like a request body. Validate it where it comes in, in the client that calls the API, before any other code sees it. A missing field should fail there, loudly, as a 502. It should not slip through and cause a `TypeError` somewhere further down.

- **Check real data first.** Before writing the response DTO, call the API for real customers (several segments and states, from a non-production environment) and note which fields are always present and which are sometimes missing, `null` or empty. The upstream Swagger or TypeScript types are not enough. If you can't call the API, ask the user for real sample responses.
- **Write a response DTO class** for the upstream response, following §1.2:
  - Fields that real data shows may be missing, and that we can do without, are required on the DTO with a default from `@Transform` (`''`, `[]`, ...).
  - Fields we need get no default. If they are missing, validation fails.
- **Run it through the validator** in the client: `plainToInstance` and then `validateSync`. Don't cast the raw response with `as`.
- **Throw a `CustomError`** (`src/exceptions/custom-error.exception-filter.ts`) when validation fails, as the other upstream clients do:
  - The source is the upstream's `ErrorSources` entry, and the method is the client method's name. Add an entry when the upstream has none.
  - Put the errors in the payload as `{ error: '<upstream>-validation-errors', validationErrors: flattenValidationErrors(errors) }` (`flattenValidationErrors` from `@luminus/microservices-common`).
  - `CustomErrorExceptionFilter` logs `{ source, method, payload }` and answers with the status of the source's `case`. Give a new source a `case` that answers 502 with `{ message: 'response.validation-failed' }`. Without a `case`, the filter answers 500.
  - Every controller whose route can reach the client must list `CustomErrorExceptionFilter` in `@UseFilters`. Without it, Nest answers 500 and logs nothing useful.
  - The validation errors are logged, never sent to our client.
- Don't throw a `BadGatewayException` for this. No filter logs an `HttpException`, so its `cause` never reaches the logs.
- Return the validated DTO's values as the internal type. From there inward every field is present, with no `??` or `?.` (§1.1).

```ts
// ❌ cast: a missing field surfaces later as a TypeError in a use case
const customer = (await this.http.get(url)).data as Customer;

// ✅ response DTO: default what may be missing, require what we need
export class CustomerResponseDto {
  @IsString()
  @IsNotEmpty()
  customerNumber!: string; // required: no default

  @Transform(({ value }: { value: unknown }) => value ?? [])
  @ValidateNested({ each: true })
  @Type(() => ContactPersonResponseDto)
  contactPersons!: ContactPersonResponseDto[]; // missing for some customers
}

// ✅ in the client: validate, or throw a CustomError (the filter logs it and answers 502)
const customer = plainToInstance(CustomerResponseDto, (await this.http.get(url)).data);
const validationErrors = flattenValidationErrors(validateSync(customer, { whitelist: true }));
if (validationErrors.length > 0) {
  throw new CustomError(ErrorSources.B2CCustomerMicroservice, 'getCustomerDetails', {
    error: 'customer-validation-errors',
    validationErrors,
  });
}
return customer;

// ✅ in each controller that reaches the client
@UseFilters(AxiosExceptionFilter, CustomErrorExceptionFilter)
```

- Test the client with a mocked HTTP call (§3): a full response passes, a response without an optional field gets the default, and a response without a required field throws a `CustomError` with the source, method and flattened validation errors.
- Test the filter `case` (the source answers 502), and test a route that reaches the client: a `CustomError` from the client gives a 502, not a 500.
