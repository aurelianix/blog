---
layout: post
title: "Magical Refs in React"
date: 2024-04-04 08:17 +0200
categories:
- technology
tags:
- react
---
I learned React by looking at it. Yes, really. I inherited some React codebase since I was the only dev with JS knowledge on my team, and I had to figure out how to make it work. Since most change requests were small there was no need to reach out to the docs, and after implementing a couple of minor fixes the pieces started to fall into place. I've never seen class components since my first contact with React was with functional components.

For the most part, my intuition about how React works was correct, which says a great deal about the framework's design. There is a lot of complaining about hooks and their complexity, but I find them logical. Maybe not as intuitive as signals, but they make sense to me. React's design is excellent because it allowed me to just look at the code and figure out how it works. That's not something I can say about the magical frameworks like Angular or anything coming out of the .NET ecosystem (I'm looking at you, ASP.NET).

However, Refs never really cliked for me. I thought Refs were used exclusively to access DOM elements. Today I learned that Refs can be used to pass any arbitrary data to and from a component outside of the normal data flow. This makes Refs so powerful, because you can store e.g. multiple DOM elements in a single Ref.

While diving a bit into Refs, I learned a couple of tricks:

### Getting the type of a Ref a component exposes

When doing a `forwardRef`, I used to omit the type signature because Refs where just this annoying thing for me that you had to plaster with `@ts-ignore` to make TypeScript happy. This amazing trick comes from [Stack Overflow](https://stackoverflow.com/questions/76937290/how-can-i-handle-refs-for-pressable-in-react-native):

```tsx
const test = <MyComponent ref={% raw %}{{}{% endraw %} as any} />
```

Now, you just hover `ref` and you'll see something like `(property) React.RefAttributes<View>.ref?: React.LegacyRef<View> | undefined`. Thank me later.


### Merging Refs in forwardRef

Suppose you make a component that exposes a ref to its parent. At the same time, you want to use a ref in the component itself. Making this possible is impossibly easy once you know:

```tsx

const MyComponent = React.forwardRef((props, ref) => {
  const innerRef = useRef(null);

  useImperativeHandle(ref, () => innerRef.current);

  return <div ref={innerRef} />;
});

```

The magic comes from `useImperativeHandle`. This hook allows you to expose a different value to the parent than the one you have in the component. In this case, we expose the `innerRef` to the parent. You can use useImperativeHandle to expose a custom ref to the parent.