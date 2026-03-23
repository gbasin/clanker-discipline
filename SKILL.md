---
name: clanker-discipline
description: Catches state bloat, grab-bag models, and mutation ambiguity from AI coding agents. Use when reviewing state types, boolean flags, optional-field models, or mutable data patterns.
---

# Clanker Discipline

Apply these rules when writing or reviewing state types, data models, and functions that manage application state. Agents tend to add flags, optional fields, and special cases that compound into state nobody intended — catch that before it lands.

When you find violations, refactor fully. The goal is clean, maintainable code, not minimal diffs. Rip out the flags, reshape the types, restructure the functions. A bigger diff now is better than layering workarounds that compound later.

---

## 1. Derive, don't store

Every boolean you add doubles the theoretical state space. When a value can be derived from data you already have, do not store it. The best source to derive from is an event stream: a log of what happened.

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

Four fields to answer one question, with four mutation sites elsewhere keeping them in sync.

### After: derive from evidence

```ts
function shouldShowFooter(events: SessionEvent[]): boolean {
  const latest = getLatestAssistantMessage(events);
  if (!latest) return false;
  return latest.completed && !latest.error && latest.finish !== 'tool-calls';
}
```

The answer is now computed from events that already exist.

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
    trigger() {
      clearTimeout(timeout!);
      timeout = setTimeout(() => { timeout = null; callback(); }, delayMs);
    },
    clear() {
      if (timeout) { clearTimeout(timeout); timeout = null; }
    },
  };
}
```

Nothing outside the closure can touch the timer.

### The debugging payoff

When state is derived from evidence, debugging becomes data-in, answer-out:

```ts
test('footer is hidden for aborted runs', () => {
  const events = loadEvents('./fixtures/aborted-session.jsonl');
  expect(shouldShowFooter(events)).toBe(false);
});
```

No mocking or timing reproduction. The bug is in the events or in the pure function.

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

## 3. Enforce function contracts

### Never add side effects to a pure function

When a pure function quietly gains a side effect, every callsite inherits behavior it did not ask for. If a function needs side effects, extract them into a separate orchestrator.

- **Semantic functions** are small, pure, and self-describing. All inputs in, all outputs out, no hidden effects.
- **Pragmatic functions** are orchestrators. They compose semantic functions and contain messy domain glue.

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

## 4. Data over procedure

When a long if-chain returns a similar shape from every branch, the logic is a lookup table encoded as code. Convert it to data.

### Before: if-chain

```ts
function getStepInfo(step: string): StepInfo | null {
  if (step === 'verify-email') {
    return { tone: 'action', title: 'Verify your email', detail: 'Check your inbox' };
  }
  if (step === 'add-payment') {
    return { tone: 'action', title: 'Add payment method', detail: 'Enter card details' };
  }
  if (step === 'review-order') {
    return { tone: 'confirm', title: 'Review your order', detail: 'Check totals' };
  }
  // ... 10 more branches
  return null;
}
```

### After: declarative table

```ts
const STEP_INFO: Array<{
  match: (step: string) => boolean;
  info: StepInfo;
}> = [
  { match: (s) => s === 'verify-email', info: { tone: 'action', title: 'Verify your email', detail: 'Check your inbox' } },
  { match: (s) => s === 'add-payment',  info: { tone: 'action', title: 'Add payment method', detail: 'Enter card details' } },
  { match: (s) => s === 'review-order', info: { tone: 'confirm', title: 'Review your order', detail: 'Check totals' } },
  // data, not code
];

function getStepInfo(step: string): StepInfo | null {
  return STEP_INFO.find(({ match }) => match(step))?.info ?? null;
}
```

Easier to scan, extend, and test. An agent adding a new step adds a data entry, not a branch in a control flow.

### When NOT to convert

If branches have different control flow — not just different return values — keep them as code. A table maps inputs to outputs; it cannot express "call X then conditionally call Y."

---

## Checklist

When reviewing code (yours or an agent's):

- [ ] Can any new field be derived from existing state? Derive it.
- [ ] Is mutable state visible beyond its minimal scope? Trap it in a closure.
- [ ] Do any models allow field combinations that should be impossible? Discriminated union.
- [ ] Are there sentinel values (`'none'`, `'unknown'`, `-1`) where `null` would work? Use null.
- [ ] Are there identical type aliases for different domain concepts? Brand or eliminate.
- [ ] Does any function both mutate its input and return it? Pick one contract.
- [ ] Has a semantic function grown side effects? Extract them.
- [ ] Is there an if-chain where every branch returns a similar shape? Make it a table.
- [ ] Are there dead type variants never constructed? Delete them.
