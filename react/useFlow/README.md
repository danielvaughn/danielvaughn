# useFlow React Hook

A super flexible wrapper around `useState`, built for maximum ergonomics.

`useFlow` addresses the following challenge - decouple the UI from busines logic (a) without creating additional complexity, and (b) without tons of boilerplate.

`useFlow` introduces a standardized way to manage both small and large stateful flows.

## Example

While `useFlow` offers a wide variety of implementation strategies, below is the package in its simplest form.
At first glance, it might not seem special, but 2 things make this unique:

1. You don't need to worry about ensuring your state object is a new instance. `useFlow` uses `immer` under the hood to handle all of that for you.
2. You don't even need to specify the entire object - only the portion of the state that you wish to update. `useFlow` performs a merge under the hood.

```javascript
const SimpleDemo = () => {
  const [state, setState] = useFlow({
    firstName: "",
    lastName: "",
  });

  // You are automatically given a "set" method for updating your state,
  // but you can define as many custom actions as you need
  return (
    <form>
      <input
        type="text"
        value={state.firstName}
        onChange={(e) => setState({ firstName: e.target.value })}
      />
      <input
        type="text"
        value={state.lastName}
        onChange={(e) => setState({ lastName: e.target.value })}
      />
    </form>
  );
};
```

## Design Principles

The `useFlow` hook aims to satisfy the following requirements:

1. ##### Flexible
   It should be possible to use as a standalone hook, or in conjunction with other hooks, or alongside `useContext`.
2. ##### Ergonomic
   It should requires minimal boilerplate code - as few keystrokes as possible.
3. ##### Async
   It should allow for a single asynchronous function to update state multiple times in the same call.
4. ##### Encapsulated
   It should allow all business logic, including network calls and side effects, to be defined in the same location.

## Usage

You can use a variety of implementation strategies depending on your needs; here are some of the most common.

### 1. Directly in-component

This method is the simplest, but also couples the business logic to the component's UI.

```javascript
import React from "react";
import { useFlow } from "danielvaughn/useFlow";

const [state, setState] = useFlow({ count: 0 });

const Counter = () => {
  return (
    <div>
      <p>The count is {state.count}</p>
      <button
        onClick={() => {
          setState({ count: state.count + 1 });
        }}
      >
        Add
      </button>
    </div>
  );
};
```

### 2. As a standalone custom hook

This method is a bit more complex, but has the advantage of segregating the business logic to a separate file.
The basic idea is to attach a component-specific custom hook that implements all the business logic for the given component - in this case, a simple counter.

```javascript
  // counter/useCounter.js
  import { useEffect } from 'react'
  import { useFlow } from 'danielvaughn/useFlow'
  import { useHistory } from 'react-router-dom'

  const useCounter = () => {
    const history = useHistory()
    const [state, actions, context] = useFlow({ count: 0, }, counterActions, context: { history })

    useEffect(() => { actions.checkCount(state.count) }, [state.count])

    return [state, actions, context]
  }

  // In useFlow, your "actions" object is a higher order function that passes in all the contextual data you need.
  const counterActions = ({ get, set, context }) => ({
    increment: (amount = 1) => {
      const { count } = get()
      set({ count: count + amount })
    },
    decrement: (amount = 1) => {
      const { count } = get()
      set({ count: Math.max(count - amount, 0) })
    },
    checkCount: async (count) => {
      const { history } = context()

      if (count > 10) {
        history.push('/done')
      }
    }
  })
```

```javascript
// counter/index.jsx
import React from "react";

// Notice how there is no logic here - just UI and event handling
const Counter = () => {
  const [state, actions] = useCounter();

  return (
    <div>
      <p>
        The current count is {state.count}. Once you click the button ten times,
        we'll navigate away from this page.
      </p>
      <button onClick={actions.increment}>Add</button>
      <button onClick={actions.decrement}>Subtract</button>
    </div>
  );
};
```

### 3. As a HOC

This method requires the most boilerplate, but is also the most decoupled.
By splitting a component into 2 pieces - one for business logic and another for UI - you obtain maximum flexibility.
Even complex, page-level components can easily be dropped into Storybook, or unit-tested, with this approach.

```javascript
// sign-up-form/Client.jsx
import React, { useEffect } from "react";
import { useFlow } from "danielvaughn/useFlow";
import Interface from "./Interface.jsx";

const Client = () => {
  const [state, actions, context] = useFlow(
    {
      firstName: "",
      lastName: "",
      age: 18,
      canVote: true,
    },
    clientActions
  );

  return <Interface {...state} {...actions} {...context} />;
};

// In this example, a custom action is used to manage a side effect
const clientActions = ({ set }) => ({
  setAge: (age) => {
    set({
      age,
      canVote: age >= 18,
    });
  },
});
```

```javascript
// sign-up-form/Interface.jsx
import React, { useEffect } from "react";

const Interface = ({ firstName, lastName, age, canVote, set, setAge }) => {
  return (
    <form>
      <input
        type="text"
        value={firstName}
        onChange={(e) => set({ firstName: e.target.value })}
      />
      <input
        type="text"
        value={lastName}
        onChange={(e) => set({ lastName: e.target.value })}
      />

      <input
        type="number"
        value={age}
        onChange={(e) => setAge(e.target.value)}
      />

      <p>{canVote ? "You can vote!" : "You cannot vote :( sorry!"}</p>
    </form>
  );
};
```
