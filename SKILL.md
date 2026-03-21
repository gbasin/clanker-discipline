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

### Before: flags tracking what happened during a process

```ts
type LoanState = {
  // ...
  tridWaitingPeriodSatisfied: boolean;
  leWaitingPeriodSatisfied: boolean;
  backgroundTasks: {
    title: 'idle' | 'pending' | 'resolved';
    insurance: 'idle' | 'pending' | 'resolved';
  };
  consent: {
    eSign: boolean;
    blanketAuth: boolean;
    biometric: boolean;
  };
};
```

Seven fields, all derivable from the current step and existing results.

### After: derive in one place

```ts
function deriveConsentState(step: LoanStep, toolResults: ToolResults): ConsentState {
  const stepIndex = STEPS.indexOf(step);
  return {
    eSign: stepIndex >= STEPS.indexOf('BankConnect'),
    blanketAuth: stepIndex >= STEPS.indexOf('BankConnect'),
    biometric: stepIndex >= STEPS.indexOf('DocumentSigning')
      || toolResults.identityVerification !== undefined,
  };
}

function deriveBackgroundTaskStatus(step: LoanStep): BackgroundTaskStatus {
  const stepIndex = STEPS.indexOf(step);
  if (stepIndex < STEPS.indexOf('PropertyValuation')) return 'idle';
  if (stepIndex < STEPS.indexOf('ClosingDisclosure')) return 'pending';
  return 'resolved';
}
```

The seven stored fields become two pure functions called from `finalizeSnapshot()`, with no mutation sites to keep in sync.

### When NOT to derive

- The domain genuinely has a state machine with ordered transitions. A loan origination step or a checkout phase is not a cached conclusion; it IS the state.
- A field contains temporal or external data that cannot be rederived: timestamps from async processes, API responses needed downstream.
- The derivation would be more complex than the stored value.

---

## 2. Make wrong states impossible

Every optional field is a question the rest of the codebase must answer every time it touches that data. Discriminated unions let the compiler answer it once.

### Before: optional fields that shouldn't coexist

```ts
type BrokerSyncState = {
  status: 'idle' | 'pending' | 'resolved';
  channel?: 'email' | 'sms';
  contactName?: string;
  requestedAt?: string;
  resolvedAt?: string;
};
```

When `status` is `'idle'`, none of the other fields should exist. When it is `'pending'`, `channel` and `contactName` must exist. The type does not enforce this. Every consumer guesses.

### After: discriminated union

```ts
type BrokerSyncState =
  | { status: 'idle' }
  | { status: 'pending'; channel: 'email' | 'sms'; contactName: string; requestedAt: string }
  | { status: 'resolved'; channel: 'email' | 'sms'; contactName: string; resolvedAt: string };
```

Now you cannot construct an `'idle'` state with a `contactName`. Consumers narrow on the discriminant and the compiler guarantees the fields are there.

### Before: sentinel value in a union

```ts
type PendingUserAction =
  | 'none'
  | 'broker-contact'
  | 'consent'
  | 'bank-connect';

type LoanState = {
  pendingUserAction: PendingUserAction;
};

// every consumer:
if (state.pendingUserAction !== 'none') { ... }
```

`'none'` is not an action. It is the absence of one. The type lies.

### After: nullable

```ts
type PendingUserAction =
  | 'broker-contact'
  | 'consent'
  | 'bank-connect';

type LoanState = {
  pendingUserAction: PendingUserAction | null;
};

// every consumer:
if (state.pendingUserAction) { ... }
```

### Before: dead variant nobody uses

```ts
type PendingInteractiveState = {
  action: PendingInteractiveAction;
  instanceId: string;
  status: 'open' | 'completed'; // 'completed' is never set
};
```

The status goes `{status: 'open'}` to `undefined`. The `'completed'` variant is dead code that suggests a lifecycle that does not exist.

### After: delete the dead variant

```ts
type PendingInteractiveState = {
  action: PendingInteractiveAction;
  instanceId: string;
  status: 'open';
};
```

### Before: grab-bag model

```ts
type BorrowerProfile = {
  firstName?: ProfileField<string>;
  lastName?: ProfileField<string>;
  dob?: ProfileField<string>;
  ssnLast4?: ProfileField<string>;
  email?: ProfileField<string>;
  phone?: ProfileField<string>;
  address?: ProfileField<string>;
  employer?: ProfileField<string>;
  jobTitle?: ProfileField<string>;
  annualIncome?: ProfileField<number>;
  propertyAddress?: ProfileField<string>;
  purchasePrice?: ProfileField<number>;
  // ... 12 more optional fields
};
```

24 optional fields. Every consumer does `profile.firstName?.value ?? DEFAULT_IDENTITY.firstName`. The model does not tell you which fields should exist at which point in the flow.

### After: phased composition

```ts
type IdentityProfile = {
  firstName: ProfileField<string>;
  lastName: ProfileField<string>;
  dob: ProfileField<string>;
  ssnLast4: ProfileField<string>;
};

type ContactProfile = {
  email: ProfileField<string>;
  phone: ProfileField<string>;
  address: ProfileField<string>;
};

type BorrowerProfile = {
  identity?: IdentityProfile;    // populated after credit pull
  contact?: ContactProfile;      // populated after bank connect
  employment?: EmploymentProfile; // populated after income verification
  property?: PropertyProfile;    // populated after broker sync
};
```

Now you check one optional (`profile.identity`) instead of four. When identity exists, all its fields are guaranteed present.

---

## 3. Brand your primitives

Values with identical shapes can represent different domain concepts.

### Before: unbranded twins

```ts
type BackgroundTaskStatus = 'idle' | 'pending' | 'resolved';
type ThirdPartyTaskStatus = 'idle' | 'pending' | 'resolved';
```

These are identical. A function accepting `BackgroundTaskStatus` will happily take a `ThirdPartyTaskStatus` without complaint. If both exist in the same codebase, they should be distinct.

### After: prefixed or branded

If the values are stored/serialized, prefix them:

```ts
type BackgroundTaskStatus = 'bg-idle' | 'bg-pending' | 'bg-resolved';
type ThirdPartyTaskStatus = 'tp-idle' | 'tp-pending' | 'tp-resolved';
```

If the values are internal only, use a branded type:

```ts
type BackgroundTaskStatus = ('idle' | 'pending' | 'resolved') & { readonly __brand: 'background' };
type ThirdPartyTaskStatus = ('idle' | 'pending' | 'resolved') & { readonly __brand: 'thirdParty' };
```

Or better: if you can derive one of them from existing state (see section 1), eliminate it entirely.

---

## 4. Functions: know what you are

### Semantic functions

Small, pure, self-describing. Take all inputs, return all outputs, no hidden effects. The name is the documentation. Should be unit-testable.

```ts
function getAvailableTools(snapshot: SessionSnapshot): ToolName[] { ... }
function shouldTreatAsBorrowerQuestion(text: string): boolean { ... }
function deriveConsentState(step: LoanStep): ConsentState { ... }
```

### Pragmatic functions

Orchestrators that compose semantic functions. Expected to change. Expected to be messy. Doc comments should describe surprising behavior, not restate the name.

```ts
/** Skips live model call when last message is assistant (guided continuation). */
async function streamAssistantTurn({ snapshot, messages, ... }) { ... }
```

### Before: semantic function that became pragmatic

```ts
// started as "apply a tool to a snapshot"
// now it creates fixtures, mutates state, emits proofs, and transitions the state machine
function applyTool(snapshot, toolName, args, occurredAt): ToolMutation {
  switch (toolName) {
    case 'pull_credit': {
      const result = createCreditResult();           // fixture
      snapshot.toolResults.credit = result;           // mutation
      snapshot.borrowerProfile = setProfileField(...); // mutation
      snapshot.borrowerProfile = setProfileField(...); // mutation
      snapshot.borrowerProfile = setProfileField(...); // mutation
      snapshot.loanState.substate = 'awaiting-user-action'; // mutation
      clearInteractiveState(snapshot);                // side effect
      const emitted = buildProofEventsForCredit(...); // proof emission
      snapshot.proofEvents.push(...emitted);          // mutation
      finalizeSnapshot(snapshot);                     // derivation
      return { snapshot, output: result, proofEvents: emitted };
    }
    // ... 16 more cases, same pattern
  }
}
```

276 lines. Each case does three things interleaved: create data, mutate state, emit events.

### After: composed from semantic functions

```ts
function applyCreditPull(snapshot: SessionSnapshot, occurredAt: string): ToolMutation {
  const result = createCreditResult();
  const updatedProfile = applyCreditToProfile(snapshot.borrowerProfile, result);
  const proofEvents = buildProofEventsForCredit(snapshot, result, occurredAt);

  return {
    snapshot: {
      ...snapshot,
      toolResults: { ...snapshot.toolResults, credit: result },
      borrowerProfile: updatedProfile,
      proofEvents: [...snapshot.proofEvents, ...proofEvents],
    },
    output: result,
    proofEvents,
  };
}
```

Each concern is a separate function. The orchestrator just composes them.

---

## 5. Mutation discipline

If a function mutates its input, make that obvious. Never do both.

### Before: mutate and return the same reference

```ts
function withInteractiveState(snapshot: SessionSnapshot, action: string): SessionSnapshot {
  snapshot.loanState.pendingUserAction = action;    // mutates input
  snapshot.loanState.pendingInteractiveState = { ... }; // mutates input
  return snapshot; // returns the same object
}
```

Does the caller use the return value or the original? Both point to the same object. The return type suggests a new value. The implementation says otherwise.

### After: pick one

Option A -- mutate, return void:

```ts
function applyInteractiveState(snapshot: SessionSnapshot, action: string): void {
  snapshot.loanState.pendingUserAction = action;
  snapshot.loanState.pendingInteractiveState = { ... };
}
```

Option B -- clone, return new:

```ts
function withInteractiveState(snapshot: SessionSnapshot, action: string): SessionSnapshot {
  const next = structuredClone(snapshot);
  next.loanState.pendingUserAction = action;
  next.loanState.pendingInteractiveState = { ... };
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

### Before: 191-line if-chain

```ts
function getCurrentStepDescriptor(snapshot: SessionSnapshot): StepDescriptor | null {
  if (!snapshot.borrowerProfile.brokerOrAgentContact) {
    return { tone: 'action', title: 'Share your agent contact', detail: '...' };
  }
  if (!snapshot.loanState.consent.eSign) {
    return { tone: 'action', title: 'Review authorization', detail: '...' };
  }
  if (snapshot.loanState.step === 'BankConnect') {
    return { tone: 'action', title: 'Connect your bank', detail: '...' };
  }
  // ... 15 more branches
}
```

### After: declarative table

```ts
const STEP_DESCRIPTORS: Array<{
  match: (s: SessionSnapshot) => boolean;
  descriptor: StepDescriptor;
}> = [
  {
    match: (s) => !s.borrowerProfile.brokerOrAgentContact,
    descriptor: { tone: 'action', title: 'Share your agent contact', detail: '...' },
  },
  {
    match: (s) => !s.loanState.consent.eSign,
    descriptor: { tone: 'action', title: 'Review authorization', detail: '...' },
  },
  {
    match: (s) => s.loanState.step === 'BankConnect',
    descriptor: { tone: 'action', title: 'Connect your bank', detail: '...' },
  },
  // data, not code
];

function getCurrentStepDescriptor(snapshot: SessionSnapshot): StepDescriptor | null {
  return STEP_DESCRIPTORS.find(({ match }) => match(snapshot))?.descriptor ?? null;
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
