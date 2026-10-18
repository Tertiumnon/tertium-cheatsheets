# Keyof

The keyof operator takes an object type and produces a string or numeric literal union of its keys.

```ts
type Point = { x: number; y: number };
type P = keyof Point; // type P = "x" | "y"
```

```ts
type Arrayish = { [n: number]: unknown };
type A = keyof Arrayish; // type A = number
```

```ts
type Mapish = { [k: string]: boolean };
type M = keyof Mapish; // type M = string | number
```

```ts
type Person = { age: number; name: string; alive: boolean };
type Age = Person["age"]; // type Age = number
type I1 = Person["age" | "name"]; // type I1 = string | number
type I2 = Person[keyof Person]; // type I2 = string | number | boolean
```
