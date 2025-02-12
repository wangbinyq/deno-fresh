---
description: |
  Islands enable client side interactivity in Fresh. They are hydrated on the client in addition to being rendered on the server.
---

Islands enable client side interactivity in Fresh. Islands are isolated Preact
components that are rendered on the client. This is different from all other
components in Fresh, as they are usually just rendered on the server.

Islands are defined by creating a file in the `islands/` folder in a Fresh
project. The name of this file must be a PascalCase or kebab-case name of the
island.

```tsx islands/my-island.tsx
import { useSignal } from "@preact/signals";

export default function MyIsland() {
  const count = useSignal(0);

  return (
    <div>
      Counter is at {count}.{" "}
      <button onClick={() => (count.value += 1)}>+</button>
    </div>
  );
}
```

An island can be used in a page like a regular Preact component. Fresh will take
care of automatically re-hydrating the island on the client.

```tsx route/index.tsx
import MyIsland from "../islands/my-island.tsx";

export default function Home() {
  return <MyIsland />;
}
```

## Passing JSX to islands

Islands support passing JSX elements via the `children` property.

```tsx islands/my-island.tsx
import { useSignal } from "@preact/signals";
import { ComponentChildren } from "preact";

interface Props {
  children: ComponentChildren;
}

export default function MyIsland({ children }: Props) {
  const count = useSignal(0);

  return (
    <div>
      Counter is at {count}.{" "}
      <button onClick={() => (count.value += 1)}>+</button>
      {children}
    </div>
  );
}
```

This allows you to pass static content rendered by the server to an island in
the browser.

```tsx routes/index.tsx
import MyIsland from "../islands/my-island.tsx";

export default function Home() {
  return (
    <MyIsland>
      <p>This text is rendered on the server</p>
    </MyIsland>
  );
}
```

## Passing other props to islands

Passing props to islands is supported, but only if the props are serializable.
Fresh can serialize the following types of values:

- Primitive types `string`, `boolean`, `bigint`, and `null`
- Most `number`s (`Infinity`, `-Infinity`, and `NaN` are silently converted to
  `null`)
- Plain objects with string keys and serializable values
- Arrays containing serializable values
- Uint8Array
- JSX Elements (restricted to `props.children`)
- Preact Signals (if the inner value is serializable)

Circular references are supported. If an object or signal is referenced multiple
times, it is only serialized once and the references are restored upon
deserialization. Passing complex objects like `Date`, custom classes, or
functions is not supported.

During server side rendering, Fresh annotates the HTML with special comments
that indicate where each island will go. This gives the code sent to the client
enough information to put the islands where they are supposed to go without
requiring hydration for the static children of interactive islands. No
Javascript is sent to the client when no interactivity is needed.

```html
<!--frsh-myisland_default:default:0-->
<div>
  Counter is at 0.
  <button>+</button>
  <!--frsh-slot-myisland_default:children-->
  <p>This text is rendered on the server</p>
  <!--/frsh-slot-myisland_default:children-->
</div>
<!--/frsh-myisland_default:default:0-->
```

### Nesting islands

Islands can be nested within other islands as well. In that scenario they act
like a normal Preact component, but still receive the serialized props if any
were present.

```tsx islands/other-island.tsx
import { useSignal } from "@preact/signals";
import { ComponentChildren } from "preact";

interface Props {
  children: ComponentChildren;
  foo: string;
}

function randomNumber() {
  return Math.floor(Math.random() * 100);
}

export default function MyIsland({ children, foo }: Props) {
  const number = useSignal(randomNumber());

  return (
    <div>
      <p>String from props: {foo}</p>
      <p>
        <button onClick={() => (number.value = randomNumber())}>Random</button>{" "}
        number is: {number}.
      </p>
    </div>
  );
}
```

In essence, Fresh allows you to mix static and interactive parts in your app in
a way that's most optimal for your use app. We'll keep sending only the
JavaScript that is needed for the islands to the browser.

```tsx route/index.tsx
import MyIsland from "../islands/my-island.tsx";
import OtherIsland from "../islands/other-island.tsx";

export default function Home() {
  return (
    <div>
      <MyIsland>
        <OtherIsland foo="this prop will be serialized" />
      </MyIsland>
      <p>Some more server rendered text</p>
    </div>
  );
}
```
