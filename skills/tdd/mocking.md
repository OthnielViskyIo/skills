# When to Mock

**NestJS backends:** a unit test mocks **every** injected dependency of the unit under test, including your own use cases, services, repositories and Mongoose models. Follow `rules/nestjs.md` §Testing and, in luminus projects, `rules/luminus.md` §3 for how (`useValue` providers, `jest.spyOn`, no `jest.Mocked`). Freeze time with fake timers.

**Frontend code:** follow `rules/react-next.md` §18 Testing (mock at the network boundary).

## Designing for Mockability

At system boundaries, design interfaces that are easy to mock:

**1. Use dependency injection**

Pass external dependencies in rather than creating them internally:

```typescript
// Easy to mock
function processPayment(order, paymentClient) {
    return paymentClient.charge(order.total);
}

// Hard to mock
function processPayment(order) {
    const client = new StripeClient(process.env.STRIPE_KEY);
    return client.charge(order.total);
}
```

**2. Prefer SDK-style interfaces over generic fetchers**

Create specific functions for each external operation instead of one generic function with conditional logic:

```typescript
// GOOD: Each function is independently mockable
const api = {
    getUser: (id) => fetch(`/users/${id}`),
    getOrders: (userId) => fetch(`/users/${userId}/orders`),
    createOrder: (data) => fetch('/orders', { method: 'POST', body: data }),
};

// BAD: Mocking requires conditional logic inside the mock
const api = {
    fetch: (endpoint, options) => fetch(endpoint, options),
};
```

The SDK approach means:
- Each mock returns one specific shape
- No conditional logic in test setup
- Easier to see which endpoints a test exercises
- Type safety per endpoint
