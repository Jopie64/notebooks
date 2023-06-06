---
marp: true
theme: gaia
paginate: true
# backgroundColor: #fff
---

<!--
_class: lead
-->

# Reactive State Management
## With RX Observables

---

## Preparation

* Download notebook

```shell
git clone https://github.com/Jopie64/notebooks.git
```

* Install vscode extension "Node.js Notebooks"
* Open `stateManagement.nnb`
* Don't cheat 😜

---

## Overview

- **Topic:** State Management with RX Observables
- **Objective:** Understand how to manage state using RX observables
- **Aggregation or Reduction:** Process of converting a stream of events into a single state
- **Library:** Redux and its derivatives (ngrx, ngxs) are popular choices in the frontend
- **Our Approach:** Implementing the pattern in RX using the `scan` operator

---


## The Challenge

- Imagine a scenario with a stream of events
- Goal: Show the current state based on the events
- Example: Displaying the current balance from a list (or stream) of transactions

---

## The Solution

- Aggregation or Reduction
- Multiple events are combined to form a single state
- Provides a reactive way to implement the pattern

---

## Reactive state libraries

- Redux: [https://redux.js.org/](https://redux.js.org/)
- ngrx: [https://ngrx.io/](https://ngrx.io/)
- ngxs: [https://www.ngxs.io/](https://www.ngxs.io/)

---

## We use only rxjs

- Alternative approach using RX Observables
- Leveraging the power of the `scan` operator
- Learn how to implement state management using RX

---

<!--
_class: lead
-->

# Reactive State Management
## Array.reduce vs. RX scan

---

## Array.reduce Operator

- The `Array.reduce` operator is a built-in JavaScript method
- Used for aggregating or reducing an array of values to a single value
- Familiar and widely used in JavaScript programming

---

### Example: Addition of a List of Numbers

```javascript
const listOfNumbers = [1,2,3,4,5,6,7,8,9,10];

listOfNumbers
  .reduce((aggregate, current) => aggregate + current, 0);

// 55
```

* Array of numbers from 1 to 10 is reduced to single value: 55
* The reduce method takes a callback function and an initial value (0 in this case)
* The callback function defines how the aggregation is performed

---

### Assignment 1

Make it _multiply_ all numbers together.

```javascript
const listOfNumbers = [1,2,3,4,5,6,7,8,9,10];

listOfNumbers
  .reduce(/* TODO: Implement me */);

// 3628800
```

---

## RX scan Operator

* Works on observable instead of Array

```javascript
const sum = listOfNumbers
  .reduce((aggregate, current) => aggregate + current, 0);
```
vs
```javascript
const sum$ = listOfNumbers$.pipe(
    scan((aggregate, current) => aggregate + current, 0));
```
* Experiment to see one other difference

---

### Assignment 2

Make it _multiply_ all numbers together.

Also experiment with `reduce` instead of `scan`.

```javascript
const aggregation$ = listOfNumbers$.pipe(
    reduce((aggregate, current) => aggregate + current, 0));
```

What is the difference?

---

<!--
_class: lead
-->

# Reactive State Management
## Account balance example

---

## Account balance example

* Using `Array.reduce` for now -> easier to use
* Checkout type definitions -> Any questions?

---

### Assignment 3

Add the missing handlers

```ts
const handleDebitEvent: HandleAccountEvent<DebitEvent> =
  (account, event) => //TODO: Implement me

// One to dispatch all events
const handleAccountEvent: HandleAccountEvent<AccountEvent> =
  (account, event) => //TODO: Implement me
```

Hint: Use `switch` statement in the last one

---

### Assignment 4

Implement `GetAccount` function

```ts
// Type of function that returns an account given a stream of events
type GetAccount = (events: AccountEvent[]) => Account;

const getAccount: GetAccount = events => // TODO: Implement me

getAccount(accountEvents);
// {
//  balance: 80
// }
```
Hint: Use handler declared earlier and `reduce`.

---

### Assignment 4

Why is the balance outcome 80?

```ts
let accountEvents: AccountEvent[] = [
    {
        type: 'CreditEvent',
        amount: 10
    }, {
        type: 'CreditEvent',
        amount: 100
    }, {
        type: 'DebitEvent',
        amount: 30
    }
];
```

---

<!--
_class: lead
-->

# Reactive State Management
## Advanced call example

---

## Advanced call example
```ts
// Idle state means call is gone.
type CallState = 'Accepted' | 'Connected' | 'OnHold' | 'Idle';

type CallParticipant = {
    name: string;
    address: string;
}

type CallInfo = {
    caller: CallParticipant;
    callee: CallParticipant;
}

type Call = {
    id: string;
    info$: Observable<CallInfo>;
    state$: Observable<CallState>;
}
```

---
Our target state: Array of current calls with flattened data.

```ts
type FlattenedCall = {
    id: string;
    info: CallInfo;
    state: CallState;
}

type FlattenedCalls = FlattenedCall[];
```

---

### Advanced call challenge

Function we need to implement:
```ts
type GetCurrentCallList = (incomingCalls$: Observable<Call>) => Observable<FlattenedCall[]>;
```

No events we can reduce to state 😐

How to do that?

---

### Advanced call challenge

Two approaches:
1. Map everything to events first
2. Super secret new approach

---

### Approach 1: Map everything to events first

Steps:
1. Define events
2. Define handlers
3. Convert original model into stream of those events
4. Use `scan` to reduce events to state using handlers

---

#### 1. Define events

```ts
type CallAddedEvent = {
    type: 'CallAddedEvent';
    id: string;
}

type CallRemovedEvent = {
    type: 'CallRemovedEvent';
    id: string;
}
//...
```

---

#### 2. Define handlers

```ts
const handleCallAddedEvent: HandleCallEvent<CallAddedEvent> =
    (state, event) =>
        // Make sure to use immutable methods.
        // So not .push, but .concat to add an element.
        state.concat([{ ...emptyCall, id: event.id }]);

const handleCallRemovedEvent: HandleCallEvent<CallRemovedEvent> =
    (state, event) =>
        state.filter(call => call.id !== event.id);

//...
```

---

#### 3. Convert original model into stream of those events

```ts
const toCallEvent$ = (incomingCall$: Observable<Call>): Observable<CallEvent> =>
  incomingCall$.pipe(
    // for every new call, start monitoring it...
    mergeMap(newCall => merge([
      // Handle call lifetime.
      // Start with CallAddedEvent. Wait until it becomes 'Idle', and then end with CallRemovedEvent
      concat(
        of<CallAddedEvent>({ type: 'CallAddedEvent', id: newCall.id }),
        newCall.state$.pipe(takeWhile(state => state !== 'Idle')),
        of<CallRemovedEvent>({ type: 'CallRemovedEvent', id: newCall.id }),
      ),
      // Monitor state and info to emit CallStateChangeEvent and CallInfoChangeEvent events.
      newCall.state$.pipe(
        map((newState): CallStateChangeEvent => ({ type: 'CallStateChangeEvent', id: newCall.id, newState }))),
      newCall.info$.pipe(
        map((newInfo): CallInfoChangeEvent => ({ type: 'CallInfoChangeEvent', id: newCall.id, newInfo })))
    ])));
```
😱

---
#### 4. Use `scan` to reduce events to state using handlers

#### Assignment 5: Put it all together

```ts

const getCurrentCallList: GetCurrentCallList =
    incomingCall$ =>
            // TODO: Implement me

            // Use toCallEvent$ to convert the incoming call stream
            // into an event stream

            // Handle the events. In other words: reduce the events into current state
            // which is the current list of calls.
```
Hint: use `toCallEvent$`, `handleCallEvent` and `scan`

---

<!--
_class: lead
-->

# The Action Pattern
## Alternative approach

---

### Remember the account example.

```ts
type Account = {
    balance: number;
}

const handleAccountEvent: HandleAccountEvent<AccountEvent> =
  (account, event) => // ...

// Type of function that returns an account given a stream of events
type GetAccount = (events: AccountEvent[]) => Account;

const getAccount: GetAccount = events =>
  events
  .reduce(handleAccountEvent, emptyAccount);

```

---

### Introducing an action:

```ts
type AccountAction = (account: Account) => Account;
```

Converting event stream to action stream:

```ts
// Remember: in our account example we use arrays instead of observables.
type EventsToActions = (events: AccountEvent[]) => AccountAction[];

const toAccountActions: EventsToActions = events =>
    events
    .map((event): AccountAction => account => handleAccountEvent(account, event));
```

---

## Assignment 6

Reimplement `GetAccount` using `toAccountActions`.

```ts
const getAccount: GetAccount = events =>
  events
  .reduce(handleAccountEvent, emptyAccount);
```
Becomes:
```ts
const getAccount2: GetAccount =
    events => toAccountActions(events)
        .reduce((account, action) => /* TODO: implement reducer here */, emptyAccount);
```

---

### Calling example `toCallEvent$`

```ts
const toCallEvent$ = (incomingCall$: Observable<Call>): Observable<CallEvent> =>
  incomingCall$.pipe(
    // for every new call, start monitoring it...
    mergeMap(newCall => merge([
      // Handle call lifetime.
      // Start with CallAddedEvent. Wait until it becomes 'Idle', and then end with CallRemovedEvent
      concat(
        of<CallAddedEvent>({ type: 'CallAddedEvent', id: newCall.id }),
        newCall.state$.pipe(takeWhile(state => state !== 'Idle')),
        of<CallRemovedEvent>({ type: 'CallRemovedEvent', id: newCall.id }),
      ),
      // Monitor state and info to emit CallStateChangeEvent and CallInfoChangeEvent events.
      newCall.state$.pipe(
        map((newState): CallStateChangeEvent => ({ type: 'CallStateChangeEvent', id: newCall.id, newState }))),
      newCall.info$.pipe(
        map((newInfo): CallInfoChangeEvent => ({ type: 'CallInfoChangeEvent', id: newCall.id, newInfo })))
    ])));
```

---
### Calling example `toCallAction$`

```ts
const toCallAction$ = (incomingCall$: Observable<Call>): Observable<CallAction> =>
  incomingCall$.pipe(
    // for every new call, start monitoring it...
    mergeMap(newCall => merge([
      // Handle call lifetime.
      // Start with add call action. Wait until it becomes 'Idle', and then end with call remove action
      concat(
        // Emit an add call action
        of<CallAction>(calls => calls.concat([{ ...emptyCall, id: newCall.id }])),
        newCall.state$.pipe(takeWhile(state => state !== 'Idle')),
        // Emit a call remove action
        of<CallAction>(calls => calls.filter(call => call.id !== newCall.id )),
      ),
      // Monitor state and info to emit Call State Change and Call Info Change actions.
      newCall.state$.pipe(
        map((newState): CallAction => handleSingleCall(newCall.id, call => ({...call, state: newState })))),
      newCall.info$.pipe(
        map((newInfo): CallAction => handleSingleCall(newCall.id, call => ({...call, info: newInfo })))),
    ])));
```

---

### Assignment 7

Implement `GetCurrentCallList` using `toCallAction$`

```ts
const getCurrentCallList2: GetCurrentCallList =
    incomingCall$ =>

            // TODO: Use the toCallAction$ function to convert the incoming call stream
            // into a stream of actions

            // Handle the actions by simply executing them
```

---

<!--
_class: lead
-->

# Recognize the pattern!

---

<!--
_class: lead
-->

# Thanks for your attention
