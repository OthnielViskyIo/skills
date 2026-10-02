# Clean Architecture Rules

These rules govern Clean Architecture and DDD alignment for TypeScript/Nx backend and service projects. Follow them precisely and without exception unless the user explicitly overrides a specific rule.

---

## 1. The Dependency Rule (Cardinal Law)

Source code dependencies MUST only point inward. Inner layers have zero knowledge of outer layers.

```
[Frameworks & Drivers]  →  [Interface Adapters]  →  [Application]  →  [Domain]
```

> If a file in `domain/` imports from `infrastructure/`, `adapters/`, or a framework — it is a violation.
> If a file in `application/` imports from `infrastructure/` or a framework — it is a violation.
> No exceptions. No "just this once".

---

## 2. Layer Definitions & Responsibilities

### Domain (innermost)
- Aggregates, Entities, Value Objects, Domain Events, Domain Services
- **Domain Port interfaces**: Entity repository contracts and domain service contracts
- Pure business logic only — no I/O, no HTTP, no DB

**May import:** nothing outside this layer. Only other domain types.
**Must NOT import:** Express, Fastify, Mongoose, Prisma, Axios, NestJS decorators, or any Nx `infrastructure/` or `adapters/` lib.

### Application
- Use Cases / Commands & Query Handlers
- **Application Port interfaces**: External notifications, identity/session providers, payment gateways, clocks, ID generators, and read-only query ports
- Orchestrates domain objects to fulfil a single user intent
- Reads/writes via port interfaces (never concrete implementations)
- Produces/consumes DTOs at its boundary

**May import:** `domain/` types and port interfaces only.
**Must NOT import:** `infrastructure/`, `adapters/`, ORM models, HTTP clients, or framework-specific code.

### Interface Adapters
- Repository implementations (e.g., `MongooseUserRepository`)
- Controller / Presenter / Gateway / Query service implementations
- Transforms DTOs ↔ framework/DB models
- Maps domain errors / `Result` failures to HTTP status codes

**May import:** `application/` and `domain/` types, framework/ORM libraries, and shared technical infrastructure libs (e.g., db client, logger).
**Must NOT import:** other feature/domain adapters directly (no cross-adapter dependencies).

### Frameworks & Drivers (outermost)
- Express / Fastify / NestJS bootstrapping & controllers setup
- Mongoose schema definitions / Prisma schemas & migrations
- Nx `apps/` shells and server entry points
- All third-party framework wiring

**May import:** any inner layer.
**Is NOT imported by:** any inner layer.

---

## 3. Nx Monorepo Layer Mapping

| Nx path pattern                        | Layer                  | Description                                         |
|----------------------------------------|------------------------|-----------------------------------------------------|
| `apps/**`                              | Frameworks & Drivers   | Application entry points, server bootstrap          |
| `libs/*/src/infrastructure/**`         | Interface Adapters     | Repositories, API clients, DB schema mappings       |
| `libs/*/src/adapters/**`               | Interface Adapters     | Controllers, presenters, event subscribers          |
| `libs/*/src/application/ports/**`      | Application            | Application-specific port interfaces (not domain)   |
| `libs/*/src/application/**`            | Application            | Use cases, query handlers, application DTOs         |
| `libs/*/src/domain/**`                 | Domain                 | Aggregates, entities, value objects, domain ports   |
| `libs/shared/infrastructure/**`        | Shared Infrastructure  | Shared DB clients (Prisma/Mongo pool), logger, utils|

When generating a new file, place it in the correct library path before writing any code.

---

## 4. Crossing Boundaries Correctly

### Inbound (caller → use case / query handler)
- Always pass a **DTO** (plain object), never a framework model or ORM document.
- The application layer defines the DTO shape — the adapter maps to it.

### Outbound (use case → infrastructure)
- Use cases call a **port interface** defined either in `domain/` (domain ports) or `application/ports/` (application ports).
- The concrete implementation lives in `infrastructure/` or `adapters/`.
- Wire implementations via dependency injection — never instantiate concrete adapters inside a use case.

### ❌ Never do this
```ts
// In a use case — VIOLATION
import { UserModel } from '../../infrastructure/mongoose/UserModel';
import mongoose from 'mongoose';
```

### ✅ Do this instead
```ts
// In domain/ — define domain port
export interface UserRepository {
  findById(id: UserId): Promise<User | null>;
  save(user: User): Promise<void>;
}

// In application/ports/ — define application port
export interface EmailNotificationPort {
  sendWelcomeEmail(email: string): Promise<void>;
}

// In application/ — use the ports
export class RegisterUserUseCase {
  constructor(
    private readonly users: UserRepository,
    private readonly notifier: EmailNotificationPort
  ) {}
}

// In infrastructure/ — implement the ports
export class MongooseUserRepository implements UserRepository { /* ... */ }
export class SendgridEmailAdapter implements EmailNotificationPort { /* ... */ }
```

---

## 5. Domain Entities vs. DDD Aggregates

Clean Architecture "entities" and DDD aggregates are related but not identical:

- A DDD aggregate root IS a Clean Architecture entity — place it in `domain/`.
- An aggregate root enforces its own invariants. Use cases do NOT contain business rules.
- Domain Events are raised by aggregates, not by use cases.
- Use cases orchestrate; aggregates decide.

```ts
// ✅ Correct: invariant lives in the aggregate
class Order {
  addItem(item: OrderItem): Result<void, OrderClosedError> {
    if (this.isClosed) return Result.fail(new OrderClosedError());
    this.items.push(item);
    return Result.ok();
  }
}

// ❌ Wrong: invariant leaking into use case
class AddItemToOrderUseCase {
  async execute(orderId, item) {
    const order = await this.orders.findById(orderId);
    if (order.isClosed) throw new Error('Order is closed'); // ← belongs in aggregate
  }
}
```

---

## 6. CQRS & Read Models (Query Optimization)

Not every database interaction requires full DDD aggregate loading:

- **Commands (Mutations):** Must load aggregate roots via repositories to enforce business invariants and emit domain events.
- **Queries (Reads):** May bypass aggregate hydration. Define a query port in `application/ports/` (or dedicated Query Service) returning read-optimized DTOs directly from the database adapter.
- Queries must never mutate state or bypass business rules for write operations.

```ts
// In application/ports/
export interface OrderQueryPort {
  getOrderSummary(orderId: string): Promise<OrderSummaryView | null>;
  listRecentOrders(filter: OrderListFilter): Promise<PaginatedResult<OrderSummaryView>>;
}

// In application/
export class GetOrderSummaryQuery {
  constructor(private readonly orderQueries: OrderQueryPort) {}

  async execute(orderId: string): Promise<OrderSummaryView | null> {
    return this.orderQueries.getOrderSummary(orderId);
  }
}
```

---

## 7. Error Handling: `Result<T, E>` vs Exceptions

- **Expected Domain & Application Failures:** Prefer returning a typed `Result<T, E>` (or `Either<L, R>`) pattern rather than throwing untyped exceptions. This makes failure cases explicit in TypeScript signatures across boundaries.
- **Unexpected / Infrastructure Crashes:** Throw exceptions strictly for unexpected failures (database network down, unhandled null pointer, assertion crash).
- Adapters map `Result` failures to transport-specific responses (e.g. HTTP 400, 404, 409).

```ts
// ✅ Typed Result for business failures
export type CreateOrderResult = Result<OrderDto, OrderClosedError | InsufficientStockError>;

export class CreateOrderUseCase {
  async execute(input: CreateOrderInput): Promise<CreateOrderResult> {
    const orderResult = Order.create(input);
    if (orderResult.isFailure) {
      return Result.fail(orderResult.error);
    }
    await this.orderRepo.save(orderResult.value);
    return Result.ok(OrderMapper.toDto(orderResult.value));
  }
}
```

---

## 8. Port Interfaces (Domain vs Application Ports)

Distinguish ports by their layer of concern:

- **Domain Ports (`domain/`):**
  - Contracts required directly by business logic and aggregate persistence.
  - Examples: `UserRepository`, `ProductPricingService`.
- **Application Ports (`application/ports/`):**
  - Contracts required for application orchestration and external communications.
  - Examples: `EmailNotifier`, `PaymentGateway`, `Clock`, `IdGenerator`, `OrderQueryPort`.
- **Naming:** Name ports by intent/capability, not technology:
  - ✅ `UserRepository`, `NotificationPort`, `PaymentGateway`, `OrderQueryPort`
  - ❌ `MongoUserRepo`, `SendgridService`, `StripeClient`

---

## 9. Shared Infrastructure in Nx Monorepos

- **Feature / Domain Adapters:** Must NOT depend on each other (no cross-domain adapter coupling).
- **Shared Technical Infrastructure (`libs/shared/infrastructure/**`):** Adapters may import shared technical primitives:
  - Database connection clients / pools (e.g., shared `PrismaClient` or `MongooseConnection`)
  - Logging / tracing clients
  - Base repository / ORM helper classes
  - Cryptography / hashing utilities

---

## 10. Naming Conventions

| Concept                  | Convention                         | Example                    |
|--------------------------|------------------------------------|----------------------------|
| Use Case / Command       | `<Verb><Noun>UseCase`              | `CreateOrderUseCase`       |
| Query Handler            | `Get<Noun>Query` / `List<Noun>Query` | `GetOrderSummaryQuery`   |
| Domain Port (repo)       | `<Entity>Repository`               | `OrderRepository`          |
| Application Port         | `<Capability>Port` / `<Capability>Gateway` | `EmailNotificationPort`, `PaymentGateway` |
| Query Port               | `<Entity>QueryPort`                | `OrderQueryPort`           |
| Adapter (implementation) | `<Tech><Port>`                     | `MongooseOrderRepository`, `SendgridEmailAdapter` |
| Input DTO                | `<UseCase>Input` or `<Action>Dto`  | `CreateOrderInput`         |
| Output DTO / View Model  | `<UseCase>Result` or `<Noun>View`  | `OrderSummaryView`         |
| Domain Event             | `<Noun><PastVerb>Event`            | `OrderPlacedEvent`         |
| Value Object             | Noun, no suffix                    | `Money`, `EmailAddress`    |

---

## 11. Common Violations — Flag These

Raise a warning or refactor suggestion when you detect:

- Import of any ORM model (Mongoose, Prisma) outside `infrastructure/` or `adapters/`
- Import of any HTTP client (Axios, Fetch wrappers) inside `application/` or `domain/`
- Business rule conditionals (`if (order.status === 'closed')`) inside a use case or controller instead of aggregate
- A use case returning an aggregate root directly instead of a DTO
- A domain entity with a constructor dependency on a framework service
- Any `new ConcreteRepository()` instantiation inside a use case
- Direct cross-adapter imports between different feature adapters
- Bypassing repository/query ports to perform direct DB queries inside use cases
