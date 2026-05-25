# Airbnb TypeScript Style Rules (Detailed)

TypeScript-specific rules that extend the core Airbnb JS style guide.
All core JS rules apply. These rules add or clarify TypeScript patterns.

---

## Type Declarations

```ts
// Use interface for object shapes
// Bad
type User = {
  name: string;
  age: number;
};
// Good
interface User {
  name: string;
  age: number;
}

// Use type for unions, intersections, and aliases
type ID = string | number;
type Result<T> = { data: T; error: null } | { data: null; error: Error };
type Callback = () => void;

// Extend with extends (interface) or & (type)
interface AdminUser extends User {
  role: 'admin';
}
type AdminOrUser = AdminUser & { lastLogin: Date };
```

---

## Never Use `any`

```ts
// Bad — loses all type safety
function parse(value: any): any {
  return JSON.parse(value);
}

// Bad — implicit any
function process(data) { ... }

// Good — use unknown when type is truly unknown
function parse(value: unknown): unknown {
  if (typeof value === 'string') {
    return JSON.parse(value);
  }
  throw new Error('Expected string');
}

// Good — use generics
function identity<T>(value: T): T {
  return value;
}
```

---

## Explicit Return Types on Exported Functions

```ts
// Bad — unclear what the function returns
export function getUser(id: string) {
  return db.users.findById(id);
}

// Good
export function getUser(id: string): Promise<User | null> {
  return db.users.findById(id);
}

// Internal/private helpers: inferred types are acceptable
const add = (a: number, b: number) => a + b;
```

---

## `readonly` for Immutable Properties

```ts
// Bad
interface Config {
  host: string;
  port: number;
}

// Good — prevents accidental mutation
interface Config {
  readonly host: string;
  readonly port: number;
}

// Arrays
function processItems(items: readonly string[]): void {
  // items.push('x'); // error — good
}
```

---

## Prefer `?.` and `??`

```ts
// Bad
const name = user && user.profile && user.profile.name;
const count = value !== null && value !== undefined ? value : 0;

// Good
const name = user?.profile?.name;
const count = value ?? 0;
```

---

## Enums

```ts
// Avoid numeric enums — they allow arbitrary number assignment
// Bad
enum Direction {
  Up,     // 0
  Down,   // 1
}

// Good — string enums are explicit
enum Direction {
  Up = 'UP',
  Down = 'DOWN',
}

// Also Good — string union type for simple cases
type Direction = 'UP' | 'DOWN';
```

---

## Generics

```ts
// Descriptive generic names (not just T unless context is obvious)
// Bad
function merge<T, U>(a: T, b: U): T & U { ... }

// Good for simple cases
function identity<T>(value: T): T { return value; }

// Good with descriptive names for complex cases
function mapValues<TInput, TOutput>(
  obj: Record<string, TInput>,
  fn: (value: TInput) => TOutput,
): Record<string, TOutput> { ... }

// Constrain generics when appropriate
function getProperty<TObj, TKey extends keyof TObj>(
  obj: TObj,
  key: TKey,
): TObj[TKey] {
  return obj[key];
}
```

---

## Type Assertions

```ts
// Prefer type predicates over assertions
// Bad
const input = document.getElementById('input') as HTMLInputElement;

// Good — when you've already validated
const element = document.getElementById('input');
if (element instanceof HTMLInputElement) {
  element.value = 'hello'; // element is HTMLInputElement here
}

// Use non-null assertion only when you're certain and can't check
// Bad — overused
const element = document.getElementById('input')!;

// Acceptable — when non-null is guaranteed by app logic
// (document.getElementById('root') is always present in SPA)
const root = document.getElementById('root')!;

// Prefer double assertion A → unknown → B over direct A → B
const foo = (bar as unknown) as Foo;
```

---

## Interfaces vs Classes for Data

```ts
// Prefer interfaces for pure data shapes
// Bad — class with no methods
class UserData {
  constructor(
    public name: string,
    public email: string,
  ) {}
}

// Good
interface UserData {
  name: string;
  email: string;
}

// Use classes when you need methods, inheritance, or instantiation
class UserService {
  private users: UserData[] = [];
  add(user: UserData): void { this.users.push(user); }
}
```

---

## Avoid Overloads When Union Types Work

```ts
// Bad
function format(value: string): string;
function format(value: number): string;
function format(value: string | number): string {
  return String(value);
}

// Good
function format(value: string | number): string {
  return String(value);
}
```

---

## Null vs Undefined

```ts
// Be consistent — prefer undefined over null for optional values
// null = intentionally absent
// undefined = not yet set / optional

interface User {
  name: string;
  middleName?: string;      // undefined — optional
  deletedAt: Date | null;   // null — intentionally absent
}

// Don't mix null and undefined for optional returns
// Bad
function find(id: string): User | null | undefined { ... }
// Good
function find(id: string): User | undefined { ... }
```

---

## File and Naming Conventions

```
// TypeScript source files
src/
  components/
    UserCard.tsx       // PascalCase for React components
    UserCard.test.tsx
  utils/
    formatDate.ts      // camelCase for utility modules
    formatDate.test.ts
  types/
    user.types.ts      // kebab-case or camelCase for type files
  constants/
    apiEndpoints.ts

// Interface names — no I prefix
// Bad
interface IUser { ... }
// Good
interface User { ... }

// Type names — no Type suffix
// Bad
type UserType = { ... }
// Good
type User = { ... }
```

---

## tsconfig Recommendations (Airbnb-aligned)

```json
{
  "compilerOptions": {
    "strict": true,
    "noImplicitAny": true,
    "strictNullChecks": true,
    "noImplicitReturns": true,
    "noUncheckedIndexedAccess": true,
    "exactOptionalPropertyTypes": true
  }
}
```

Enable all strict checks. Do not disable `noImplicitAny` or `strictNullChecks`.
