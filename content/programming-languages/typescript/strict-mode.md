# TypeScript Strict Mode

TypeScript Strict Mode is the first line of defense against AI-generated code errors. It enables all strict type-checking options, catching type mismatches, null/undefined errors, and implicit type coercions before runtime.

## Why TypeScript Strict Mode Matters with AI

AI can generate syntactically valid JavaScript that TypeScript in non-strict mode accepts, but which contains type errors that cause runtime failures:

```typescript
// ❌ Without strict mode: TypeScript allows this
function processUser(user: User) {
  return user.email.toUpperCase(); // Error if email is undefined!
}

// ✅ With strict mode: TypeScript catches this
function processUser(user: User) {
  return user.email.toUpperCase(); // Error: Object is possibly 'undefined'
}
```

Strict mode forces AI-generated code to be explicit about nullable types, making errors visible during development rather than production.

## Enabling Strict Mode

### tsconfig.json Configuration

```json
{
  "compilerOptions": {
    "strict": true,
    "target": "ES2020",
    "module": "ESNext",
    "lib": ["ES2020"],
    "moduleResolution": "node",
    
    // This enables all strict settings:
    // - noImplicitAny
    // - noImplicitThis
    // - alwaysStrict
    // - strictNullChecks
    // - strictFunctionTypes
    // - strictBindCallApply
    // - strictPropertyInitialization
    // - noImplicitReturns
    // - noFallthroughCasesInSwitch
    // - noUncheckedIndexedAccess
    // - noImplicitOverride
    // - noPropertyAccessFromIndexSignature
    
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist"]
}
```

## Strict Mode Rules and AI-Generated Code

### Rule 1: noImplicitAny

Types must be explicit. No `any` without justification.

```typescript
// ❌ AI often generates this (implicit any)
function process(data) {
  return data.toString();
}

// ✅ Strict mode forces explicit typing
function process(data: string | number): string {
  return data.toString();
}

// ✅ If truly any, must be explicit
function process(data: any): string {
  return data.toString();
}
```

### Rule 2: strictNullChecks

Variables cannot be null/undefined unless explicitly typed.

```typescript
// ❌ Without strict null checks: TypeScript allows this
function getUser(id: string): User {
  const user = users.find(u => u.id === id);
  return user; // Error: might be undefined!
}

// ✅ Strict null checks force proper handling
function getUser(id: string): User | null {
  const user = users.find(u => u.id === id);
  return user;
}

// ✅ Or with non-null assertion (rare)
function getUser(id: string): User {
  const user = users.find(u => u.id === id);
  if (!user) throw new Error('User not found');
  return user;
}
```

### Rule 3: strictFunctionTypes

Function parameter types must be compatible.

```typescript
// ❌ Without strict function types: TypeScript allows this
class Animal {}
class Dog extends Animal {}

let handler: (animal: Animal) => void = (dog: Dog) => {};
handler(new Animal()); // Error at runtime: not a Dog!

// ✅ Strict mode catches this
let handler: (animal: Animal) => void = (animal: Animal) => {};
```

### Rule 4: noImplicitThis

`this` context must be explicit.

```typescript
// ❌ Implicit this
function getUserName() {
  return this.user.name; // Error: this has type any
}

// ✅ Explicit this
function getUserName(this: User) {
  return this.user.name;
}

// ✅ Or arrow function (captures this from enclosing scope)
const getUserName = () => {
  return this.user.name;
};
```

### Rule 5: noImplicitReturns

All code paths must return a value.

```typescript
// ❌ Missing return path
function getStatus(code: number): string {
  if (code === 200) {
    return 'OK';
  }
  // Error: not all code paths return a value
}

// ✅ All paths return
function getStatus(code: number): string {
  if (code === 200) {
    return 'OK';
  }
  return 'ERROR';
}
```

### Rule 6: strictPropertyInitialization

Class properties must be initialized or optional.

```typescript
// ❌ Property not initialized
class User {
  name: string; // Error: not initialized
  email: string;

  constructor(email: string) {
    this.email = email;
  }
}

// ✅ All properties initialized
class User {
  name: string;
  email: string;

  constructor(name: string, email: string) {
    this.name = name;
    this.email = email;
  }
}

// ✅ Or marked optional
class User {
  name?: string;
  email: string;

  constructor(email: string) {
    this.email = email;
  }
}

// ✅ Or initialized with default
class User {
  name: string = '';
  email: string;

  constructor(email: string) {
    this.email = email;
  }
}
```

### Rule 7: noFallthroughCasesInSwitch

Switch cases must break or return.

```typescript
// ❌ Fallthrough case
function getColor(code: number): string {
  switch (code) {
    case 1:
      return 'red';
    case 2:
      return 'blue';
    default:
      return 'gray';
  }
}

// ✅ All cases handled (this was already good)
// But this would fail:
function process(type: string) {
  switch (type) {
    case 'a':
      console.log('A');
      // Error: missing break or return
    case 'b':
      console.log('B');
      break;
  }
}

// ✅ Fixed:
function process(type: string) {
  switch (type) {
    case 'a':
      console.log('A');
      break;
    case 'b':
      console.log('B');
      break;
  }
}
```

### Rule 8: noUncheckedIndexedAccess

Array access must check bounds.

```typescript
// ❌ Unchecked array access
function getFirst(items: string[]): string {
  return items[0]; // Error: might be undefined
}

// ✅ Proper handling
function getFirst(items: string[]): string | undefined {
  return items[0];
}

// ✅ Or check bounds
function getFirst(items: string[]): string {
  if (items.length === 0) throw new Error('Empty array');
  return items[0];
}
```

## Common AI-Generated Errors Caught by Strict Mode

### Error 1: Unsafe Optional Chaining

```typescript
// ❌ AI writes this
function getName(user: User) {
  return user.profile.name; // Error: profile might be undefined
}

// ✅ Strict mode forces this
function getName(user: User) {
  return user.profile?.name;
}
```

### Error 2: Missing Type Annotations

```typescript
// ❌ AI omits types
const users = []; // Error: implicit any[]

// ✅ Strict mode requires
const users: User[] = [];
```

### Error 3: Unsafe Object Access

```typescript
// ❌ AI doesn't check existence
function process(config: Record<string, any>) {
  return config.database.host; // Error: might not exist
}

// ✅ Strict mode forces proper checking
function process(config: Record<string, any>) {
  return config?.database?.host;
}
```

### Error 4: Unhandled Promises

```typescript
// ❌ AI forgets async
function loadUser(id: string): Promise<User> {
  return fetchUser(id); // Must have return or await
}

// ✅ Strict mode catches
async function loadUser(id: string): Promise<User> {
  const response = await fetchUser(id);
  return response.data;
}
```

## Best Practices with Strict Mode

### Practice 1: Use Branded Types for Domain Validation

```typescript
// Prevent passing wrong IDs
type UserId = string & { readonly brand: 'UserId' };
type OrderId = string & { readonly brand: 'OrderId' };

function createBrandedId<T>(id: string): T {
  return id as T;
}

function getUser(id: UserId): Promise<User> {
  return api.get(`/users/${id}`);
}

// Error: OrderId is not UserId
const order = getOrder(createBrandedId<UserId>('123'));
```

### Practice 2: Use const Assertions for Literals

```typescript
// Prevents accidentally allowing other values
const Status = {
  PENDING: 'pending',
  COMPLETED: 'completed',
  FAILED: 'failed'
} as const;

type StatusType = typeof Status[keyof typeof Status];

function updateStatus(status: StatusType) {
  // Only allows 'pending', 'completed', or 'failed'
}
```

### Practice 3: Discriminated Unions for Type Safety

```typescript
type Result<T> =
  | { success: true; data: T }
  | { success: false; error: string };

function handle<T>(result: Result<T>) {
  if (result.success) {
    console.log(result.data); // Type is T
  } else {
    console.log(result.error); // Type is string
  }
}
```

### Practice 4: Strict Function Signatures

```typescript
// Not just parameter types, but return types
function fetchUser(id: string): Promise<User> {
  // Must return Promise<User>, not Promise<any>
  return api.get(`/users/${id}`);
}

// Arrow functions with explicit return types
const calculateTotal = (items: OrderItem[]): number => {
  return items.reduce((sum, item) => sum + item.price, 0);
};
```

## Gradual Migration to Strict Mode

If enabling strict mode all at once is too disruptive:

### Step 1: Enable in tsconfig

```json
{
  "compilerOptions": {
    "strict": true,
    "skipLibCheck": true
  }
}
```

### Step 2: Fix errors file by file

1. Start with most critical files (domain logic, services)
2. Use `// @ts-ignore` sparingly for temporary debt
3. Create issues for remaining strict violations
4. Enforce strict mode in CI/CD for new code

### Step 3: Enforce in CI/CD

```bash
# Fail CI if TypeScript has errors
npx tsc --noEmit --skipLibCheck false
```

## TypeScript Strict Mode Checklist for AI-Generated Code

When reviewing AI-generated code:

- ✅ All variables have explicit types (no implicit `any`)
- ✅ All nullable values use `Type | null` or `Type | undefined`
- ✅ All optional properties marked with `?`
- ✅ Optional chaining (`?.`) used where appropriate
- ✅ Nullish coalescing (`??`) used for defaults
- ✅ All functions have return types
- ✅ All code paths in functions return a value
- ✅ All class properties initialized or optional
- ✅ Switch statements have break/return in all cases
- ✅ No implicit `this` usage
- ✅ No unchecked array/object access
- ✅ Promises properly awaited or returned

## Testing Strict Mode Compliance

```bash
# Check TypeScript compilation without emitting
npx tsc --noEmit

# Check specific file
npx tsc src/services/UserService.ts --noEmit

# Get error count
npx tsc --noEmit 2>&1 | grep -c "error TS"
```

## Why Strict Mode is Essential for AI

1. **Catches Logical Errors:** AI can write type-correct but logically wrong code; strict mode forces thinking about nullability and edge cases
2. **Prevents Silent Failures:** Many AI errors only manifest at runtime in permissive mode
3. **Forces Explicit Contracts:** Strict mode makes dependencies and assumptions visible in types
4. **Automatic Enforcement:** Unlike linters, TypeScript strict mode is enforced by the compiler—AI can't bypass it

## Related Topics

- TypeScript Strict Mode Configuration
- Type Safety Patterns
- Testing Strategy (Layer 2 filter)
- Code Review Guidelines (Layer 3 filter)

## References & Sources

- TypeScript Strict Mode: https://www.typescriptlang.org/tsconfig#strict
- TypeScript Handbook - Type Checking: https://www.typescriptlang.org/docs/handbook/type-checking-javascript-files.html
- "Effective TypeScript" - Dan Vanderkam
- Type-Level Programming in TypeScript - https://www.typescriptlang.org/docs/handbook/2/types-from-types.html
