# Generics and Advanced Types

[Home](../README.md) › [TypeScript](README.md) › Generics and Advanced Types

> **In one sentence:** Generics are parameters for types, and together with `keyof`, indexed access, mapped types and conditional types they form a small language for computing new types from existing ones, so that one source of truth can produce every related type.

**Before this:** [Type system fundamentals](01-type-system-fundamentals.md).
**After this:** [Patterns for backends](04-patterns-for-backends.md), [NestJS internals](../06-nodejs/08-nestjs-internals.md).

## Generics: type parameters

A function has value parameters, so that one function works for many values. A **generic** has **type parameters**, so that one definition works for many types while keeping the relationship between them.

```ts
function first(items: unknown[]): unknown { return items[0]; }       // the type is lost
function firstT<T>(items: T[]): T | undefined { return items[0]; }   // the type is carried through

const n = firstT([1, 2, 3]);          // number | undefined  — T inferred as number
const s = firstT(['a', 'b']);         // string | undefined
```

`T` is a placeholder that is filled in at each call. Usually the compiler **infers** it from the arguments. You can also pass it explicitly: `firstT<string>([])`.

The test for a useful type parameter: it should **connect at least two things** (two parameters, or a parameter and the return type). A type parameter that appears only once does nothing.

```ts
function log<T>(value: T): void {}             // T is pointless here; use `unknown`
function identity<T>(value: T): T {}           // T connects input and output
```

Generic types, interfaces and classes work the same way:

```ts
interface Page<T> { items: T[]; nextCursor: string | null }

type ApiResponse<T> =
  | { ok: true; data: T }
  | { ok: false; error: { code: string; message: string } };

class Cache<K, V> {
  private store = new Map<K, V>();
  get(key: K): V | undefined { return this.store.get(key); }
  set(key: K, value: V): void { this.store.set(key, value); }
}
```

### Constraints

Inside a generic function, `T` could be anything, so you can do almost nothing with it. A **constraint** (`extends`) says what `T` must at least be.

```ts
function longest<T extends { length: number }>(a: T, b: T): T {
  return a.length >= b.length ? a : b;
}

longest('abc', 'de');          // OK: strings have length
longest([1], [1, 2]);          // OK
longest(1, 2);                 // Error: number has no length
```

In a constraint, `extends` means "is assignable to", not class inheritance.

### Defaults

```ts
interface Event<TPayload = unknown> { name: string; payload: TPayload }
```

### Connecting a key to its object

```ts
function getProp<T, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key];
}

const user = { id: 1, email: 'a@b.c' };
getProp(user, 'email');        // string
getProp(user, 'id');           // number
getProp(user, 'name');         // Error: '"name"' is not assignable to '"id" | "email"'
```

This uses two operators that the rest of the page depends on.

## `keyof`, `typeof` and indexed access

| Operator | Result |
|---|---|
| `keyof T` | The union of `T`'s property names |
| `T[K]` | The type of property `K` in `T` (indexed access) |
| `typeof value` | The type of a value (in a type position) |

```ts
interface User { id: number; email: string; role: 'admin' | 'member' }

type UserKey = keyof User;              // 'id' | 'email' | 'role'
type Email = User['email'];             // string
type Role = User['role'];               // 'admin' | 'member'
type AnyValue = User[keyof User];       // number | string

const statuses = ['pending', 'paid', 'shipped'] as const;
type Status = (typeof statuses)[number];       // 'pending' | 'paid' | 'shipped'
```

The last line is a common pattern: write the list of values **once**, as a run-time array, and derive the union type from it. The array is available for validation at run time and the type for checking at compile time.

## Mapped types

A **mapped type** creates a new object type by going over a union of keys.

```ts
type Flags<T> = { [K in keyof T]: boolean };

type UserFlags = Flags<User>;       // { id: boolean; email: boolean; role: boolean }
```

Modifiers can be added or removed with `+` and `-`:

```ts
type MyPartial<T>  = { [K in keyof T]?: T[K] };              // add optional
type MyRequired<T> = { [K in keyof T]-?: T[K] };             // remove optional
type MyReadonly<T> = { readonly [K in keyof T]: T[K] };      // add readonly
type Mutable<T>    = { -readonly [K in keyof T]: T[K] };     // remove readonly
```

Keys can be renamed or filtered with `as`:

```ts
type Getters<T> = {
  [K in keyof T as `get${Capitalize<string & K>}`]: () => T[K];
};
// Getters<{ name: string }> → { getName: () => string }

type OnlyStrings<T> = {
  [K in keyof T as T[K] extends string ? K : never]: T[K];       // mapping a key to `never` removes it
};
```

## The built-in utility types

Most of these are short mapped or conditional types from the standard library. Know them: they cover most daily needs.

| Utility | Result |
|---|---|
| `Partial<T>` | All properties optional |
| `Required<T>` | All properties required |
| `Readonly<T>` | All properties readonly (shallow) |
| `Pick<T, K>` | Only the properties `K` |
| `Omit<T, K>` | Everything except `K` |
| `Record<K, V>` | An object with keys `K` and values `V` |
| `Exclude<U, E>` | Remove members from a union |
| `Extract<U, E>` | Keep only matching members of a union |
| `NonNullable<T>` | Remove `null` and `undefined` |
| `ReturnType<F>` | Return type of a function type |
| `Parameters<F>` | Parameter types as a tuple |
| `Awaited<T>` | The type a promise resolves to, unwrapped recursively |
| `NoInfer<T>` | Stops this position from being used to infer `T` |

A typical backend use: derive the input types of an API from the entity, so they change together.

```ts
interface User {
  id: string;
  email: string;
  name: string;
  passwordHash: string;
  createdAt: Date;
}

type CreateUserInput = Pick<User, 'email' | 'name'> & { password: string };
type UpdateUserInput = Partial<Pick<User, 'email' | 'name'>>;
type PublicUser = Omit<User, 'passwordHash'>;

type UserService = { findById(id: string): Promise<User | undefined> };
type Found = Awaited<ReturnType<UserService['findById']>>;      // User | undefined
```

A warning about `Omit`: it is not strict about the keys. `Omit<User, 'pasword'>` (with a typo) compiles and removes nothing. And when a new sensitive field is added to `User`, `Omit` exposes it by default. For response types, `Pick` (an allow-list) is safer than `Omit` (a deny-list).

## Conditional types

A **conditional type** chooses between two types, based on a test.

```ts
type IsString<T> = T extends string ? true : false;

type A = IsString<'x'>;         // true
type B = IsString<42>;          // false
```

### `infer`

Inside the `extends` clause, `infer` declares a type variable that the compiler fills in by pattern matching.

```ts
type ElementOf<T> = T extends readonly (infer E)[] ? E : never;
type E1 = ElementOf<string[]>;                       // string

type MyReturnType<F> = F extends (...args: any[]) => infer R ? R : never;
type Unwrap<T> = T extends Promise<infer U> ? Unwrap<U> : T;       // recursive
type U1 = Unwrap<Promise<Promise<number>>>;          // number
```

### Distribution over unions

When the checked type is a **bare type parameter** and you pass a union, the conditional type is applied to **each member separately**, and the results are joined.

```ts
type ToArray<T> = T extends unknown ? T[] : never;
type R = ToArray<string | number>;        // string[] | number[]     (not (string | number)[])

type MyExclude<U, E> = U extends E ? never : U;
type S = MyExclude<'a' | 'b' | 'c', 'a'>;         // 'b' | 'c'
```

This is how `Exclude` and `Extract` work: members that map to `never` disappear from the union. To switch distribution off, wrap both sides in a tuple: `[T] extends [U] ? ... : ...`.

Extracting a member of a discriminated union uses the same idea:

```ts
type Event =
  | { type: 'user.created'; userId: string }
  | { type: 'order.paid'; orderId: string; amount: number };

type EventOf<T extends Event['type']> = Extract<Event, { type: T }>;
type OrderPaid = EventOf<'order.paid'>;       // { type: 'order.paid'; orderId: string; amount: number }
```

## Template literal types

String literal types can be built and matched with template syntax.

```ts
type Method = 'GET' | 'POST';
type Version = 'v1' | 'v2';
type Route = `${Method} /${Version}/users`;
// 'GET /v1/users' | 'GET /v2/users' | 'POST /v1/users' | 'POST /v2/users'

type EventName<T extends string> = `on${Capitalize<T>}`;        // 'onClick' for 'click'

// extract the parameter names from a route pattern
type PathParams<P extends string> =
  P extends `${string}:${infer Param}/${infer Rest}` ? Param | PathParams<`/${Rest}`> :
  P extends `${string}:${infer Param}` ? Param :
  never;

type P = PathParams<'/users/:userId/orders/:orderId'>;       // 'userId' | 'orderId'
```

This is how typed routers give you a `params` object whose keys match the route string.

## Putting it together: a typed event bus

One map from event names to payload types is the single source of truth. Everything else is derived.

```ts
interface Events {
  'user.created': { userId: string; email: string };
  'order.paid': { orderId: string; amount: number };
}

class TypedBus<TEvents extends Record<string, unknown>> {
  private handlers: { [K in keyof TEvents]?: Array<(payload: TEvents[K]) => void> } = {};

  on<K extends keyof TEvents>(name: K, handler: (payload: TEvents[K]) => void): void {
    (this.handlers[name] ??= []).push(handler);
  }

  emit<K extends keyof TEvents>(name: K, payload: TEvents[K]): void {
    this.handlers[name]?.forEach(h => h(payload));
  }
}

const bus = new TypedBus<Events>();
bus.on('order.paid', p => p.amount.toFixed(2));            // `p` is typed from the event name
bus.emit('user.created', { userId: '1', email: 'a@b.c' });
bus.emit('order.paid', { orderId: '1' });                  // Error: 'amount' is missing
bus.emit('order.shipped', {});                             // Error: unknown event
```

## Variance

**Variance** answers this question: if `Dog` is assignable to `Animal`, how do `Box<Dog>` and `Box<Animal>` relate?

| Variance | `Box<Dog>` → `Box<Animal>`? | Applies to |
|---|---|---|
| **Covariant** | Yes (same direction) | Types that only **produce** `T`: return values, readonly properties, `Promise<T>`, `ReadonlyArray<T>` |
| **Contravariant** | No, the reverse: `Box<Animal>` → `Box<Dog>` | Types that only **consume** `T`: function parameters |
| **Invariant** | Neither | Types that both read and write `T` |
| **Bivariant** | Both (unsound) | Method parameters declared with method syntax |

The function-parameter case is the one to understand:

```ts
type Handler<T> = (value: T) => void;

declare let handleAnimal: Handler<Animal>;
declare let handleDog: Handler<Dog>;

handleDog = handleAnimal;      // OK: a function that accepts any Animal can safely receive a Dog
handleAnimal = handleDog;      // Error (strictFunctionTypes): this function may receive a Cat
```

A handler that needs **less** (any animal) can be used where a handler for something **more specific** (a dog) is expected. Not the other way around.

Two unsound exceptions exist for convenience:

- **Mutable arrays are covariant.** `Dog[]` is accepted as `Animal[]`, although the receiver could push a `Cat`. Use `readonly T[]` for parameters you do not modify. It is honest, and it accepts both mutable and readonly arrays.
- **Method syntax is bivariant.** In an interface, `handle(v: T): void` is checked loosely, while `handle: (v: T) => void` is checked strictly under `strictFunctionTypes`. Prefer the property form in callback interfaces.

## Other tools worth knowing

**`const` type parameters** infer literal types without the caller writing `as const`:

```ts
function defineRoutes<const T extends readonly string[]>(routes: T): T { return routes; }
const r = defineRoutes(['/users', '/orders']);       // readonly ['/users', '/orders']
```

**Recursive types:**

```ts
type Json = string | number | boolean | null | Json[] | { [key: string]: Json };

type DeepReadonly<T> = T extends (infer E)[] ? ReadonlyArray<DeepReadonly<E>>
  : T extends object ? { readonly [K in keyof T]: DeepReadonly<T[K]> }
  : T;
```

**Variadic tuples** type functions that take and return argument lists:

```ts
function withRetry<Args extends unknown[], R>(fn: (...args: Args) => Promise<R>) {
  return async (...args: Args): Promise<R> => { /* retry loop */ return fn(...args); };
}
```

The wrapper keeps the exact parameter list and return type of the function it wraps. This is the right way to type decorators, middleware and higher-order functions.

## When to stop

Type-level programming is powerful and easy to overuse.

- **Complex types are code.** Somebody must read, debug and maintain them. Error messages from deep conditional types are very hard to understand.
- **They cost compile time.** Deep recursion and large unions make the checker and the editor slow.
- **A type that is too clever fails in ways that are hard to see.** A mistake in a conditional type often produces `any` or `never` quietly, not an error.

Use advanced types where they **remove duplication between types that must stay in sync**: a schema and its type, an entity and its DTOs, an event map and its handlers. In application code, prefer simple and explicit types. Most of the complex type machinery belongs in libraries.

## How this connects

- **The basics** (unions, narrowing, `never`) are in [type system fundamentals](01-type-system-fundamentals.md).
- **Schema libraries** infer a static type from a run-time schema using exactly these features (`infer`, mapped types). See [patterns for backends](04-patterns-for-backends.md).
- **Generic containers and tokens** are how dependency injection stays typed. See [NestJS internals](../06-nodejs/08-nestjs-internals.md).
- **Query builders and ORMs** use mapped and conditional types to type query results from a schema. See [data access from Node.js](../07-databases/11-data-access-from-node.md).
- **`Awaited` and promise types** model [async JavaScript](../04-javascript/04-async-javascript.md).
- **Compile-time cost** of complex types is discussed in [compiler, tsconfig and modules](03-compiler-tsconfig-modules.md).

## Interview questions

<details>
<summary><strong>🟢 Q1. What are generics and why are they useful?</strong></summary>

Generics are type parameters. They let me write a function, class or type once so it works with many types, while keeping the relationship between the types. For example, a function that returns the first element of an array can say: for an array of `T`, I return a `T`. The caller gets back the exact type that went in, not `any` or `unknown`.

Without generics I would have to choose between writing the same code for every type or losing the type information. They are how collections, promises, repositories and API response wrappers are typed.

</details>

<details>
<summary><strong>🟢 Q2. What do <code>Partial</code>, <code>Pick</code>, <code>Omit</code> and <code>Record</code> do?</strong></summary>

`Partial<T>` makes every property of `T` optional. That is common for update inputs. `Pick<T, K>` creates a type with only the listed properties, and `Omit<T, K>` creates a type with everything except the listed ones. `Record<K, V>` is an object type whose keys are `K` and whose values are `V`, for example a dictionary from string to number.

I use them to derive related types from one base type, like create and update DTOs from an entity, so that I do not write the same fields twice.

</details>

<details>
<summary><strong>🟡 Q3. What does <code>keyof</code> do, and how do you write a type-safe property getter?</strong></summary>

`keyof T` gives the union of the property names of `T`. For a `User` with `id` and `email`, it is the type `'id' | 'email'`.

For a safe getter I use two type parameters: `function get<T, K extends keyof T>(obj: T, key: K): T[K]`. `K` is constrained to be one of the keys of `T`, so a wrong key is a compile error. The return type `T[K]` is an indexed access type, which is the type of that specific property. So `get(user, 'id')` returns a number and `get(user, 'email')` returns a string.

</details>

<details>
<summary><strong>🟡 Q4. What is a mapped type?</strong></summary>

A mapped type builds a new object type by going over a set of keys, usually `keyof T`, and defining the type of each property. The syntax is `{ [K in keyof T]: ... }`.

I can change modifiers while mapping, like adding or removing `readonly` or the optional marker. That is how `Partial`, `Required` and `Readonly` are implemented. I can also rename or remove keys with an `as` clause, for example to create a type with a `getX` method for every property `x`.

</details>

<details>
<summary><strong>🟡 Q5. What is a conditional type, and what does <code>infer</code> do?</strong></summary>

A conditional type has the form `T extends U ? X : Y`. If `T` is assignable to `U`, the result is `X`, otherwise it is `Y`. It is an if-statement at the type level.

`infer` is used inside the `extends` part to capture a piece of the type being matched. For example, `T extends Promise<infer U> ? U : T` says: if `T` is a promise of something, call that something `U` and return it. That is how `ReturnType`, `Parameters` and `Awaited` work. They pattern-match on a type and extract a part of it.

</details>

<details>
<summary><strong>🔴 Q6. What does it mean that conditional types are distributive?</strong></summary>

When the type being checked is a plain type parameter and I pass a union, the conditional type is evaluated for each member of the union separately, and the results are combined into a union.

For example, `type ToArray<T> = T extends unknown ? T[] : never`, applied to `string | number`, gives `string[] | number[]`, not an array of the union. This is what makes `Exclude` work: each member is tested, and the ones that match become `never`, which disappears from a union.

If I do not want this, I wrap both sides in a tuple, like `[T] extends [U]`. Then the union is checked as a whole.

</details>

<details>
<summary><strong>🔴 Q7. Explain covariance and contravariance. Why are function parameters contravariant?</strong></summary>

Variance describes how a generic type relates when its type argument has a subtype relationship. Say `Dog` is a subtype of `Animal`. A type is covariant if the relationship keeps its direction: a producer of dogs can be used where a producer of animals is expected. Return types, promises and readonly arrays are covariant.

Function parameters go the other way, which is contravariance. If some code expects a handler for `Dog`, I can give it a handler that accepts any `Animal`, because it can certainly handle a dog. But if the code expects a handler for any `Animal` and I give it a handler that only works for `Dog`, it may be called with a cat and break. So a handler of the wider type is assignable to a handler of the narrower type.

TypeScript enforces this with `strictFunctionTypes`, but only for function-typed properties. Methods written with method syntax are still checked bivariantly, and mutable arrays are covariant. Both are known unsound spots.

</details>

<details>
<summary><strong>🔴 Q8. When would you decide not to use advanced type-level programming?</strong></summary>

When the cost is higher than the duplication it removes. Complex conditional and recursive types are hard to read, give very long error messages, and can slow down the compiler and the editor. If a clever type has a bug, it often becomes `any` or `never` silently, and the safety I wanted is gone without any warning.

So I use them where types must stay in sync with a single source of truth, such as deriving a type from a validation schema, deriving DTOs from an entity, or typing an event map. That is mostly library-like code that is written once and used in many places. In normal application code I prefer explicit, simple types, even with a little repetition, because the next person can read them immediately.

</details>

## Key takeaways

- A generic is a type parameter. It should connect two or more positions.
- `extends` in a constraint means "assignable to".
- `keyof T` gives the keys, `T[K]` gives a property type, and `typeof value` turns a value into a type.
- Mapped types transform every property. Conditional types choose a type. `infer` extracts a part.
- Conditional types distribute over unions when the checked type is a bare type parameter.
- Parameters are contravariant, outputs are covariant. Mutable arrays and method syntax are unsound exceptions.
- Derive types from one source of truth. Keep application types simple.

---

← [Type System Fundamentals](01-type-system-fundamentals.md) | Next: [Compiler, tsconfig and Modules →](03-compiler-tsconfig-modules.md)
