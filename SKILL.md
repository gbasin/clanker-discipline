---
name: clanker-discipline
description: Catch the state bloat, loose models, and mutation confusion that AI coding agents produce
---

# Clanker Discipline

AI coding agents are good at local fixes and bad at respecting the total state surface of an app. Every bug looks like it wants one more flag. One more cached answer. One more special case. That is how a codebase turns into a boolean landfill — fields nobody reads, states nobody intended.

This skill teaches you to catch and fix the patterns agents overproduce.

---

## 1. Store evidence, not conclusions

Every boolean you add doubles the theoretical state space. Three booleans is eight states. Five is thirty-two. Most of those combinations are impossible in practice but nothing in the code says so.

When a value can be derived from data you already have, do not store it.

### Before: cached flags

The agent was asked to show a footer only when the assistant finishes naturally. It invented four flags:

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

### After: derive from events

```ts
function shouldShowFooter(events: SessionEvent[]): boolean {
  const latest = getLatestAssistantMessage(events);
  if (!latest) return false;
  return latest.completed && !latest.error && latest.finish !== 'tool-calls';
}

function getLatestAssistantMessage(events: SessionEvent[]) {
  for (let i = events.length - 1; i >= 0; i--) {
    if (events[i].type === 'message.updated' && events[i].role === 'assistant') {
      return events[i];
    }
  }
  return undefined;
}
```

The flags disappeared. The answer is computed from the events that already exist.

### Before: flags tracking what happened during a pipeline

```ts
type OrderState = {
  // ...
  paymentCaptured: boolean;
  inventoryReserved: boolean;
  shippingLabelCreated: boolean;
  backgroundChecks: {
    fraud: 'idle' | 'pending' | 'resolved';
    addressVerification: 'idle' | 'pending' | 'resolved';
  };
  consent: {
    termsAccepted: boolean;
    marketingOptIn: boolean;
    ageVerified: boolean;
  };
};
```

Eight fields, all derivable from the current step and existing results.

### After: derive in one place

```ts
function deriveConsentState(step: OrderStep, results: StepResults): ConsentState {
  const stepIndex = STEPS.indexOf(step);
  return {
    termsAccepted: stepIndex >= STEPS.indexOf('Payment'),
    marketingOptIn: stepIndex >= STEPS.indexOf('Payment'),
    ageVerified: stepIndex >= STEPS.indexOf('Fulfillment')
      || results.identityCheck !== undefined,
  };
}

function deriveBackgroundCheckStatus(step: OrderStep): CheckStatus {
  const stepIndex = STEPS.indexOf(step);
  if (stepIndex < STEPS.indexOf('Payment')) return 'idle';
  if (stepIndex < STEPS.indexOf('Shipping')) return 'pending';
  return 'resolved';
}
```

The eight stored fields become two pure functions called from `finalizeOrder()`, with no mutation sites to keep in sync.

### When NOT to derive

- The domain genuinely has a state machine with ordered transitions. A checkout step or a deployment phase is not a cached conclusion; it IS the state.
- A field contains temporal or external data that cannot be rederived: timestamps from async processes, API responses needed downstream.
- The derivation would be more complex than the stored value.

---

## 2. Make wrong states impossible

Every optional field is a question the rest of the codebase must answer every time it touches that data. Discriminated unions let the compiler answer it once.

### Before: optional fields that shouldn't coexist

```ts
type PaymentState = {
  status: 'idle' | 'processing' | 'settled';
  gateway?: 'stripe' | 'paypal';
  transactionId?: string;
  initiatedAt?: string;
  settledAt?: string;
};
```

When `status` is `'idle'`, none of the other fields should exist. When it is `'processing'`, `gateway` and `transactionId` must exist. The type does not enforce this. Every consumer guesses.

### After: discriminated union

```ts
type PaymentState =
  | { status: 'idle' }
  | { status: 'processing'; gateway: 'stripe' | 'paypal'; transactionId: string; initiatedAt: string }
  | { status: 'settled'; gateway: 'stripe' | 'paypal'; transactionId: string; settledAt: string };
```

Now you cannot construct an `'idle'` state with a `transactionId`. Consumers narrow on the discriminant and the compiler guarantees the fields are there.

### Before: sentinel value in a union

```ts
type PendingAction =
  | 'none'
  | 'confirm-address'
  | 'select-shipping'
  | 'review-order';

type OrderState = {
  pendingAction: PendingAction;
};

// every consumer:
if (state.pendingAction !== 'none') { ... }
```

`'none'` is not an action. It is the absence of one. The type lies.

### After: nullable

```ts
type PendingAction =
  | 'confirm-address'
  | 'select-shipping'
  | 'review-order';

type OrderState = {
  pendingAction: PendingAction | null;
};

// every consumer:
if (state.pendingAction) { ... }
```

### Before: dead variant nobody uses

```ts
type ModalState = {
  view: ModalView;
  instanceId: string;
  status: 'open' | 'completed'; // 'completed' is never set
};
```

The status goes `{status: 'open'}` to `undefined`. The `'completed'` variant is dead code that suggests a lifecycle that does not exist.

### After: delete the dead variant

```ts
type ModalState = {
  view: ModalView;
  instanceId: string;
  status: 'open';
};
```

### Before: grab-bag model

```ts
type UserProfile = {
  firstName?: string;
  lastName?: string;
  dob?: string;
  avatarUrl?: string;
  email?: string;
  phone?: string;
  address?: string;
  company?: string;
  jobTitle?: string;
  timezone?: string;
  preferredLanguage?: string;
  billingAddress?: string;
  cardLast4?: string;
  // ... 10 more optional fields
};
```

20+ optional fields. Every consumer does `profile.firstName ?? defaults.firstName`. The model does not tell you which fields should exist at which point in onboarding.

### After: phased composition

```ts
type IdentityInfo = {
  firstName: string;
  lastName: string;
  dob: string;
  avatarUrl: string;
};

type ContactInfo = {
  email: string;
  phone: string;
  address: string;
};

type UserProfile = {
  identity?: IdentityInfo;   // populated after signup
  contact?: ContactInfo;     // populated after email verification
  employment?: WorkInfo;     // populated after profile completion
  billing?: BillingInfo;     // populated after first purchase
};
```

Now you check one optional (`profile.identity`) instead of four. When identity exists, all its fields are guaranteed present.

---

## 3. Brand your primitives

Values with identical shapes can represent different domain concepts.

### Before: unbranded twins

```ts
type BuildStatus = 'idle' | 'pending' | 'resolved';
type DeployStatus = 'idle' | 'pending' | 'resolved';
```

These are identical. A function accepting `BuildStatus` will happily take a `DeployStatus` without complaint. If both exist in the same codebase, they should be distinct.

### After: prefixed or branded

If the values are stored/serialized, prefix them:

```ts
type BuildStatus = 'build-idle' | 'build-pending' | 'build-resolved';
type DeployStatus = 'deploy-idle' | 'deploy-pending' | 'deploy-resolved';
```

If the values are internal only, use a branded type:

```ts
type BuildStatus = ('idle' | 'pending' | 'resolved') & { readonly __brand: 'build' };
type DeployStatus = ('idle' | 'pending' | 'resolved') & { readonly __brand: 'deploy' };
```

Or better: if you can derive one of them from existing state (see section 1), eliminate it entirely.

---

## 4. Functions: know what you are

### Semantic functions

Small, pure, self-describing. Take all inputs, return all outputs, no hidden effects. The name is the documentation. Should be unit-testable.

```ts
function getAvailableActions(order: OrderSnapshot): ActionName[] { ... }
function isReturnEligible(item: LineItem, now: Date): boolean { ... }
function deriveShippingOptions(address: Address, weight: number): ShippingRate[] { ... }
```

### Pragmatic functions

Orchestrators that compose semantic functions. Expected to change. Expected to be messy. Doc comments should describe surprising behavior, not restate the name.

```ts
/** Retries with the backup gateway if the primary returns a soft decline. */
async function processPayment({ order, gateway, ... }) { ... }
```

### Before: semantic function that became pragmatic

```ts
// started as "handle a webhook event"
// now it validates payloads, mutates state, sends notifications, and updates analytics
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

250+ lines. Each case does three things interleaved: create data, mutate state, produce side effects.

### After: composed from semantic functions

```ts
function handlePaymentCaptured(state: AppState, payload: PaymentPayload, receivedAt: string): WebhookResult {
  const receipt = buildReceipt(payload);
  const updatedUser = applyPurchaseToUser(state.user, receipt, receivedAt);
  const notifications = buildPaymentNotifs(state, receipt);

  return {
    state: {
      ...state,
      order: { ...state.order, paymentStatus: 'captured', receipt },
      user: updatedUser,
    },
    output: receipt,
    notifications,
  };
}
```

Each concern is a separate function. The orchestrator just composes them.

---

## 5. Mutation discipline

If a function mutates its input, make that obvious. Never do both.

### Before: mutate and return the same reference

```ts
function withPendingAction(state: AppState, action: string): AppState {
  state.pendingAction = action;          // mutates input
  state.actionStartedAt = Date.now();    // mutates input
  return state; // returns the same object
}
```

Does the caller use the return value or the original? Both point to the same object. The return type suggests a new value. The implementation says otherwise.

### After: pick one

Option A — mutate, return void:

```ts
function applyPendingAction(state: AppState, action: string): void {
  state.pendingAction = action;
  state.actionStartedAt = Date.now();
}
```

Option B — clone, return new:

```ts
function withPendingAction(state: AppState, action: string): AppState {
  const next = structuredClone(state);
  next.pendingAction = action;
  next.actionStartedAt = Date.now();
  return next;
}
```

---

## 6. Encapsulate what you cannot eliminate

If you must have mutable state, trap it in the smallest possible scope.

### Before: state visible to the whole class

```ts
class MessageWriter {
  private debounceTimeout: ReturnType<typeof setTimeout> | null = null;

  queueSend(text: string): void {
    if (this.debounceTimeout) clearTimeout(this.debounceTimeout);
    this.debounceTimeout = setTimeout(() => this.write(text), 300);
  }

  flushNow(): void {
    if (!this.debounceTimeout) return;
    clearTimeout(this.debounceTimeout);
    this.debounceTimeout = null;
  }

  private write(text: string): void { console.log(text); }
}
```

Every method in the class can see and touch `debounceTimeout`. Next time an agent adds a feature, it might read or write this field from somewhere unexpected.

### After: state trapped in a closure

```ts
function createDebouncedAction(callback: () => void, delayMs = 300) {
  let timeout: ReturnType<typeof setTimeout> | null = null;

  function clear() {
    if (!timeout) return;
    clearTimeout(timeout);
    timeout = null;
  }

  function trigger() {
    clear();
    timeout = setTimeout(() => { timeout = null; callback(); }, delayMs);
  }

  return { trigger, clear };
}
```

The timer still exists but nothing outside the closure can touch it. It cannot expand the state space of anything else.

---

## 7. Data-driven over procedural

When you have a long chain of if-statements that each return a similar shape, the logic is a lookup table encoded as code. Convert it to data.

### Before: long if-chain

```ts
function getStepDescriptor(state: WizardState): StepDescriptor | null {
  if (!state.profile.email) {
    return { tone: 'action', title: 'Enter your email', detail: '...' };
  }
  if (!state.emailVerified) {
    return { tone: 'waiting', title: 'Check your inbox', detail: '...' };
  }
  if (state.step === 'SetPassword') {
    return { tone: 'action', title: 'Create a password', detail: '...' };
  }
  // ... 15 more branches
}
```

### After: declarative table

```ts
const STEP_DESCRIPTORS: Array<{
  match: (s: WizardState) => boolean;
  descriptor: StepDescriptor;
}> = [
  {
    match: (s) => !s.profile.email,
    descriptor: { tone: 'action', title: 'Enter your email', detail: '...' },
  },
  {
    match: (s) => !s.emailVerified,
    descriptor: { tone: 'waiting', title: 'Check your inbox', detail: '...' },
  },
  {
    match: (s) => s.step === 'SetPassword',
    descriptor: { tone: 'action', title: 'Create a password', detail: '...' },
  },
  // data, not code
];

function getStepDescriptor(state: WizardState): StepDescriptor | null {
  return STEP_DESCRIPTORS.find(({ match }) => match(state))?.descriptor ?? null;
}
```

Easier to scan, extend, reorder, and test. An agent adding a new step adds a data entry, not a branch in a control flow.

---

## 8. Debugging with event streams

When state is derived from evidence, debugging becomes data-in, answer-out:

```bash
# export the event stream from a broken session
app session export-events-jsonl --session ses_123 --out ./tmp/session.jsonl
```

```ts
function loadEvents(file: string): SessionEvent[] {
  return fs.readFileSync(file, 'utf8')
    .split('\n')
    .filter(Boolean)
    .map((line) => JSON.parse(line));
}

test('footer is hidden for aborted runs', () => {
  const events = loadEvents('./tmp/session.jsonl');
  expect(shouldShowFooter(events)).toBe(false);
});
```

No mocking, no timing reproduction. The artifact is data. The bug is in the events or the pure function.

---

## Checklist

When reviewing code (yours or an agent's), check:

- [ ] Can any new field be derived from existing state? If yes, derive it.
- [ ] Do any models allow field combinations that should be impossible? Use discriminated unions.
- [ ] Are there sentinel values (`'none'`, `'unknown'`, `-1`) where `null` would be clearer?
- [ ] Are there identical type aliases for different domain concepts? Brand them or eliminate one.
- [ ] Does any function both mutate its input and return it? Pick one.
- [ ] Has a semantic function grown side effects it did not start with? Extract them.
- [ ] Is there a long if-chain where each branch returns a similar shape? Make it a table.
- [ ] Are there dead type variants that are never constructed? Delete them.
