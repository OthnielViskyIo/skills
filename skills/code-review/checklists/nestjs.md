# Code Review Checklist – NestJS

- [ ] Controllers, services, and modules are organized and named consistently.
- [ ] DTOs (Data Transfer Objects) are used for all input validation and type safety.
- [ ] Validation pipes properly configured (never trust raw input).
- [ ] Error handling uses global exception filters or interceptors.
- [ ] Dependency injection is used correctly for all providers/services.
- [ ] No business logic in controllers—keep logic in services/providers.
- [ ] Guards and interceptors are used for authentication/authorization and cross-concerns.
- [ ] All async code is awaited/handled—no unhandled promise rejections.
- [ ] Database queries are parameterized to safeguard against injection.
- [ ] Logging (with Nest logger or Winston) is in place for errors and key operations.
- [ ] Circular dependencies are avoided.
- [ ] Swagger/OpenAPI is kept up to date if API docs are part of the project.
- [ ] Rate limiting is applied to public/unauthenticated endpoints.
