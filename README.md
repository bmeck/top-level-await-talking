# Intent

The intent of adding top level await is to add a means to allow Modules
to asynchronously initialize their namespace prior to notifying the consumer
that the Module is finished evaluating.

## Uses

### Dynamic dependency pathing

```mjs
const strings = await import(`/i18n/${navigator.language}`);
```

This allows for Modules to use runtime values in order to determine
dependencies. This is useful for things like development/production splits,
internationalization, environment splits, etc.

### Resource initialization

```mjs
const connection = await dbConnector();
```

This allows Modules to represent resources and also to produce errors in 
cases where the Module will never be able to be used.

### Dependency fallbacks

```mjs
let jQuery;
try {
  jQuery = await import('https://cdn-a.com/jQuery');
} catch {
  jQuery = await import('https://cdn-b.com/jQuery');
}
```

# Designs

The main problem space in designing top level await is to aid in detecting
and preventing forms of deadlock that can occur. All examples below will
use a cyclic `import()` to a graph of `'a' -> 'b', 'b' -> 'a'` with both using
top level await to halt progress until the other finishes loading. For brevity
the examples will only show one side of the graph, the other side is a mirror.

## Await during Evaluation

This design would introduce an ability to block Module loading while in the
Evaluation phase of Module loading.

```mjs
await import('b');
```

## Async instantiate blocks

This design would introduce an ability to execute code within the Instantiate
phase of Module loading.

```mjs
// using static as bikeshed for way to define this action is during
// instantiate
static async {
  await import('b');
}
```

Since `import()` requires the target Module finish Evaluation to complete
having a cycle in Instantiate still suffers from deadlock.

# Halting Progress

## Existing Ways to halt progress

### Infinite Loops

```mjs
for (const n of primes()) {
  console.log(`${n} is prime}`);
}
```

Infinite series or lack of base condition means static control structures
are vulnerable to infinite looping.

### Infinite Recursion

```mjs
const fibb = n => (n ? fibb(n - 1) : 1);
fibb(Infinity);
```

Proper tail calls allow for recursion to never overflow the stack. This makes
it vulnerable to infinite recursion.

### Atomics.wait

```mjs
Atomics.wait(shared_array_buffer, 0, 0);
```

Atomics allow blocking forward progress by waiting on an index that never changes.

### `export function then`

```mjs
// a
export function then(f, r) {}
```

```mjs
async function start() {
  const a = await import('a');
  console.log(a);
}
```

Exporting a `then` function allows blocking `import()`.


## Guarding against

### Implementing a TDZ

```mjs
// a
await import('b');

// implement a hoistable then()
export function then(f, r) {
  r('not finished');
};

// remove the rejection
then = null;
```

```mjs
// b
await import('a');
```

Having a `then` in the TDZ is a way to prevent cycles while a module is still 
evaluating. `try{}catch{}` can also be used as a recovery or notification
mechanism.

```mjs
let b;
try {
  b = await import('b');
} catch {
  // do something
}
```

This mechanism could even be codified into `import()` by making it reject if
the target module has not run to end of source text in Evaluation yet.
