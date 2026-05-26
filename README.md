# Table of Contents

1. [The First Principle: Re-renders Aren't the Enemy](#the-first-principle-re-renders-arent-the-enemy)
2. [What Triggers a Re-render](#what-triggers-a-re-render)
3. [The Render Phase: What Actually Happens](#the-render-phase-what-actually-happens)
4. [`React.memo` — Mechanics](#reactmemo--mechanics)
5. [`useMemo` — Mechanics](#usememo--mechanics)
6. [`useCallback` — Mechanics](#usecallback--mechanics)
7. [Referential Equality — The Core Problem](#referential-equality--the-core-problem)
8. [When Memoization Helps](#when-memoization-helps)
9. [When Memoization Hurts](#when-memoization-hurts)
10. [Profiling Methodology](#profiling-methodology)
11. [The React DevTools Profiler](#the-react-devtools-profiler)
12. [Chrome DevTools for React Performance](#chrome-devtools-for-react-performance)
13. [Common Performance Patterns](#common-performance-patterns)
14. [Real-World Performance Anti-patterns](#real-world-performance-anti-patterns)
15. [Compiler-Era Performance (React 19+)](#compiler-era-performance-react-19)
16. [Interview-Ready Answers](#interview-ready-answers)

---

# The First Principle: Re-renders Aren't the Enemy

Most React performance content starts from a false premise: that re-renders are inherently bad and must be minimized. They aren't, and they don't.

**Re-rendering is what React does. It's the unit of work, not a sign of malfunction.**

A re-render only matters if it's slow enough for users to perceive. The threshold for "slow" is roughly:

- **16ms** — one 60fps frame. Stay under this for animations and scroll
- **100ms** — perceived as instant for click responses
- **1000ms** — perceived as a noticeable delay
- **3000ms** — perceived as broken

A component that re-renders 50 times during a session but each render takes 0.3ms produces no user-visible cost. A component that re-renders 5 times but each render takes 80ms causes jank.

This reframes the optimization question. Instead of "how do I prevent re-renders," ask: **"is this re-render causing perceptible work? If so, where is the time actually going?"**

Most of the time, the answer is "nowhere meaningful." React is genuinely fast. Premature memoization is the single most common performance anti-pattern in production React code — adding `useMemo`/`useCallback` everywhere costs CPU on every render (the equality checks aren't free) without buying anything because there's no expensive child to memoize.

The senior move isn't "memoize everything." It's "measure, identify the actual slow thing, fix that."

---

# What Triggers a Re-render

A component re-renders when one of these happens:

1. **Its own state changes** — `setState` or `dispatch` is called with a new value
2. **A hook it uses signals an update** — `useContext` value changes, `useSyncExternalStore` snapshot changes
3. **Its parent re-renders** — by default, all children re-render when the parent does

The third point is the one most engineers underuse. When `<Parent>` re-renders, every child of `<Parent>` is also re-rendered, even if their props didn't change. This is the default. `React.memo` is the opt-in mechanism to skip this.

## The cascade

```jsx
function App() {
  const [count, setCount] = useState(0);
  return (
    <div>
      <Header />          {/* re-renders on every count change */}
      <Counter value={count} />
      <Footer />          {/* re-renders on every count change */}
    </div>
  );
}
```

Every time `count` changes, all three children re-render. `Header` and `Footer` don't even use `count`, but they re-render because their parent does.

This is *almost always fine.* React's reconciler is fast. If `Header` and `Footer` are cheap to render, you've spent 0.1ms on unnecessary work. Don't reach for `React.memo` yet.

It becomes a problem only when:
- A child does expensive work in render (heavy computation, large list iteration)
- The parent re-renders very frequently (every keystroke, every animation frame)
- The child tree is deep and the cumulative re-render cost adds up

## State colocation

A frequently overlooked optimization: **move state down to the smallest component that needs it.**

```jsx
// Bad — count change re-renders the entire app
function App() {
  const [count, setCount] = useState(0);
  return (
    <>
      <HeavyTree />
      <Counter value={count} setCount={setCount} />
    </>
  );
}

// Good — count change only re-renders Counter
function App() {
  return (
    <>
      <HeavyTree />
      <Counter />
    </>
  );
}

function Counter() {
  const [count, setCount] = useState(0);
  return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

This is structurally cleaner *and* faster. Often beats `React.memo` because there's no memoization overhead — the re-render simply doesn't happen.

## Composition as memoization

A related pattern: pass children as JSX rather than rendering them inside the stateful component.

```jsx
// Bad — Heavy re-renders every time count changes
function StatefulShell() {
  const [count, setCount] = useState(0);
  return (
    <div onClick={() => setCount(c => c + 1)}>
      <Heavy />  {/* re-renders on every count change */}
      {count}
    </div>
  );
}

// Good — Heavy doesn't re-render
function App() {
  return <StatefulShell><Heavy /></StatefulShell>;
}

function StatefulShell({ children }) {
  const [count, setCount] = useState(0);
  return (
    <div onClick={() => setCount(c => c + 1)}>
      {children}  {/* same React element across renders */}
      {count}
    </div>
  );
}
```

Why does this work? When `App` renders, it creates `<Heavy />` as a React element and passes it to `StatefulShell` as `children`. When `StatefulShell` re-renders due to `count` changing, `children` is the *same React element reference* — React's reconciler sees no change and skips re-rendering `Heavy`.

This is "memoization via composition" and is often cleaner than `React.memo`. No memoization API needed, no equality checks, just structural restructuring.

---

# The Render Phase: What Actually Happens

To reason about performance, you need a clear mental model of what a re-render does.

A render of component `Foo` means:

1. React calls `Foo(props)` — the function body executes
2. All hooks in `Foo` run, in order
3. JSX is evaluated — `<div>...</div>` becomes `React.createElement('div', ...)` calls
4. The result is a tree of React elements (plain JS objects)
5. React's reconciler diffs this new tree against the previous tree
6. Based on the diff, React decides which DOM mutations are needed
7. (Commit phase, separate) The DOM mutations are applied

Steps 1-5 are the "render phase." Step 7 is the "commit phase." Performance work mostly targets steps 1-6.

## What's free, what's not

**Free or near-free:**
- Function calls themselves
- JSX evaluation (creating plain objects)
- Reconciliation of small subtrees with stable types
- Hook execution that doesn't allocate

**Not free:**
- Heavy computation in the render body (sorting, filtering, transforming large arrays)
- Large list rendering (1000+ items with non-trivial item components)
- Deep tree reconciliation (rare in well-structured apps)
- Creating new object/array/function references that flow into expensive children

This is why "every render is slow" is wrong. A render is slow when *what's in the function body is slow*. The function call itself is meaningless overhead.

---

# `React.memo` — Mechanics

`React.memo` is a higher-order component that wraps a component to skip re-renders when props are shallowly equal to the previous render.

```jsx
const Row = React.memo(function Row({ data, onClick }) {
  return <li onClick={onClick}>{data.label}</li>;
});
```

## What "shallow equality" means

`React.memo` compares each prop using `Object.is` (essentially `===`, with special handling for NaN and -0/+0). For each prop:

```js
Object.is(prevProps[key], nextProps[key])
```

If every key matches, React skips the render. If any key differs, it renders.

**Critical implication:** primitives compare by value, but objects, arrays, and functions compare by *reference*. A new object with the same contents is "different."

```jsx
// Bad — new object every render, React.memo never bails
<Row data={{ id: 1, label: 'a' }} />

// Good — stable reference
const data = useMemo(() => ({ id: 1, label: 'a' }), []);
<Row data={data} />
```

## Custom comparison function

You can override the equality check:

```jsx
const Row = React.memo(function Row({ data, onClick }) {
  return <li>{data.label}</li>;
}, (prev, next) => {
  return prev.data.id === next.data.id;
});
```

The second argument returns `true` to skip render, `false` to render. This is the opposite of `shouldComponentUpdate` in class components, which is a common source of bugs.

Custom comparison is rarely the right answer. If you need it, you're usually solving the wrong problem — your parent is passing unstable references, and you should fix that instead.

## What `React.memo` doesn't do

- **Doesn't stop re-renders from internal state changes** — if a `memo`-wrapped component calls `setState`, it re-renders normally
- **Doesn't stop re-renders from `useContext`** — context changes always trigger a render in consumers
- **Doesn't stop re-renders of children** — if a `memo` component re-renders, its children re-render unless they're also memoized

## When `React.memo` actually pays off

`React.memo` has a cost — the equality check runs every time the parent re-renders. For trivial components, the equality check itself can be slower than just rendering.

It pays off when:
1. The component renders frequently *because of parent re-renders, not its own state changes*
2. Its render is non-trivial (otherwise the check costs more than the render)
3. Its props are typically stable across parent re-renders

It doesn't pay off when:
1. Props are unstable (new objects/arrays/functions every render — the check always fails)
2. The component is cheap to render anyway
3. The component re-renders due to context or its own state

The classic case where it works: rows in a long list, when the parent re-renders for unrelated reasons. Each row's props are stable (each `data` item is the same reference), so the memo check succeeds for unchanged rows.

---

# `useMemo` — Mechanics

`useMemo` caches the return value of a function across renders. It recomputes only when its dependency array changes.

```js
const sortedItems = useMemo(() => items.sort((a, b) => a.id - b.id), [items]);
```

## How it works internally

Each `useMemo` call gets a slot in the fiber's hook list (same as other hooks). The slot stores:
- The previous deps array
- The previous return value

On each render:
1. React looks up the hook slot
2. Compares the new deps with the previous deps using `Object.is`, element by element
3. If all match, returns the cached value — does NOT call the function
4. If any differ, calls the function, stores the new value and deps

The function is only called when needed. Note that `useMemo` always allocates the deps array on every render — that allocation is the unavoidable cost.

## When `useMemo` is correctness, not optimization

Most `useMemo` is optimization. But some uses are about stability of references, which can affect correctness:

```js
const filterOptions = useMemo(() => ({ active: true }), []);
useEffect(() => {
  fetchData(filterOptions);
}, [filterOptions]);  // without useMemo, this would fire every render
```

Without `useMemo`, `filterOptions` would be a new object every render, the effect's dep would always differ, and it would refetch forever.

But — **never rely on `useMemo` for correctness.** The React docs explicitly state that React may discard the cache to free memory (in practice, current React doesn't, but the docs reserve the right to). Use a ref or restructure instead.

## What `useMemo` doesn't help with

```js
const handler = useMemo(() => () => doThing(), []);
```

This works but is wrong. For functions, use `useCallback` — it has the same effect with clearer intent.

```js
const Component = useMemo(() => <ExpensiveChild />, []);
```

This caches a React element. It works but the better pattern is composition (passing children) or `React.memo`.

## Cost of `useMemo`

Every `useMemo` call has overhead:
- Allocate the deps array
- Compare deps with previous
- Store result on the hook slot

For trivial computations, this overhead exceeds the saved work. Memoizing `const x = a + b` is strictly slower than just computing `a + b`.

Reach for `useMemo` when:
1. The computation is genuinely expensive (sort a 10K-item list, complex object construction)
2. The result is used as a dep elsewhere and reference stability matters
3. The result flows into a `React.memo`'d child and you need referential equality

---

# `useCallback` — Mechanics

`useCallback` returns a memoized version of a callback that only changes when its dependencies change.

```js
const handleClick = useCallback(() => {
  doSomething(id);
}, [id]);
```

It's literally sugar for `useMemo(() => fn, deps)`. The signature is just transposed (function-first rather than function-wrapper-first) for ergonomics.

## When `useCallback` helps

The same conditions as `React.memo`:

1. You're passing the callback to a `React.memo`'d child
2. The callback is in a dependency array of another hook

Without `useCallback`, every parent render creates a new function reference. If that function is passed to a `memo` child, the child's prop check fails and it re-renders anyway — defeating `memo`'s purpose.

```jsx
// Bad — onClick is a new function every render
const Parent = () => {
  const [count, setCount] = useState(0);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <Child onClick={() => doStuff()} />  {/* memo doesn't help */}
    </>
  );
};

// Good — onClick is stable
const Parent = () => {
  const [count, setCount] = useState(0);
  const handleStuff = useCallback(() => doStuff(), []);
  return (
    <>
      <button onClick={() => setCount(c => c + 1)}>+</button>
      <Child onClick={handleStuff} />  {/* memo can now bail */}
    </>
  );
};
```

## When `useCallback` doesn't help

Used in isolation, `useCallback` does nothing for performance. The callback being "stable" only matters if something downstream actually checks its reference equality. If the consumer is a plain DOM element (`<button onClick={handler}>`), no one is checking — the work is the same either way.

This is the most common useCallback misuse: wrapping every function in the codebase "just in case." The wrapper costs more than the savings, because nothing downstream uses the stability.

## The "this is the dependency" problem

A function captured in `useCallback` closes over the variables in its scope. Forgetting a dep gives you stale closures:

```js
const handleClick = useCallback(() => {
  console.log(count);  // stale if count changes
}, []);  // missing count
```

The fix is to add `count` to deps — but then the callback isn't stable anymore. This is the classic tension. Options:

1. Add the dep, accept that the callback changes when `count` changes
2. Use a ref to read the latest value without reactive deps:
   ```js
   const countRef = useRef(count);
   countRef.current = count;
   const handleClick = useCallback(() => {
     console.log(countRef.current);
   }, []);
   ```
3. Use the functional setter pattern when possible:
   ```js
   const handleClick = useCallback(() => {
     setCount(c => c + 1);  // no count dep needed
   }, []);
   ```

Option 1 is correct most of the time. Options 2 and 3 are escape hatches.

---

# Referential Equality — The Core Problem

Most React performance bugs trace back to one issue: **new references created on every render flowing into places that check equality.**

JavaScript's equality for non-primitives is reference-based:

```js
{} === {}  // false
[] === []  // false
(() => {}) === (() => {})  // false
```

Every JSX expression creates new objects. Every inline function creates a new function. Every literal object or array creates a new instance.

```jsx
function Component() {
  return (
    <Child
      style={{ color: 'red' }}        // new object every render
      items={[1, 2, 3]}                // new array every render
      onClick={() => doThing()}        // new function every render
      config={{ theme: 'dark' }}       // new object every render
    />
  );
}
```

Every render: four new references. If `Child` is `React.memo`'d, the memo check fails on all four — every render rebuilds the child anyway.

This is why "I added `React.memo` and nothing changed" is a common complaint. Memo can only help if the props are actually stable, and they aren't if the parent creates them inline.

## Fixes, in order of preference

1. **Restructure to avoid the inline creation entirely:**
   ```jsx
   const ITEMS = [1, 2, 3];  // hoisted out of component
   <Child items={ITEMS} />
   ```

2. **Memoize the value:**
   ```jsx
   const items = useMemo(() => [1, 2, 3], []);
   <Child items={items} />
   ```

3. **Move the state down so the parent doesn't re-render (best of all):**
   - Often eliminates the need for memoization entirely

## The context case

Context has its own referential trap:

```jsx
function App() {
  const [user, setUser] = useState(null);
  return (
    <AuthContext.Provider value={{ user, setUser }}>
      {/* every render creates a new value object — all consumers re-render */}
    </AuthContext.Provider>
  );
}
```

Fix:

```jsx
const value = useMemo(() => ({ user, setUser }), [user]);
```

Or split the context — separate `user` and `setUser` into different providers, so consumers can subscribe to just what they need. (`setUser` is stable, so splitting it avoids unnecessary re-renders of consumers that only need the setter.)

---

# When Memoization Helps

The conditions all need to be true:

1. **There's a real cost to skip.** Either an expensive computation (`useMemo`) or an expensive child render (`React.memo`).
2. **The skip mechanism can actually skip.** Props are stable across renders, or deps are stable.
3. **The savings outweigh the cost.** Equality checks have overhead. Memoization isn't free.

Real-world cases where it pays off:

## Long lists with stable rows

```jsx
const Row = React.memo(({ item }) => <li>{item.name}</li>);

function List({ items, onClick }) {
  return items.map(item => <Row key={item.id} item={item} />);
}
```

When `List` re-renders because some unrelated parent state changed, each `Row` checks its prop. If `items[i]` is the same reference, the row skips. With 1000 rows, this is the difference between rendering 1000 components and skipping 1000 equality checks.

## Expensive computations

```jsx
const sortedResults = useMemo(() => {
  return data.filter(d => d.active).sort((a, b) => a.priority - b.priority);
}, [data]);
```

A 100ms sort that runs only when `data` changes is much better than running it on every render due to unrelated state.

## Stabilizing references for downstream effects

```jsx
const options = useMemo(() => ({ method: 'POST', body }), [body]);
useEffect(() => {
  fetchData(options);
}, [options]);
```

Without the memo, `options` is new every render, the effect fires every render. With the memo, it fires only when `body` changes.

## Wrapping context values

Context's `value` should usually be memoized to prevent consumer re-renders.

---

# When Memoization Hurts

## Wrapping cheap components

```jsx
const Label = React.memo(({ text }) => <span>{text}</span>);
```

Cost of the memo check: small, but non-zero (allocations, comparisons).
Cost of just rendering: also small.
Result: no savings, plus added complexity.

## Memoizing primitive computations

```jsx
const total = useMemo(() => a + b, [a, b]);
```

The `useMemo` does more work than `a + b`. The deps array allocation alone exceeds the cost of the addition.

## Memoizing every callback by reflex

```jsx
const handleClick = useCallback(() => setOpen(o => !o), []);
return <button onClick={handleClick}>Toggle</button>;
```

The `<button>` element doesn't care about stability — it'll attach the new handler either way. The `useCallback` is pure overhead.

## When deps change every render anyway

```jsx
const result = useMemo(() => compute(data), [data, makeNewObj()]);
```

If a dep changes every render, the memo never bails. You're paying for the equality check AND running the computation every time.

## Making code harder to read

Memoization adds visual noise. A component with 12 `useMemo` and `useCallback` calls is harder to read than the same component without them. If the perf benefit is unmeasurable, the cognitive cost wins.

## The cumulative tax

In a large codebase where every function is memoized "just in case," the cumulative overhead becomes measurable. Each `useMemo`/`useCallback` is a hook slot, a deps allocation, an equality check. Multiply by hundreds of components, and you have a tax that exceeds anything you'd save.

---

# Profiling Methodology

Before any optimization, measure. The discipline is:

1. **Define what's slow** — be specific. "Page feels slow" is not actionable. "Typing in the search box drops frames" is.
2. **Reproduce it reliably** — find the exact interaction that triggers it
3. **Measure with the right tool** — DevTools Profiler for React-level, Performance tab for browser-level
4. **Identify the bottleneck** — what specifically is taking the time?
5. **Hypothesize a fix** — based on the actual bottleneck, not a guess
6. **Apply the fix in isolation** — one change at a time
7. **Re-measure** — confirm the fix worked

Skipping any of these is how you end up adding `useMemo` everywhere and shipping no measurable improvement.

## Production vs development

**Always profile in production builds.** Development React has:
- Extra checks (PropTypes, warnings)
- Larger reconciler with debug info
- `StrictMode` double-invocations
- No optimizations

A component that takes 50ms in dev might take 5ms in prod. Profiling dev numbers will lead you to optimize the wrong things.

For local profiling, use the `react-dom/profiling` build (Webpack alias: `react-dom$ -> react-dom/profiling`). This keeps the production fast path but enables Profiler instrumentation.

---

# The React DevTools Profiler

The Profiler tab in React DevTools is the primary tool for diagnosing React-level perf.

## Setup

1. Install React DevTools (Chrome, Firefox, or standalone)
2. Open the Profiler tab
3. Click the record button (●)
4. Perform the slow interaction
5. Click stop
6. Analyze the recorded commits

## Reading the flame graph

The flame graph shows each *commit* — one click of the record cycle. For each commit:

- **Width** = how long the component took to render
- **Color** — gray means the component didn't re-render this commit (bailed out)
- **Hover** for details: render time, props that changed

You're looking for:
- **Wide bars** — slow components
- **Many narrow bars in a parent** — too many children re-rendering
- **Components that re-render when their props didn't change** — they might need `React.memo`

## "Why did this render?" — the killer feature

In Profiler settings, enable "Record why each component rendered while profiling." Then for any component, you can see exactly why it re-rendered:

- "Props changed: { onClick, data }"
- "State changed"
- "Context changed"
- "Hooks changed"
- "Parent rerendered"

This single feature replaces hours of guesswork. If you see "Parent rerendered" for a child whose props are stable, that's where `React.memo` could help. If you see "Props changed: onClick" and you didn't intend for `onClick` to change, you've found a referential equality bug.

## Ranked view

The "Ranked" view sorts components by render time within a commit. Useful for finding the single slowest component when you don't know where to look.

## Commits and interactions

The Profiler shows multiple commits — each `setState` causes a commit. Watching the commit count can reveal unnecessary re-renders (e.g., 5 commits during a single user action when 1 was expected). This often points to:
- `useEffect` dispatching unnecessary state updates
- Multiple state setters that could be batched
- Context cascades triggering downstream renders

---

# Chrome DevTools for React Performance

The React DevTools Profiler shows React-level work. Chrome DevTools' Performance tab shows the entire main-thread story — DOM mutations, layout, paint, GC, network — including non-React work.

## When to use which

- **Slow renders that React reports** → React Profiler
- **Jank during scroll, animation, or input** → Chrome Performance
- **Memory growth, leaks, large GC pauses** → Chrome Memory
- **Network-related slowness, waterfall problems** → Chrome Network

## Reading a Performance recording

After recording:

1. **Frames row** — shows actual rendered frames. Green = on time (16ms), yellow/red = janked
2. **Main thread row** — every task that ran on the main thread. Long tasks (>50ms) cause input lag and animation jank
3. **Bottom-up view** — aggregated where time was spent. Useful for finding "death by a thousand cuts" patterns
4. **Call tree** — full call stacks. Useful for tracing specific bottlenecks

For React apps, you'll see:
- **Function call stacks** matching your component names (in dev mode) or generic names (in prod)
- **React internals** — `performWorkOnRoot`, `beginWork`, `completeWork`, `commitMutationEffects`
- **Layout/paint phases** after commits

A flame in the main thread spanning 100ms with "Function call" frames stacked deep usually means heavy component work. Click into the stack to see what specifically.

## Forced synchronous layout (layout thrashing)

Performance tab will flag this with a pink mark. It happens when you:
1. Mutate the DOM
2. Read a layout property (`offsetHeight`, `getBoundingClientRect`, `scrollTop`)
3. Mutate again
4. Read again

Each read forces the browser to recompute layout *before* the read can return. Doing this in a loop is catastrophic — N writes interleaved with N reads = N² layout work.

Fix: separate reads and writes. Read all values first, then perform all writes.

```js
// Bad — N forced layouts
for (const el of elements) {
  el.style.width = el.offsetWidth + 10 + 'px';
}

// Good — read once, then write
const widths = elements.map(el => el.offsetWidth);
elements.forEach((el, i) => {
  el.style.width = widths[i] + 10 + 'px';
});
```

This isn't usually a React issue (React batches DOM writes), but it happens with `useLayoutEffect`, ref-based measurements, or interop with non-React DOM code.

## Long tasks and INP

Chrome shows tasks longer than 50ms as "long tasks." These directly hurt **INP (Interaction to Next Paint)**, a Core Web Vital metric.

INP measures: from user input → next paint with a response. The 75th percentile across a session is the reported metric.

For React apps, the biggest INP wins:
1. **Code-split routes** — smaller initial JS = less parse/eval blocking the first interaction
2. **Lazy-load heavy interactions** — defer the heaviest code paths
3. **`useDeferredValue` for non-urgent updates** — typing stays responsive even while a list filters
4. **`useTransition` for state updates that trigger heavy renders**
5. **Web Workers for CPU-bound work** — anything sync that takes >50ms should consider this

## React 18+ scheduler integration

React 18 plays nice with the browser's scheduler. Long renders get sliced into chunks, with `shouldYield()` calls between fibers. This means React can pause mid-render to handle higher-priority work like user input.

In Chrome Performance, you'll see React tasks broken into multiple "chunks" rather than one monolithic task. This is concurrent mode working as designed.

To take advantage, mark non-urgent updates as transitions:

```js
const [isPending, startTransition] = useTransition();

const handleChange = (e) => {
  setInput(e.target.value);  // urgent
  startTransition(() => {
    setFilter(e.target.value);  // can be interrupted
  });
};
```

Without `startTransition`, both updates are urgent and a slow downstream render blocks input handling.

---

# Common Performance Patterns

## List virtualization

Rendering 10,000 items is slow no matter what. Don't.

Use `react-window`, `@tanstack/react-virtual`, or `react-virtuoso` to render only visible items.

```jsx
import { FixedSizeList } from 'react-window';

<FixedSizeList height={600} itemCount={10000} itemSize={40} width="100%">
  {({ index, style }) => <div style={style}>{items[index].label}</div>}
</FixedSizeList>
```

Trade-offs:
- `Ctrl+F` browser find only matches visible items
- Variable row heights need estimation or measurement (slower)
- Tab navigation can be awkward across virtualized boundaries

For lists over ~200 items with non-trivial item components, virtualization is usually worth it.

## Debounced and throttled updates

Frequent updates (typing, scrolling, mouse move) flood React with state changes.

**Debounce** — wait for a quiet period before firing:

```js
const debouncedSetSearch = useMemo(
  () => debounce(value => setSearch(value), 300),
  []
);

<input onChange={e => debouncedSetSearch(e.target.value)} />
```

**Throttle** — fire at most once per interval:

```js
const throttledScroll = useMemo(
  () => throttle(handleScroll, 100),
  []
);
```

In React 18+, `useDeferredValue` is often a better fit than debouncing — it doesn't have a fixed delay; it just defers the work until the browser is idle.

## Splitting state to avoid cascades

```jsx
// Bad — single state object means every change re-renders consumers of any field
const [form, setForm] = useState({ name: '', email: '', age: 0 });

// Good — separate state means only changed-field consumers re-render
const [name, setName] = useState('');
const [email, setEmail] = useState('');
const [age, setAge] = useState(0);
```

This is more relevant when fields are used by different children that you'd want to optimize independently.

## Memoizing expensive computations near the consumer

```jsx
function Component({ items, filter }) {
  const filtered = useMemo(
    () => items.filter(i => i.name.includes(filter)),
    [items, filter]
  );
  return <List items={filtered} />;
}
```

Filter only when inputs change.

## Lazy initial state

```jsx
const [state, setState] = useState(() => computeExpensiveInitial());
```

The function runs only on mount. Without the lazy form, `computeExpensiveInitial()` would run every render and the result would be discarded after the first.

### Avoiding re-renders with `useSyncExternalStore`

For state that lives outside React (a global store, a browser API):

```js
const theme = useSyncExternalStore(
  (cb) => themeStore.subscribe(cb),
  () => themeStore.getTheme()
);
```

This is the official way to subscribe to external sources. It handles concurrent mode correctness (no tearing) and only re-renders when the snapshot actually changes.

Zustand, Jotai, and Redux all use this hook internally.

---

# Real-World Performance Anti-patterns

## Anti-pattern: Memoizing everything by reflex

```jsx
const x = useMemo(() => a + b, [a, b]);
const y = useMemo(() => `${name}!`, [name]);
const handler = useCallback(() => doStuff(), []);
```

If you're memoizing without a measured benefit, you're adding overhead. The mental cost (code review, debugging, deps array maintenance) compounds.

## Anti-pattern: Inline objects/arrays/functions in props

```jsx
<Component
  style={{ marginTop: 10 }}
  items={items.filter(i => i.active)}
  onClick={() => handleClick()}
/>
```

Each is a new reference every render. If `Component` is memoized, the memo can't bail.

## Anti-pattern: useMemo with new deps every render

```jsx
const result = useMemo(() => expensive(input), [{ x: input.x }]);
```

The second dep is a new object every render. Memo never bails. The function runs every render PLUS you pay the equality check overhead.

## Anti-pattern: Context for high-frequency updates

```jsx
const MouseContext = createContext();

function MouseProvider({ children }) {
  const [pos, setPos] = useState({ x: 0, y: 0 });
  useEffect(() => {
    const handler = (e) => setPos({ x: e.clientX, y: e.clientY });
    window.addEventListener('mousemove', handler);
    return () => window.removeEventListener('mousemove', handler);
  }, []);
  return <MouseContext.Provider value={pos}>{children}</MouseContext.Provider>;
}
```

Every mouse move re-renders every component consuming the context. For a 60fps mouse, that's 60 cascading re-renders per second.

Fix: use an external store (Zustand) with selectors, so components subscribe only to the slice they need. Or use refs and skip React entirely for non-visual tracking.

## Anti-pattern: Deeply nested `React.memo`

```jsx
const A = React.memo(...);
const B = React.memo(...);  // child of A
const C = React.memo(...);  // child of B
```

If A's props are unstable, A re-renders. B's props might still be stable, so B bails — but the check still runs. The memo cost compounds at each level, but the actual savings only apply at the deepest point where references actually stabilize.

Often better to memoize only the leaf that does heavy work, not every intermediate component.

## Anti-pattern: `useEffect` for derived state

```jsx
const [items, setItems] = useState([]);
const [filtered, setFiltered] = useState([]);

useEffect(() => {
  setFiltered(items.filter(i => i.active));
}, [items]);
```

Two renders per `items` change: one to set `items`, one to set `filtered`. Use `useMemo` (derived value) or just compute inline.

```jsx
const filtered = useMemo(() => items.filter(i => i.active), [items]);
```

One render per change.

## Anti-pattern: Mutating state directly

```jsx
items.push(newItem);
setItems(items);  // same reference — React might not re-render reliably
```

This is a correctness bug, not just performance. Always create new references for state updates. Some bailout logic in React (`Object.is` checks) may skip the render thinking nothing changed.

---

# Compiler-Era Performance (React 19+)

The React Compiler (formerly "React Forget") changes the performance landscape significantly. As of React 19, it's available but not yet enabled by default in all setups.

## What it does

The compiler is a Babel plugin that **automatically memoizes everything that should be memoized**. It analyzes your component, identifies which values are stable, which are dependent on what, and inserts the equivalent of `useMemo`/`useCallback`/`React.memo` automatically.

```jsx
// Source code
function Component({ items, filter }) {
  const filtered = items.filter(i => i.name.includes(filter));
  const handleClick = () => doStuff();
  return <Child items={filtered} onClick={handleClick} />;
}

// Compiler output (conceptual)
function Component({ items, filter }) {
  const filtered = $memo(() => items.filter(i => i.name.includes(filter)), [items, filter]);
  const handleClick = $memo(() => () => doStuff(), []);
  return <Child items={filtered} onClick={handleClick} />;
}
```

Result: optimal memoization without writing `useMemo`/`useCallback`. The compiler is more thorough than humans and doesn't make mistakes about dep arrays.

## What this means for your career

For most components, manually written memoization becomes unnecessary. The compiler handles it.

For senior engineers, the implications are:

1. **Understanding *what* memoization does is still essential** — you still need to debug perf issues, and the compiler isn't magic
2. **The "memoize everything" arguments lose force** — the compiler does it better
3. **Profiling discipline matters more** — when the compiler handles the easy cases, the remaining problems are deeper
4. **Restructuring patterns (composition, state colocation) are still important** — the compiler can't restructure your code architecture

## When to enable

Check the React docs for the current rollout status. The compiler is opt-in for now and has known limitations:

- Requires strict mode compliance (your components must be pure)
- Some patterns (mutating refs in render, closures over mutable values) confuse it
- It bails out (falls back to non-memoized code) when it can't prove safety

If your codebase has older patterns or non-trivial mutations, you may see less benefit. New code following modern React conventions gets the most lift.

## Don't preemptively delete memoization

Even when you adopt the compiler, don't rush to delete every `useMemo` and `useCallback`. The compiler will leave them in place and just do its own analysis around them. Clean up gradually as you touch each file.

---

# Interview-Ready Answers

## "Why is my React app slow?"

Diagnose, don't guess. The senior answer:

> "First, I'd identify *what* is slow. Specifically — is it initial load, a particular interaction, scrolling, or animations? Each points to different work. For interactions, I'd open the React DevTools Profiler, record the slow flow, and look for unexpectedly slow components or unnecessary re-renders. For initial load, I'd check the bundle size, look at the network waterfall, and run Lighthouse. Only after measurement would I look at fixes — and the fix depends on what the measurement says."

## "When should I use `React.memo`?"

> "When three things are true: the component renders frequently because of parent re-renders rather than its own state, its render is expensive, and its props are stable across those parent re-renders. The most common real case is rows in a long list where the parent re-renders for unrelated reasons. If any of those three aren't true — props are unstable, render is cheap, or it doesn't re-render unnecessarily — memo is overhead without benefit."

## "What's the difference between `useMemo` and `useCallback`?"

> "`useCallback(fn, deps)` is exactly `useMemo(() => fn, deps)`. They're the same hook with different ergonomics — `useCallback` is sugar for memoizing functions. Both cache a value across renders and recompute only when deps change."

## "What causes unnecessary re-renders?"

> "Three things, in order of frequency: parent re-renders cascading down (default React behavior), context value changes triggering all consumers, and inline object/array/function props breaking memoization. The first is usually fine — React is fast. The second is the most common production issue in apps that use context for high-frequency state. The third is what makes `React.memo` not work — every render creates new references that fail the equality check."

## "How would you optimize a 10,000-row table?"

> "Virtualization, full stop. I'd use `react-window` or `@tanstack/react-virtual` to render only visible rows — usually 20-30 instead of 10,000. Even with `React.memo` on each row, rendering 10,000 mounted components causes layout, paint, and memory issues regardless of how fast each individual render is. Virtualization is the only correct answer for tables of that size."

## "How does `useTransition` help with performance?"

> "It marks the wrapped state update as low-priority. React can interrupt the resulting render to handle higher-priority work like keystrokes. So when you type in a search box that filters a heavy list, the input stays responsive even if the filter render is slow. Without `useTransition`, both updates are urgent, and a slow filter blocks the input."

## "How do you decide if a component needs memoization?"

> "I don't decide upfront — I measure. I write code straightforwardly first, profile when there's a perf complaint, and identify the specific bottleneck. If a child is rendering expensively because its parent re-renders frequently with stable props, that's where `React.memo` helps. If a computation is heavy and runs on every render, that's where `useMemo` helps. Without measurement, memoization is usually a tax for no benefit. With React 19's compiler, much of this becomes automatic anyway."

## "What's the difference between `useEffect` and `useLayoutEffect` for performance?"

> "`useLayoutEffect` runs synchronously after DOM mutation but before paint — it blocks the browser from showing the new frame. Heavy `useLayoutEffect` work causes jank because paint waits. `useEffect` runs after paint, so the browser can show the frame first and run effects later. For performance, prefer `useEffect` unless you need to read DOM layout and write changes before paint (e.g., positioning a tooltip after measuring it)."

## "How would you debug `React.memo` not working?"

> "Open the React Profiler with 'Record why each component rendered' enabled. Trigger the issue. Look at the memoized component — it'll tell you exactly which props changed. Nine times out of ten, it's an inline object, array, or function that's a new reference every parent render. The fix is to memoize that prop with `useMemo`/`useCallback`, or restructure so it's hoisted out of the parent, or move state down so the parent doesn't re-render."

---

# Further Reading

- [React docs — Render and Commit](https://react.dev/learn/render-and-commit) — the official phase model
- [React docs — useMemo](https://react.dev/reference/react/useMemo) — when and when not to use it
- [Mark Erikson — A (Mostly) Complete Guide to React Rendering Behavior](https://blog.isquaredsoftware.com/2020/05/blogged-answers-a-mostly-complete-guide-to-react-rendering-behavior/) — exhaustive
- [Kent C. Dodds — Don't Optimize Your React App, Use One of These Tools Instead](https://kentcdodds.com/blog/optimize-react-re-renders) — the colocation pattern
- [Dan Abramov — Before You memo()](https://overreacted.io/before-you-memo/) — composition over memoization
- [Vercel — Optimizing React for INP](https://vercel.com/blog/how-react-18-improves-application-performance) — concurrent mode and INP
- [React Compiler docs](https://react.dev/learn/react-compiler) — for the upcoming compiler era
- [WebPageTest, Lighthouse, Chrome DevTools Performance docs] — for browser-level profiling
