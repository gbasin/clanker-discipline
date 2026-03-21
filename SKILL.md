---
name: clanker-discipline
description: Catch the state bloat, loose models, and mutation confusion that AI coding agents produce
---

# Clanker Discipline

AI coding agents are good at local fixes and bad at respecting the total state surface of an app. Every bug looks like it wants one more flag. One more cached answer. One more special case. That is how a codebase turns into a boolean landfill — fields nobody reads, states nobody intended.

---

## 1. Derive, don't store

Every boolean you add doubles the theoretical state space. When a value can be derived from data you already have, do not store it.

### Before: cached flags

An agent was asked to show a footer only when the assistant finishes naturally. It invented four flags:

```ts
type ThreadState = {
  wasInterrupted: boolean;
  didAssistantFinish: boolean;
  didAssistantError: boolean;
  wasToolCallOnly: boolean;
};

function shouldShowFooter(state: ThreadState): boolean {
  return state.didAssistantFinish
    && !state.wasInterrupted
    && !state.didAssistantError
    && !state.wasToolCallOnly;
}
```

Four fields to answer one question. And somewhere else, four mutation sites keeping them in sync.

### After: derive from evidence

```ts
function shouldShowFooter(events: SessionEvent[]): boolean {
  const latest = getLatestAssistantMessage(events);
  if (!latest) return false;
  return latest.completed && !latest.error && latest.finish !== 'tool-calls';
}
```

The flags disappeared. The answer is computed from the events that already exist.

### When NOT to derive

- The domain genuinely has a state machine with ordered transitions. A checkout step is not a cached conclusion; it IS the state.
- A field contains temporal or external data that cannot be rederived (timestamps from async processes, API responses needed downstream).
- The derivation would be more complex than the stored value.

### If you cannot derive, encapsulate

If mutable state must exist, trap it in the smallest possible scope. A closure is better than a class field:

```ts
// Bad: state visible to the whole class
class Writer {
  private debounceTimeout: ReturnType<typeof setTimeout> | null = null;
  queueSend(text: string) { /* can touch debounceTimeout */ }
  flushNow() { /* can touch debounceTimeout */ }
  somethingElse() { /* can also touch debounceTimeout */ }
}

// Good: state trapped in a closure
function createDebouncedAction(callback: () => void, delayMs = 300) {
  let timeout: ReturnType<typeof setTimeout> | null = null;
  return {
    trigger() { clearTimeout(timeout!); timeout = setTimeout(() => { timeout = null; callback(); }, delayMs); },
    clear() { if (timeout) { clearTimeout(timeout); timeout = null; } },
  };
}
```

Nothing outside the closure can touch the timer. It cannot expand the state space of anything else.

### The debugging payoff

When state is derived from evidence, debugging becomes data-in, answer-out:

```ts
test('footer is hidden for aborted runs', () => {
  const events = loadEvents('./fixtures/aborted-session.jsonl');
  expect(shouldShowFooter(events)).toBe(false);
});
```

No mocking, no timing reproduction. The bug is in the events or in the pure function.

---

## 2. Make wrong states impossible

Every optional field is a question the rest of the codebase must answer every time it touches that data.

### Discriminated unions over optional bags

```ts
// Bad: when status is 'idle', should gateway/transactionId exist? The type doesn't say.
type PaymentState = {
  status: 'idle' | 'processing' | 'settled';
  gateway?: 'stripe' | 'paypal';
  transactionId?: string;
  initiatedAt?: string;
  settledAt?: string;
};

// Good: each status carries exactly the fields it needs.
type PaymentState =
  | { status: 'idle' }
  | { status: 'processing'; gateway: 'stripe' | 'paypal'; transactionId: string; initiatedAt: string }
  | { status: 'settled'; gateway: 'stripe' | 'paypal'; transactionId: string; settledAt: string };
```

### Null over sentinels

```ts
// Bad: 'none' is not an action. It is the absence of one.
type PendingAction = 'none' | 'confirm-address' | 'select-shipping';

// Good
type PendingAction = 'confirm-address' | 'select-shipping';
type OrderState = { pendingAction: PendingAction | null };
```

### Phased composition over grab-bags

```ts
// Bad: 20+ optional fields. Every consumer does profile.firstName ?? defaults.firstName.
type UserProfile = {
  firstName?: string;
  lastName?: string;
  email?: string;
  phone?: string;
  company?: string;
  jobTitle?: string;
  billingAddress?: string;
  cardLast4?: string;
  // ... more
};

// Good: check one optional instead of eight. When identity exists, all its fields are present.
type UserProfile = {
  identity?: { firstName: string; lastName: string; email: string };
  billing?: { address: string; cardLast4: string };
};
```

### Brand identical primitives

```ts
// Bad: a function accepting UserId will happily take a TeamId.
type UserId = string;
type TeamId = string;

// Good
type UserId = string & { readonly __brand: 'user' };
type TeamId = string & { readonly __brand: 'team' };
```

### Delete dead variants

If a type has a variant that is never constructed, delete it. A `status: 'open' | 'completed'` where `'completed'` is never set suggests a lifecycle that does not exist.

---

## 3. Keep functions honest

### Semantic vs. pragmatic

Semantic functions are small, pure, and self-describing. They take all inputs, return all outputs, and have no hidden effects. The name is the documentation.

Pragmatic functions are orchestrators. They compose semantic functions and contain messy domain glue. Doc comments on pragmatic functions should describe surprising behavior, not restate the name.

The break happens when a semantic function silently becomes pragmatic — someone adds a side effect for convenience, and other callsites inherit behavior they did not intend.

### Before: semantic function that grew into a pragmatic one

```ts
function handleWebhook(state, eventType, payload, receivedAt): WebhookResult {
  switch (eventType) {
    case 'payment.captured': {
      const receipt = buildReceipt(payload);            // data creation
      state.order.paymentStatus = 'captured';           // mutation
      state.order.receipt = receipt;                     // mutation
      state.user.lastPurchaseAt = receivedAt;           // mutation
      state.user.lifetimeSpend += receipt.amount;        // mutation
      clearPendingAction(state);                         // side effect
      const notifications = buildPaymentNotifs(state);   // notification
      state.notifications.push(...notifications);        // mutation
      recalculateDashboard(state);                       // derivation
      return { state, output: receipt, notifications };
    }
    // ... 12 more cases, same pattern
  }
}
```

### After: composed from semantic functions

```ts
function handlePaymentCaptured(state: AppState, payload: PaymentPayload, receivedAt: string): WebhookResult {
  const receipt = buildReceipt(payload);
  const updatedOrder = applyPaymentToOrder(state.order, receipt);
  const updatedUser = applyPurchaseToUser(state.user, receipt, receivedAt);
  const notifications = buildPaymentNotifs(state, receipt);

  return {
    state: { ...state, order: updatedOrder, user: updatedUser },
    output: receipt,
    notifications,
  };
}
```

### Pick a mutation contract

If a function mutates its input, return `void`. If it returns a value, clone first. Never mutate the input and return the same reference — callers cannot tell whether to use the return value or the original.

```ts
// Bad: mutates AND returns the same object
function withPendingAction(state: AppState, action: string): AppState {
  state.pendingAction = action;
  return state;
}

// Good: mutate, return void
function applyPendingAction(state: AppState, action: string): void {
  state.pendingAction = action;
}

// Also good: clone, return new
function withPendingAction(state: AppState, action: string): AppState {
  return { ...state, pendingAction: action };
}
```

---

## Checklist

When reviewing code (yours or an agent's):

- [ ] Can any new field be derived from existing state? Derive it.
- [ ] Do any models allow field combinations that should be impossible? Discriminated union.
- [ ] Are there sentinel values (`'none'`, `'unknown'`, `-1`) where `null` would work? Use null.
- [ ] Are there identical type aliases for different domain concepts? Brand or eliminate.
- [ ] Does any function both mutate its input and return it? Pick one contract.
- [ ] Has a semantic function grown side effects? Extract them.
- [ ] Are there dead type variants never constructed? Delete them.
