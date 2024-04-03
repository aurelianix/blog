---
layout: post
title: "TIL: React Memoization"
date: 2024-04-01 12:16 +0200
categories:
- technology
tags:
- react
- TIL
---
Today I learned more about React memoization. I was stuck optimizing a component that was re-rendering too often and after digging a bit, some things stuck out to me.

Memoization, for the uninitiated, is the process of storing the results of expensive function calls and returning the cached result when the same inputs occur again. React has a built-in way to do this with the `React.memo` function. By default, React will re-render whenever its parent re-renders **even if the props haven't changed**. You can use `React.memo` tell React to skip re-rendering if the props haven't changed, but this requires that your component be pure.

I was hesitant to using memo because I thought my component wasn't pure when I access context or use some internal state. That is not true. Internal state (at least through hooks) and context are still considered pure. You can test this yourself:

```jsx
import { memo, useEffect, useState } from "react";

let render = 0;
let renderMemo = 0;

function Child({ name }) {
  const [state, setState] = useState(0);
  render++;
  console.count("norm");
  useEffect(() => {
    const id = setInterval(() => {
      setState((state) => state + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
  return (
    <div>
      Component Counter:&nbsp;{state}
      <br />
      Component Render:&nbsp;{render}
    </div>
  );
}

function ChildForMemo({ name }) {
  const [state, setState] = useState(0);
  renderMemo++;
  console.count("memo");
  useEffect(() => {
    const id = setInterval(() => {
      setState((state) => state + 1);
    }, 1000);
    return () => clearInterval(id);
  }, []);
  return (
    <div>
      Memo Counter:&nbsp;{state}
      <br />
      Memo Render:&nbsp;{renderMemo}
    </div>
  );
}

const ChildMemo = memo(ChildForMemo);

export { Child, ChildMemo };
```

ChildForMemo will only render when its name changes or when its internal state changes. Child will render every time its parent renders. This is a simple example, but it shows that you can still use memo even if you have internal state or context.

However, memo is much less useful than I had hoped and only helps with the most simple component. The reason is that memo only does a shallow comparison of the props. If your props are objects or arrays, the component will always re-render because you get a new reference every time the component renders. Unfortunately, this also applies to children because children are just props and will be passed as object or array.

There is a way around this, but it's a bit more involved. You can use `useMemo` to memoize the props that you pass to the child. This way, the child will only re-render when the memoized props change.

```jsx
import { memo, useEffect, useState, useMemo } from "react";

function App() {
    // memoizing the children will make sure the same reference is passed every time.
    const children = useMemo(()=><Child></Child>,[]);
    return (
        <MyComponent>
            {children}
        </MyComponent>
    )
}
```

For now, I won't be memoizing my components much. I hear SolidJS or Preact handle this much more elegantly with signals since signals always will have the same reference. Another solution would be using value types or objects with value semantics (which js doesn't have natively).