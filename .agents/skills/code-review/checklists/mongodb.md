# Code Review Checklist – MongoDB

- [ ] All queries are parameterized and user input is sanitized.
- [ ] Collections, indexes, and schemas use proper naming conventions.
- [ ] Use of indexes is justified (performance checked, no over-indexing).
- [ ] Deletes and updates use criteria—never unscoped (risk of data loss!).
- [ ] Aggregations are optimized, avoid unnecessary stages or large in-memory sorts.
- [ ] No blocking or long-running queries (review explain plans).
- [ ] Sensitive data (passwords, tokens) is never stored in plain text.
- [ ] When using Mongoose, all schemas are validated and all fields are typed.
- [ ] Proper error handling for all database operations.
- [ ] Transactions are used where consistency is required.
- [ ] Code handles potential null/undefined returns on queries.
- [ ] No business logic embedded in schemas/models unless strictly necessary.
- [ ] Connection pool size is configured appropriately for the expected load.
