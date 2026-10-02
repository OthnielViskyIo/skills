# SOLID Principles — Coding Agent Rules

These rules are based on the SOLID design principles introduced by Robert C. Martin ("Uncle Bob").
The agent must apply these principles when generating, reviewing, or refactoring code.

---

## S — Single Responsibility Principle (SRP)

- Every class, module, or function must have **one and only one reason to change**.
- Do not mix unrelated concerns in the same unit (e.g., business logic + persistence + formatting).
- If a class does more than one job, split it into focused, cohesive units.
- ✅ Good: `OrderProcessor`, `ReceiptPrinter` as separate classes.
- ❌ Bad: One `Order` class that processes, prints, and saves to the database.

---

## O — Open/Closed Principle (OCP)

- Software entities (classes, modules, functions) must be **open for extension, closed for modification**.
- Add new behavior by extending (via inheritance, composition, or interfaces), not by editing existing code.
- Use abstractions (interfaces, abstract classes) to allow new implementations without touching existing ones.
- ✅ Good: Add a new `Shape` subclass to extend area calculation behavior.
- ❌ Bad: Add an `if/else` or `switch` block inside an existing method every time a new case appears.

---

## L — Liskov Substitution Principle (LSP)

- Any instance of a subclass must be **substitutable for its parent class** without breaking program correctness.
- Subclasses must honor the contract of their parent: same preconditions, postconditions, and invariants.
- Do not override methods in a way that weakens behavior or throws unexpected exceptions.
- ✅ Good: A `Square` and `Rectangle` each implement a `Shape` interface independently.
- ❌ Bad: A `Square` extends `Rectangle` and breaks `setWidth`/`setHeight` semantics.

---

## I — Interface Segregation Principle (ISP)

- No client should be forced to depend on methods it does not use.
- Prefer **many small, focused interfaces** over one large general-purpose interface.
- Split fat interfaces into role-specific ones so implementing classes only carry what they need.
- ✅ Good: `Readable`, `Writable`, `Serializable` as separate interfaces.
- ❌ Bad: One `IFileManager` interface with read, write, delete, compress, and encrypt methods all together.

---

## D — Dependency Inversion Principle (DIP)

- High-level modules must **not depend on low-level modules**. Both must depend on abstractions.
- Abstractions must not depend on details. Details must depend on abstractions.
- Use dependency injection to provide concrete implementations from the outside.
- ✅ Good: A `NotificationService` depends on a `IMessageSender` interface; inject `EmailSender` or `SmsSender` at runtime.
- ❌ Bad: A `NotificationService` directly instantiates `new EmailSender()` inside its methods.

---

## General Guidance

- These are **guidelines, not rigid rules** — apply them with judgment based on context and complexity.
- Avoid over-engineering: do not apply SOLID principles where they add unnecessary indirection.
- When in doubt, prefer **clarity and simplicity** over strict adherence.
- SOLID principles are language-agnostic and apply to any object-oriented or modular codebase.
