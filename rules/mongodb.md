---
paths:
  - "**/*.{schema,model,repository}.ts"
  - "**/infrastructure/**"
---

# MongoDB Rules

These rules govern all MongoDB code generated in this project, whether it uses the official Node.js driver or Mongoose. Follow them precisely and without exception unless the user explicitly overrides a specific rule.

`rules/typescript.md` applies in full to all database code. `rules/clean-architecture.md` decides **where** database code lives: schemas, models, and repository implementations belong in `infrastructure/`, behind repository ports defined in the domain. This document covers **how** that code is written.

---

## 1. Schema Design Follows Access Patterns

- Design documents around how the application reads and writes them, not around normalised tables. Start from the queries, then shape the documents.
- **Embed** data that is read together, owned by the parent, and bounded in size (e.g. an order's line items, a user's addresses).
- **Reference** data that is shared between documents, updated independently, or grows without limit (e.g. a product referenced by many orders, comments on a popular post).
- Never let an array grow without bound. A document has a hard 16 MB limit, and large arrays make every read and update of the document slower. Move unbounded child data into its own collection.

```typescript
// ❌ unbounded — every new comment grows the post document forever
{ _id, title, body, comments: [/* thousands */] }

// ✅ comments in their own collection, indexed by postId
// posts:    { _id, title, body, commentCount }
// comments: { _id, postId, authorId, body, createdAt }
```

- Duplicate a small, rarely-changing field (e.g. an author's display name) when it avoids a join on a hot read path. Document who is responsible for keeping the copy in sync.
- Keep field names consistent across collections (`createdAt`, not a mix of `created_at` and `creationDate`). Use camelCase.

---

## 2. Data Types

- Use `ObjectId` for `_id` and for references to other documents. Never store a reference as a string in one collection and an `ObjectId` in another — queries silently match nothing.
- Store dates as BSON `Date`, never as strings or numbers. Store them in UTC.
- Never store money as a floating-point `number`. Use `Decimal128`, or integers in the smallest unit (cents).
- Validate and convert external ids at the boundary (e.g. `ObjectId.isValid` + `new ObjectId(id)`) before they reach a query. An invalid id is a 400, not a 500.

---

## 3. Schema Validation

- Every collection must have an enforced schema: a Mongoose schema, or a `$jsonSchema` validator on the collection when using the native driver.
- Mark required fields as required and constrain values (`enum`, `min`/`max`, `match`) in the schema, not only in application code.
- Keep Mongoose schemas strict (the default). Never set `strict: false` to "make a field save".
- Mongoose does **not** run validators on update queries by default. Pass `runValidators: true` to `updateOne`, `updateMany`, and `findOneAndUpdate`, or set it globally.
- Enable `timestamps: true` on Mongoose schemas instead of setting `createdAt`/`updatedAt` by hand.
- Type every model. Define a TypeScript interface for the document and pass it to `Schema<T>` / `model<T>()` or `Collection<T>`; never work with untyped documents.

```typescript
interface OrderDocument {
    customerId: Types.ObjectId;
    status: "pending" | "paid" | "shipped";
    totalCents: number;
}

const orderSchema = new Schema<OrderDocument>(
    {
        customerId: { type: Schema.Types.ObjectId, ref: "Customer", required: true, index: true },
        status: { type: String, enum: ["pending", "paid", "shipped"], required: true },
        totalCents: { type: Number, required: true, min: 0 },
    },
    { timestamps: true },
);
```

---

## 4. Indexes

- Every query on a hot path must be supported by an index. Check with `explain("executionStats")`: a `COLLSCAN`, or `totalDocsExamined` far above `nReturned`, means the index is missing or wrong.
- Order compound index fields by the **ESR rule**: **E**quality fields first, then **S**ort fields, then **R**ange fields.

```typescript
// query: { status: "paid", createdAt: { $gte: since } }, sort: { totalCents: -1 }
orderSchema.index({ status: 1, totalCents: -1, createdAt: 1 }); // E, S, R
```

- A compound index on `{ a, b }` also serves queries on `a` alone. Do not add a separate `{ a }` index.
- Do not over-index. Every index slows down writes and uses memory. Remove indexes that no query uses (`$indexStats`).
- Enforce uniqueness with a `unique` index, never with a "find, then insert if missing" check — that check is a race condition.
- Use TTL indexes to expire temporary data (sessions, tokens, logs) instead of cleanup jobs.
- Disable Mongoose `autoIndex` in production. Build indexes through migrations or a deploy step so a large index build does not block startup.

---

## 5. Queries

- Return only the fields you need with a projection. Never fetch whole documents to read one field.
- Use `.lean()` in Mongoose for read-only queries. It returns plain objects and skips document hydration.
- Always bound result sets: every list query has a `limit`.
- Paginate large collections with range queries on an indexed field (`{ _id: { $gt: lastId } }`), not with `skip`. `skip` still walks every skipped document, so deep pages get slower and slower.
- Use `countDocuments(filter)` for exact filtered counts and `estimatedDocumentCount()` for a fast total. Do not count by loading documents.
- Avoid N+1 queries. Load related documents in one query with `$in`, or with `$lookup` / `populate` when a join is truly needed — not in a loop.

```typescript
// ❌ one query per order
for (const order of orders) {
    order.customer = await Customer.findById(order.customerId);
}

// ✅ one query for all customers
const customerIds = orders.map((order) => order.customerId);
const customers = await Customer.find({ _id: { $in: customerIds } }).lean();
```

- Handle `null` results. `findOne` and `findById` return `null` when nothing matches; never assume a document exists.

---

## 6. Writes

- Update with atomic operators (`$set`, `$inc`, `$push`, `$pull`, `$addToSet`). Never read a document, change it in memory, and write the whole thing back — concurrent writes will overwrite each other.

```typescript
// ❌ lost update under concurrency
const product = await Product.findById(id);
product.stock -= quantity;
await product.save();

// ✅ atomic and guarded
const result = await Product.updateOne(
    { _id: id, stock: { $gte: quantity } },
    { $inc: { stock: -quantity } },
);
if (result.modifiedCount === 0) throw new OutOfStockError(id);
```

- Every `updateMany` and `deleteMany` must have a specific filter. Never run either with an empty filter (`{}`) outside a migration or test.
- Check the write result (`matchedCount`, `modifiedCount`, `deletedCount`) when the operation is expected to affect a document.
- Use `upsert: true` only together with a unique index on the filter fields, or concurrent upserts can create duplicates.
- Use `bulkWrite` / `insertMany` for batches instead of one round trip per document.
- Use `returnDocument: "after"` (driver) or `new: true` (Mongoose) on `findOneAndUpdate` when the caller needs the updated document.

---

## 7. Transactions

- Use a transaction when several writes must succeed or fail together, especially across collections. Transactions need a replica set or sharded cluster.
- Prefer a schema where the operation touches a single document — single-document writes are already atomic and need no transaction.
- Use `session.withTransaction()` (or Mongoose `connection.transaction()`), which retries transient errors. Pass the `session` to **every** operation inside it.
- Keep transactions short. Never call external services (HTTP, email, queues) inside a transaction.

```typescript
await session.withTransaction(async () => {
    await Account.updateOne({ _id: fromId }, { $inc: { balanceCents: -amount } }, { session });
    await Account.updateOne({ _id: toId }, { $inc: { balanceCents: amount } }, { session });
});
```

---

## 8. Aggregation

- Put `$match` and `$project` as early in the pipeline as possible so later stages process fewer, smaller documents. A `$match` at the start of the pipeline can use an index.
- `$sort` before `$limit` so the server can do a top-k sort instead of sorting everything.
- Index the `foreignField` of every `$lookup`.
- A pipeline stage is limited to 100 MB of memory. Restructure the pipeline before reaching for `allowDiskUse`.
- Type pipeline results. Pass the result type to `aggregate<T>()` instead of working with `any`.

---

## 9. Security

- Never build a filter from raw user input. A body such as `{ "password": { "$ne": null } }` turns an equality check into an operator injection. Validate input with a DTO/schema so fields are primitives, and enable Mongoose `sanitizeFilter` (or cast values explicitly).

```typescript
// ❌ req.body.email could be { $ne: null }
await User.findOne({ email: req.body.email });

// ✅ validated primitive
const { email } = loginSchema.parse(req.body); // email: string
await User.findOne({ email });
```

- Never use `$where`, `$function`, or `$accumulator` with user input — they execute JavaScript on the server.
- Hash passwords (bcrypt/argon2) and never store tokens, API keys, or secrets in plain text.
- Exclude sensitive fields from query results by default (`select: false` in Mongoose), and select them explicitly only where needed.
- Read the connection string from configuration, never hard-code it. Connect with TLS and a database user that has only the roles the service needs.

---

## 10. Connections

- Create **one** `MongoClient` / Mongoose connection per process at startup and reuse it. Never connect per request.
- In serverless or hot-reloading environments (e.g. Next.js dev), cache the client on a module-level or global variable so it is not recreated on every invocation.
- Configure `maxPoolSize` and timeouts (`serverSelectionTimeoutMS`, `socketTimeoutMS`) explicitly for the expected load.
- Close the connection on graceful shutdown.
- Keep retryable writes enabled (the default). Use write concern `majority` for data that must not be lost.

---

## 11. Error Handling

- Translate database errors into domain or application errors in the repository. Never leak driver errors (or their messages) to API responses.
- Handle duplicate key errors (code `11000`) explicitly and map them to a conflict error.
- Distinguish "not found" from failure: a missing document is a normal result, a network error is not.

```typescript
try {
    await this.users.insertOne(user);
} catch (error) {
    if (error instanceof MongoServerError && error.code === 11000) {
        throw new EmailAlreadyTakenError(user.email);
    }
    throw error;
}
```

---

## 12. Migrations

- Change schemas and indexes through versioned migration scripts (e.g. `migrate-mongo`), never by hand in production.
- Make migrations idempotent and safe to re-run. Process large collections in batches.
- Write code that tolerates both the old and new document shape while a migration is rolling out.

---

## 13. Testing

- Test repositories against a real MongoDB (`mongodb-memory-server` or a Testcontainers instance), not a mocked driver. Mocks cannot catch wrong filters, missing indexes, or validation errors.
- Isolate tests: clear collections or use a fresh database per test file.
- Assert on the stored documents, not only on the return value.
- Follow `/tdd` for non-trivial repository logic.

---

## 14. Things to Never Do

- Never run `updateMany` or `deleteMany` with an empty filter in application code.
- Never pass unvalidated request data into a query filter.
- Never read-modify-write a document when an atomic update operator can do it.
- Never store an unbounded array in a document.
- Never paginate deep result sets with `skip`.
- Never open a new connection per request.
- Never store money as a float, dates as strings, or references as strings.
- Never disable schema strictness or validation to get a write through.
- Never import Mongoose or the MongoDB driver outside `infrastructure/`.
