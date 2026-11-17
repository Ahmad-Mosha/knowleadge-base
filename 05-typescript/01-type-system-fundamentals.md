# Type System Fundamentals

[Home](../README.md) › [TypeScript](README.md) › Type System Fundamentals

> **In one sentence:** TypeScript checks that the shapes of your values are used consistently, using types that it mostly infers and that are removed before the program runs, so it catches mistakes in your own code but gives no protection against data that arrives from outside at run time.

**Before this:** [Types and coercion](../04-javascript/01-types-and-coercion.md), [`this`, prototypes and classes](../04-javascript/03-this-prototypes-classes.md).
**After this:** [Generics and advanced types](02-generics-advanced-types.md), [patterns for backends](04-patterns-for-backends.md).

## What TypeScript is

TypeScript is a **superset of JavaScript**: valid JavaScript is valid TypeScript (possibly with type errors reported). It adds syntax for describing types, and a **type checker** that analyses the program without running it.

Three properties define how it behaves.

| Property | Meaning |
|---|---|
| **Static** | Checking happens before the program runs, on the source code |
| **Erased** | Types are removed during compilation. The output is JavaScript with no trace of them |
| **Gradual** | You can mix typed and untyped code. `any` switches checking off for a value |

## Erasure: types do not exist at run time

```ts
interface User { id: number; email: string }

function greet(user: User): string {
  return `Hello ${user.email}`;
}
```

compiles to

```js
function greet(user) {
  return `Hello ${user.email}`;
}
```

Consequences that you must always keep in mind:

- **No run-time checks are added.** If `greet` is called with something that is not a `User`, nothing stops it.
- **You cannot test a type at run time.** `if (x instanceof User)` does not compile for an interface, because `User` is not a value.
- **Types cost nothing at run time.** They do not affect performance or memory.
- **Data from outside is not checked.** `JSON.parse`, `req.body`, a database row, an environment variable: the compiler believes whatever type you write for them.

```ts
const user = JSON.parse(body) as User;     // a promise to the compiler, not a check
user.email.toLowerCase();                  // crashes at run time if `email` is missing
```

This is the most important practical fact about TypeScript in a backend. Every boundary of the system needs **run-time validation**. See [patterns for backends](04-patterns-for-backends.md).

### Type space and value space

Every name lives in the type space, the value space, or both.

| Declaration | Type | Value |
|---|---|---|
| `interface`, `type` | Yes | No |
| `const`, `let`, `function` | No | Yes |
| `class` | Yes (the instance type) | Yes (the constructor function) |
| `enum` | Yes | Yes (an object at run time) |
| `namespace` | Yes | Yes |

`typeof` means different things in the two spaces:

```ts
const config = { port: 3000, host: 'localhost' };

typeof config;                    // value space: the JavaScript operator → 'object'
type Config = typeof config;      // type space: { port: number; host: string }
```

## Structural typing

TypeScript decides compatibility by **structure**: does the value have the required properties with compatible types? The name of the type does not matter. (Java and C# use *nominal* typing, where the declared name matters.)

```ts
interface Point { x: number; y: number }

class Coordinate {
  constructor(public x: number, public y: number, public label: string) {}
}

function length(p: Point) { return Math.hypot(p.x, p.y); }

length(new Coordinate(3, 4, 'home'));     // OK: it has x and y. The extra property is fine
```

A type is a **set of values**. `Point` is the set of all objects that have at least a numeric `x` and a numeric `y`. `Coordinate` values are inside that set. "Assignable to" means "is a subset of".

Two results of this follow directly:

**Extra properties are allowed**, except in one case. When you write an **object literal** directly where a type is expected, TypeScript runs an **excess property check**, because an unknown property in a literal is almost always a typo.

```ts
const p: Point = { x: 1, y: 2, z: 3 };     // Error: 'z' does not exist in type 'Point'

const tmp = { x: 1, y: 2, z: 3 };
const q: Point = tmp;                      // OK: not a fresh literal
```

**Two different concepts with the same shape are interchangeable.** A `UserId` and an `OrderId` that are both `string` can be mixed up without any error. The fix is [branded types](04-patterns-for-backends.md).

## The basic types

```ts
let n: number = 42;
let s: string = 'text';
let b: boolean = true;
let big: bigint = 10n;
let sym: symbol = Symbol();
let nothing: null = null;
let missing: undefined = undefined;

let list: number[] = [1, 2, 3];                 // or Array<number>
let pair: [string, number] = ['age', 30];       // tuple: fixed length, a type per position
let dict: Record<string, number> = { a: 1 };
let fn: (a: number, b: number) => number = (a, b) => a + b;
```

### Literal types and widening

A **literal type** is a type with exactly one value: `'GET'`, `200`, `true`.

```ts
let method = 'GET';         // type: string       (a `let` can change, so the type is widened)
const verb = 'GET';         // type: 'GET'        (a `const` cannot change)

const req = { method: 'GET' };              // { method: string }  — properties can be reassigned
const req2 = { method: 'GET' } as const;    // { readonly method: 'GET' }
```

`as const` tells the compiler to infer the narrowest type: literal types, `readonly` properties, and `readonly` tuples for arrays.

### `any`, `unknown`, `never`

| Type | Set of values | What you can do with it |
|---|---|---|
| `any` | Everything, with checking **disabled** | Anything. It is assignable to and from every type. Errors are silenced |
| `unknown` | Everything | Nothing, until you narrow it. The safe way to say "I do not know yet" |
| `never` | Nothing (the empty set) | Represents code that cannot be reached and values that cannot exist |

```ts
function parse(input: string): unknown { return JSON.parse(input); }

const data = parse('{"id":1}');
data.id;                                      // Error: 'data' is of type 'unknown'
if (typeof data === 'object' && data !== null && 'id' in data) {
  data.id;                                    // OK after narrowing
}
```

`any` is **contagious**: a value read from an `any` is also `any`, so one `any` can silently remove checking from a large part of the program. `JSON.parse`, `res.json()` and many old library types return `any`.

Rules:

- Use `unknown` for values whose type you do not know.
- The variable in `catch (err)` is `unknown` (with `strict`). Narrow it before use.
- Treat `any` as an escape hatch that needs a reason. Turn on `noImplicitAny` (part of `strict`).

`never` appears when narrowing removes every possibility, and as the return type of functions that never return (they always throw, or loop forever).

### `void`, `null`, `undefined`

- `void` is the return type of a function whose result should be ignored.
- With `strictNullChecks` (part of `strict`), `null` and `undefined` are **separate types** and are not part of `string`, `number` and so on. You must write `string | null` to allow `null`. This removes the most common class of run-time errors: "cannot read properties of undefined".
- Optional properties and parameters (`name?: string`) have type `string | undefined`.

```ts
function find(id: number): User | undefined { /* ... */ }

const user = find(1);
user.email;                // Error: 'user' is possibly 'undefined'
user?.email;               // string | undefined
user!.email;               // non-null assertion: silences the error with NO run-time check. Avoid
```

## `interface` and `type`

Both can describe an object shape. For plain objects they are nearly interchangeable.

```ts
interface User { id: number; email: string }
type UserT = { id: number; email: string };
```

| | `interface` | `type` alias |
|---|---|---|
| Object shapes | Yes | Yes |
| Unions, tuples, primitives, mapped and conditional types | No | Yes |
| Extending | `extends` (errors on conflicting properties) | `&` intersection (conflicts become `never` silently) |
| Declaration merging | Yes: two interfaces with the same name are merged | No: duplicate name is an error |
| Class `implements` | Yes | Yes, for object types |

Declaration merging is what lets you add a property to a library's type, for example a `user` field on Express's `Request`. See [patterns for backends](04-patterns-for-backends.md).

A reasonable convention: `interface` for object shapes that describe a public contract, `type` for everything else. Being consistent matters more than the choice.

## Unions and intersections

A **union** `A | B` is a value that is an `A` **or** a `B`. You may only use what is common to all members, until you narrow.

An **intersection** `A & B` is a value that is an `A` **and** a `B` at the same time. It has the properties of both.

```ts
type Id = string | number;

type Timestamped = { createdAt: Date };
type Order = { id: string; total: number };
type StoredOrder = Order & Timestamped;      // { id, total, createdAt }
```

The names feel reversed until you think in sets. The union of two sets of values is larger, so you know **less** about a value in it. The intersection is smaller, so you know **more**.

## Narrowing

**Narrowing** is how TypeScript goes from a wide type to a specific one. The compiler follows your control flow and understands ordinary JavaScript checks.

```ts
function format(value: string | number | Date | null) {
  if (value === null) return '';                    // null is removed below
  if (typeof value === 'string') return value.trim();          // string
  if (value instanceof Date) return value.toISOString();       // Date
  return value.toFixed(2);                          // only number is left
}
```

| Check | Narrows by |
|---|---|
| `typeof x === 'string'` | Primitive type |
| `x instanceof Class` | Prototype chain (classes only, not interfaces) |
| `'prop' in x` | Presence of a property |
| `x === null`, `x == null`, truthiness | `null` / `undefined` / falsy values |
| `x.kind === 'literal'` | Discriminant property |
| `Array.isArray(x)` | Arrays |
| A user-defined **type guard** | Whatever you declare |

### Discriminated unions

This is the most useful pattern in TypeScript. Give every member of a union a common property with a different **literal** type (the **discriminant**). Checking that property narrows the whole object.

```ts
type PaymentResult =
  | { status: 'succeeded'; chargeId: string }
  | { status: 'failed'; reason: string; retryable: boolean }
  | { status: 'pending'; checkAfterMs: number };

function describe(result: PaymentResult): string {
  switch (result.status) {
    case 'succeeded': return `Charged: ${result.chargeId}`;       // chargeId exists only here
    case 'failed':    return `Failed: ${result.reason}`;
    case 'pending':   return `Retry in ${result.checkAfterMs} ms`;
    default: {
      const unreachable: never = result;            // compile error if a case is missing
      throw new Error(`Unhandled: ${JSON.stringify(unreachable)}`);
    }
  }
}
```

The `never` assignment in `default` is an **exhaustiveness check**. If someone adds a fourth status later, this line stops compiling until the new case is handled. The compiler finds every place that needs to change.

Compare with the alternative, one type with optional fields:

```ts
type Bad = { status: string; chargeId?: string; reason?: string; checkAfterMs?: number };
```

This allows impossible combinations (a `succeeded` result with a `reason`) and forces checks for `undefined` everywhere. The union makes **invalid states impossible to represent**.

### Type guards and assertion functions

When a built-in check is not enough, write a function whose return type tells the compiler what was proved.

```ts
// type guard: returns a boolean and narrows in the `if`
function isUser(value: unknown): value is User {
  return typeof value === 'object' && value !== null
    && typeof (value as any).id === 'number'
    && typeof (value as any).email === 'string';
}

// assertion function: throws, or narrows for the rest of the scope
function assertDefined<T>(value: T, name: string): asserts value is NonNullable<T> {
  if (value === null || value === undefined) throw new Error(`${name} is required`);
}
```

The compiler **trusts** the body of a type guard. If the body is wrong, the type is wrong. In practice, generate these from a schema library rather than writing them by hand.

## Inference

You do not need to annotate everything. TypeScript infers types from values, and from context.

```ts
const port = 3000;                                 // number (literal 3000 for const)
const users = [{ id: 1 }, { id: 2 }];              // { id: number }[]
const ids = users.map(u => u.id);                  // number[]; `u` is typed from context
```

A practical rule:

- **Annotate function parameters** and **exported function return types**. They are the contracts between parts of the program. An explicit return type also makes errors appear in the function, where the mistake is, and not in a caller far away.
- **Let local variables be inferred.**

## Type assertions and `satisfies`

`value as T` is a **type assertion**. It tells the compiler to treat the value as `T`. Nothing is checked or converted at run time. It is not a cast.

```ts
const el = input as User;              // "trust me"
const n = '42' as unknown as number;   // a double assertion forces anything. n is still a string
```

Assertions are sometimes necessary, but each one is a place where the compiler has stopped helping you.

`satisfies` is the safe alternative when you want to **check** a value against a type without **changing** its inferred type:

```ts
type Routes = Record<string, { method: 'GET' | 'POST'; auth: boolean }>;

const routes = {
  listUsers: { method: 'GET', auth: true },
  createUser: { method: 'POST', auth: true },
} satisfies Routes;

routes.listUsers;          // OK and precisely typed: the compiler knows this key exists
routes.missing;            // Error
```

With an annotation (`const routes: Routes = ...`) the checking is the same, but the precise keys are lost and `routes.anything` is allowed.

## Functions

```ts
function createUser(email: string, name?: string, role: Role = 'member'): User { /* ... */ }

function log(level: Level, ...parts: unknown[]): void { /* ... */ }

// overloads: several call signatures, one implementation
function find(id: number): Promise<User | undefined>;
function find(email: string): Promise<User | undefined>;
function find(key: number | string): Promise<User | undefined> { /* ... */ }
```

Prefer a union parameter or [generics](02-generics-advanced-types.md) to overloads when you can. Overloads are checked less strictly against their implementation.

`this` can be typed as a fake first parameter (`function f(this: Server, ...)`), which lets the compiler catch the [lost `this`](../04-javascript/03-this-prototypes-classes.md) mistakes.

## Classes

TypeScript adds access modifiers and a few conveniences on top of JavaScript classes.

```ts
abstract class Repository<T> {
  constructor(protected readonly pool: Pool) {}       // parameter property: declares and assigns the field
  abstract findById(id: string): Promise<T | undefined>;
}

class UserRepository extends Repository<User> implements Disposable {
  #cache = new Map<string, User>();                   // real run-time privacy (JavaScript)
  private hits = 0;                                   // compile-time privacy only

  async findById(id: string) { /* ... */ return undefined; }
  [Symbol.dispose]() { this.#cache.clear(); }
}
```

- `public`, `protected`, `private` and `readonly` are **compile-time only**. At run time a `private` field is an ordinary property, visible to `JSON.stringify` and to any JavaScript code. `#field` is the JavaScript feature with real privacy.
- `implements` only checks that the class matches the interface. It adds nothing at run time.
- Classes are compared **structurally** too, except that a class with a `private` or `protected` member is only compatible with itself and its subclasses.

## Enums and their alternative

```ts
enum Role { Admin = 'ADMIN', Member = 'MEMBER' }      // creates a real object at run time
```

Enums are one of the few TypeScript features that **generate code**. Numeric enums also have surprises (a reverse mapping, and historically any number was accepted). Many teams use a plain object with `as const` instead, which is ordinary JavaScript plus a derived type:

```ts
const Role = { Admin: 'ADMIN', Member: 'MEMBER' } as const;
type Role = (typeof Role)[keyof typeof Role];         // 'ADMIN' | 'MEMBER'
```

or simply a union of string literals: `type Role = 'ADMIN' | 'MEMBER'`.

This choice also matters for tooling. Node.js can run TypeScript files directly by **stripping** types, which only works for syntax that can be erased. `enum`, `namespace` and constructor parameter properties need real code generation, so they are not supported in that mode. See [compiler, tsconfig and modules](03-compiler-tsconfig-modules.md).

## Where the type system is not sound

A **sound** type system guarantees that a value always matches its static type. TypeScript chooses convenience over soundness in several places. Know them:

| Hole | Example |
|---|---|
| `any` | Disables checking and spreads |
| Type assertions | `x as T`, `x!` |
| Index access | `arr[10]` has type `T`, not `T \| undefined`, unless `noUncheckedIndexedAccess` is on |
| Mutable array covariance | A `Dog[]` is accepted as an `Animal[]`, and then a `Cat` can be pushed into it |
| Method parameter bivariance | Parameters of methods declared with method syntax are checked loosely |
| Data from outside | `JSON.parse`, `req.body`, database rows, `process.env` |
| Incorrect third-party type definitions | The `.d.ts` says one thing, the library does another |

```ts
const xs: number[] = [1, 2, 3];
const x = xs[10];            // type: number. Value: undefined
x.toFixed();                 // compiles. Crashes at run time
```

Enable `strict` and `noUncheckedIndexedAccess`, validate at boundaries, and keep assertions rare. That closes most of these holes.

## How this connects

- **Run-time behavior is JavaScript's.** Coercion, `this`, prototypes and the event loop are unchanged. See [JavaScript](../04-javascript/README.md).
- **Erasure means boundaries need validation.** See [patterns for backends](04-patterns-for-backends.md) and [web security](../08-backend-engineering/03-web-security.md).
- **Discriminated unions model states and errors.** See [software design](../08-backend-engineering/10-software-design.md).
- **Classes are still prototypes** at run time. See [`this`, prototypes and classes](../04-javascript/03-this-prototypes-classes.md).
- **Decorator metadata** is how NestJS recovers constructor parameter types after erasure. See [NestJS internals](../06-nodejs/08-nestjs-internals.md).
- **Strictness flags** are set in tsconfig. See [compiler, tsconfig and modules](03-compiler-tsconfig-modules.md).

## Interview questions

<details>
<summary><strong>🟢 Q1. What is TypeScript, and what happens to types at run time?</strong></summary>

TypeScript is JavaScript with a static type system. I write types in the source, and the compiler checks before the program runs that values are used consistently with those types.

At run time the types do not exist. The compiler removes them and outputs plain JavaScript. So TypeScript adds no run-time checks and no run-time cost. It also means it cannot protect me from data that comes from outside, like a request body. For that I need run-time validation.

</details>

<details>
<summary><strong>🟢 Q2. What is the difference between <code>any</code> and <code>unknown</code>?</strong></summary>

Both can hold any value. With `any`, the compiler stops checking: I can call anything on it and assign it to anything, and whatever I read from it is also `any`, so it spreads. With `unknown`, I cannot do anything with the value until I narrow it with a check like `typeof` or a type guard.

So `unknown` is the type-safe choice when I really do not know the type, for example for parsed JSON or a caught error. `any` is an escape hatch that I try to avoid.

</details>

<details>
<summary><strong>🟢 Q3. What is the difference between <code>interface</code> and <code>type</code>?</strong></summary>

Both can describe the shape of an object. A `type` alias can also name any other type: unions, tuples, primitives, mapped and conditional types. An interface can only describe object shapes, but it supports declaration merging: if I declare the same interface twice, the declarations are combined. That is how I add fields to a library's types, for example to the Express request.

For extending, an interface uses `extends` and reports an error if properties conflict. A type uses an intersection, where conflicts quietly produce `never`. In practice I use interfaces for object contracts and types for unions and computed types, and I stay consistent.

</details>

<details>
<summary><strong>🟡 Q4. What does structural typing mean? Give an example where it surprises people.</strong></summary>

It means that two types are compatible if their shapes are compatible, whatever their names are. If a function expects an object with `x` and `y`, I can pass any object that has at least those two properties with the right types.

The first surprise is that different concepts with the same shape are interchangeable. If `UserId` and `OrderId` are both aliases of `string`, I can pass one where the other is expected and the compiler says nothing. The fix is a branded type. The second surprise is excess property checks: an object literal with an extra property is an error when I pass it directly, but the same object passed through a variable is accepted.

</details>

<details>
<summary><strong>🟡 Q5. What is a discriminated union and why is it useful?</strong></summary>

It is a union of object types where every member has a common property with a different literal value, like `status: 'succeeded'` and `status: 'failed'`. When I check that property in an `if` or a `switch`, TypeScript narrows the value to that member, so I can only access the fields that exist in that case.

It is useful because it makes invalid states impossible to represent. A failed result cannot have a charge ID, and a successful one cannot have an error reason. And with a `never` check in the default branch, the compiler tells me every place I have to update when a new case is added.

</details>

<details>
<summary><strong>🟡 Q6. What is the difference between a type assertion and <code>satisfies</code>?</strong></summary>

A type assertion with `as` tells the compiler to treat a value as a certain type. It overrides what the compiler inferred, and nothing is checked at run time. If I am wrong, I get a run-time error later.

`satisfies` checks that a value is compatible with a type, but it keeps the value's own, more precise inferred type. So I get an error if the object does not match the expected shape, and I still get exact keys and literal types when I use it. I use `satisfies` for configuration objects and lookup tables, and I keep `as` for the rare cases where I really know more than the compiler.

</details>

<details>
<summary><strong>🟡 Q7. What does <code>strict</code> mode enable, and which flags matter most?</strong></summary>

`strict` turns on a group of checks. The two most important are `strictNullChecks` and `noImplicitAny`. With `strictNullChecks`, `null` and `undefined` are not part of every type. I have to say explicitly when a value can be missing and handle it. That removes most "cannot read property of undefined" errors. With `noImplicitAny`, a parameter that I forgot to type is an error instead of silently being `any`.

It also includes `strictFunctionTypes` for safer function parameter checking, `strictPropertyInitialization` for class fields, and it makes the variable in a `catch` clause `unknown`. On top of `strict` I usually add `noUncheckedIndexedAccess`, so that reading an array element or a record key gives a type that includes `undefined`.

</details>

<details>
<summary><strong>🔴 Q8. Is TypeScript's type system sound? Give examples.</strong></summary>

No, and that is a deliberate choice to stay practical and compatible with JavaScript. A sound system would guarantee that a value always matches its static type. TypeScript has several holes.

The obvious ones are `any` and type assertions, which let me say anything. Less obvious: indexing an array out of bounds gives the element type, not `undefined`, unless I enable `noUncheckedIndexedAccess`. Arrays are covariant even though they are mutable, so I can pass a `Dog[]` as an `Animal[]` and the function can push a `Cat` into it. Method parameters are checked bivariantly. And all data from outside, like `JSON.parse` or a database row, has whatever type I claim.

So I treat the compiler as a very good tool for consistency inside my code, not as a proof. I enable the strict flags, avoid assertions, and validate data at the edges at run time.

</details>

<details>
<summary><strong>🔴 Q9. A function is typed as returning <code>User</code>, but in production <code>user.email</code> is <code>undefined</code>. How is that possible, and how do you prevent it?</strong></summary>

The type says what I promised, not what happened at run time. The value came from somewhere the compiler could not see. Typical cases: a database row was asserted to be a `User` but the query did not select `email` or the column is nullable. A response from another service was typed with `as`. A library returned `any` and it flowed into my `User` type without an error. An array index or a record lookup was out of range. Or the type definition of a dependency was wrong.

To prevent it, I validate at every boundary with a schema, and I derive the TypeScript type from that schema so the two cannot drift apart. For the database, I use a tool that generates types from the real schema, or I validate rows. I enable `strict` and `noUncheckedIndexedAccess`, I ban `any` with a lint rule, and I review every `as` in code review.

</details>

## Key takeaways

- Types are checked before run time and then erased. No run-time checks exist unless you write them.
- Compatibility is structural: shape, not name.
- `unknown` is the safe top type. `any` turns checking off and spreads. `never` is the empty type.
- Turn on `strict`. `strictNullChecks` is the single most valuable check.
- Narrowing follows control flow. Discriminated unions with a `never` exhaustiveness check are the core modelling tool.
- `as` is a promise, not a check. Prefer `satisfies` and real validation.
- The type system has known holes. Validate at boundaries.

---

← [TypeScript](README.md) | Next: [Generics and Advanced Types →](02-generics-advanced-types.md)
