# KAT Notes: Building petite-vue

## TL;DR

**Always use `pnpm run build` instead of `npm run build` to regenerate dist files.**

---

## The Problem

When you run `npm run build`, npm resolves dependencies using `package-lock.json`. Because `package-lock.json` is gitignored, it is not pinned in the repository. npm will resolve `@vue/reactivity@^3.2.27` (the version range in `package.json`) to whatever the **latest compatible release** is at the time — which as of this writing is `3.2.47`.

When you run `pnpm run build`, pnpm uses `pnpm-lock.yaml`, which **is** committed to the repo and pins `@vue/reactivity` to exactly `3.2.27`.

These two versions behave differently in a way that breaks the build.

---

## Why `@vue/reactivity@3.2.47` Is Incompatible

petite-vue's `createScopedContext()` in `src/context.ts` creates a reactive scope like this:

```ts
const mergedScope = Object.create(parentScope)   // parentScope is itself reactive
const reactiveProxy = reactive(new Proxy(mergedScope, { set() { ... } }))
```

This creates a **proxy-inside-a-proxy** where:
- The inner proxy (`new Proxy(mergedScope, ...)`) has only a `set` trap
- The outer proxy (`reactive(...)`) is Vue's reactive proxy
- `mergedScope`'s prototype chain includes `parentScope`, which is also a reactive proxy

In `@vue/reactivity@3.2.27`, this worked fine.

In `@vue/reactivity@3.2.47`, Vue added a **custom `hasOwnProperty` interceptor** inside the reactive getter to improve dependency tracking:

```js
function hasOwnProperty(key) {
    const obj = toRaw(this);        // unwrap the proxy to its raw target
    track(obj, "has", key);
    return obj.hasOwnProperty(key); // call hasOwnProperty on the raw target
}
```

This interceptor is returned any time code accesses `.hasOwnProperty` on a reactive proxy. The problem is the `toRaw(this)` call: `toRaw` works by reading the `__v_raw` property off the proxy chain. With the nested proxy structure above, reading `__v_raw` traverses the prototype chain from `mergedScope` up to the reactive `parentScope`. Because the receiver is wrong at that point (it's the outer `reactiveProxy`, not `parentScope`), Vue's `get` trap does **not** short-circuit and instead calls `Reflect.get`, which triggers the reactive getter again — which calls `hasOwnProperty` again — infinitely.

The runtime error is:

```
RangeError: Maximum call stack size exceeded
    at Object.get (petite-vue.js:522)
    at Object.get (petite-vue.js:552)
    at toRaw (petite-vue.js:713)
    at Object.hasOwnProperty (petite-vue.js:517)
    at Object.hasOwnProperty (petite-vue.js:519)  ← infinite loop
    at Object.hasOwnProperty (petite-vue.js:519)
    ...
```

---

## Why pnpm Fixes It

`pnpm-lock.yaml` locks `@vue/reactivity` to `3.2.27`, which does not contain the `hasOwnProperty` instrumentation. The recursive trap does not exist, and the code runs correctly.

---

## Guardrails Added

`package.json` now has a `preinstall` hook that blocks accidental `npm install`:

```json
"preinstall": "npx only-allow pnpm"
```

If you accidentally run `npm install`, you will see:

```
Use the right package manager -> pnpm
```

There is also a documentation script as a reminder:

```json
"use_pnpm": "echo Always use pnpm to run these scripts."
```
