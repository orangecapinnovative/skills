# Airbnb JS Style Guide — Core Rules (Detailed)

Detailed reference for every rule category. Read this when you need the full rationale and extended examples.

---

## 1. Types

Primitives are accessed by value; complex types by reference.

```js
// Primitives: string, number, bigint, boolean, undefined, null, symbol
const foo = 1;
let bar = foo;
bar = 9;
// foo === 1, bar === 9

// Complex: object, array, function
const foo = [1, 2];
const bar = foo;
bar[0] = 9;
// foo[0] === 9 — same reference
```

---

## 2. References

```js
// Bad
var a = 1;
var b = 2;

// Good
const a = 1;
const b = 2;

// Only use let when reassignment is needed
let count = 1;
count += 1;
```

---

## 3. Objects

```js
// Bad
const item = new Object();

// Good
const item = {};

// Dynamic keys — use computed property names
function getKey(k) { return `a key named ${k}`; }
const obj = {
  id: 5,
  name: 'San Francisco',
  [getKey('enabled')]: true,
};

// Method shorthand
// Bad
const atom = {
  value: 1,
  addValue: function(v) { return atom.value + v; },
};
// Good
const atom = {
  value: 1,
  addValue(v) { return atom.value + v; },
};

// Property shorthand
const name = 'John';
const age = 30;
// Bad
const person = { name: name, age: age };
// Good
const person = { name, age };

// Group shorthand at top
const obj = {
  name,      // shorthand first
  age,
  foo: 'bar',
  baz: 42,
};

// Only quote invalid identifiers
// Bad
const obj = { 'foo': 1, 'bar-baz': 2 };
// Good
const obj = { foo: 1, 'bar-baz': 2 };

// Prefer spread over Object.assign
// Bad
const original = { a: 1 };
const copy = Object.assign({}, original, { c: 3 });
// Good
const copy = { ...original, c: 3 };

// Get subset of object
const { a, b, ...rest } = fullObject;
```

---

## 4. Arrays

```js
// Bad
const items = new Array();

// Good
const items = [];
items.push('something');

// Copy with spread
const copy = [...items];

// Convert iterable to array
const arr = Array.from(new Set([1, 2, 3]));
// or
const arr = [...new Set([1, 2, 3])];

// Use Array.from for mapping (avoids spread intermediate)
const mapped = Array.from([1, 2, 3], (x) => x * 2);

// Array method callbacks must always return
[1, 2, 3].map((x) => {
  const doubled = x * 2;
  return doubled; // required
});
```

---

## 5. Destructuring

```js
// Objects
// Bad
function getFullName(user) {
  const firstName = user.firstName;
  const lastName = user.lastName;
  return `${firstName} ${lastName}`;
}
// Good
function getFullName({ firstName, lastName }) {
  return `${firstName} ${lastName}`;
}

// Arrays
const arr = [1, 2, 3, 4];
const [first, second] = arr;

// Multiple return values — use object destructuring
// Bad
function processInput() { return [left, right, top, bottom]; }
const [l, r, t, b] = processInput(); // order-dependent

// Good
function processInput() { return { left, right, top, bottom }; }
const { left, right } = processInput(); // order-independent
```

---

## 6. Strings

```js
// Single quotes
const name = 'Airbnb';

// Template literals for interpolation
const greeting = `How are you, ${name}?`;

// Lines over 100 chars — don't break with concatenation; use template literal
const errorMessage = `This is a super long error that was thrown because
of Batman. When you stop to think about how Batman had anything to do
with this, you would get nowhere fast.`;

// Never eval()
// Bad
eval('console.log("hello")');
```

---

## 7. Functions

```js
// Named function expression, not declaration
// Bad — hoisting can cause surprises
function foo() { ... }
// Good
const foo = function foo() { ... };

// IIFE
(function () { console.log('IIFE'); }());

// Never declare functions in blocks (if, while, etc.)
// Bad
if (x) { function test() { return true; } }
// Good
let test;
if (x) { test = () => true; }

// Rest over arguments
// Bad
function concat() {
  const args = Array.prototype.slice.call(arguments);
  return args.join('');
}
// Good
function concat(...args) {
  return args.join('');
}

// Default parameters
// Bad
function handleThings(opts) { opts = opts || {}; }
// Good
function handleThings(opts = {}) { ... }

// Default params last
// Bad
function f(opts = {}, name) { ... }
// Good
function f(name, opts = {}) { ... }

// Never mutate or reassign parameters
// Bad
function foo(obj) { obj.key = 1; }
function bar(a) { a = 1; }
// Good
function foo(obj) { const key = Object.prototype.hasOwnProperty.call(obj, 'key') ? obj.key : 1; }
function bar(a) { const b = a || 1; }

// Space before function body
// Bad
const f = function() {};
// Good
const f = function named() {};
const g = async () => {};

// Never use Function constructor
// Bad
const add = new Function('a', 'b', 'return a + b');
```

---

## 8. Arrow Functions

```js
// Use for callbacks
// Bad
[1, 2, 3].map(function (x) { return x ** 2; });
// Good
[1, 2, 3].map((x) => x ** 2);

// Implicit return for single expression (no braces)
[1, 2, 3].map((x) => x * x);

// Explicit return for multiline
[1, 2, 3].map((x) => {
  const y = x + 1;
  return x * y;
});

// Wrap object literal in parens
[1, 2, 3].map((x) => ({ id: x }));

// Always include parens around parameters
// Bad
[1, 2, 3].map(x => x * x);
// Good
[1, 2, 3].map((x) => x * x);

// Avoid confusing arrow with comparisons
// Bad
const isLegal = (age) => age >= 18;  // this is fine actually
// Confusing: const foo = a <= b ? a : b; — use parens
const foo = (a <= b) ? a : b;
```

---

## 9. Classes & Constructors

```js
// Use class syntax
// Bad
function Queue(contents = []) {
  this._queue = [...contents];
}
Queue.prototype.pop = function () { ... };

// Good
class Queue {
  constructor(contents = []) {
    this.queue = [...contents];
  }
  pop() { ... }
}

// Inheritance with extends
class PriorityQueue extends Queue {
  constructor(contents = []) {
    super(contents);
    this.offset = 0;
  }
  pop() {
    const value = super.pop();
    return value;
  }
}

// Method chaining — return this
class Jedi {
  jump() { this.jumping = true; return this; }
  setHeight(height) { this.height = height; return this; }
}
const jedi = new Jedi().jump().setHeight(20);

// No empty constructors
// Bad
class Jedi { constructor() {} }
class Rey extends Jedi { constructor(...args) { super(...args); } }

// No duplicate class members
// Bad
class Foo { bar() { return 1; } bar() { return 2; } }

// Class methods should use this or be made static
// Bad
class Foo { baz() { return 42; } }  // never uses this
// Good
class Foo { static baz() { return 42; } }
```

---

## 10. Modules

```js
// ES modules only
// Bad
const AirbnbStyleGuide = require('./AirbnbStyleGuide');
module.exports = AirbnbStyleGuide.es6;

// Good
import AirbnbStyleGuide from './AirbnbStyleGuide';
export default AirbnbStyleGuide.es6;

// No wildcard imports
// Bad
import * as AirbnbStyleGuide from './AirbnbStyleGuide';

// Single import per path
// Bad
import foo from 'foo';
import { bar, baz } from 'foo';
// Good
import foo, { bar, baz } from 'foo';

// Don't export mutable bindings
// Bad
let count = 0;
export { count };

// Imports above non-imports
// Bad
import foo from 'foo';
foo.init();
import bar from 'bar';
// Good
import foo from 'foo';
import bar from 'bar';
foo.init();

// No default + named from same re-export
// Bad
export { foo as default } from './foo';
```

---

## 11. Iterators & Generators

```js
// Prefer functional iteration
// Bad
const increasedByOne = [];
for (let i = 0; i < numbers.length; i++) {
  increasedByOne.push(numbers[i] + 1);
}
// Good
const increasedByOne = numbers.map((num) => num + 1);

// Use reduce for accumulation
const sum = [1, 2, 3].reduce((acc, val) => acc + val, 0);

// Avoid generators (transpilation issues)
// If you must, space them properly:
function* myGenerator() { yield 1; }
```

---

## 12. Properties

```js
const obj = { foo: true, bar: 42, baz: 'hello' };

// Dot notation
const foo = obj.foo;

// Bracket when using a variable
const key = 'bar';
const val = obj[key];

// Exponentiation operator
const binary = 2 ** 10;
```

---

## 13. Variables

```js
// One declaration per variable
// Bad
const a = 1, b = 2, c = 3;
// Good
const a = 1;
const b = 2;
const c = 3;

// Group const then let
const a = 1;
const b = 2;
let c;
let d;

// Assign where needed (not all at top like C)
function foo() {
  const a = getA();
  // ... lots of code ...
  const b = getB();  // assigned near use
}

// No chained assignment
// Bad
const a = b = c = 1;
// Good
const a = 1;
let b = 1;
let c = 1;

// No unary ++ / --
// Bad
for (let i = 0; i < 10; i++) { }
// Good
for (let i = 0; i < 10; i += 1) { }

// No linebreak around =
// Bad
const foo =
  longVariableName;
// Good
const foo = longVariableName;
// or
const foo = (
  longVariableName
);
```

---

## 14. Comparison Operators & Equality

```js
// Always ===
if (a === b) { }
if (a !== b) { }

// Shorthand for booleans
// Bad
if (isValid === true) { }
if (collection.length > 0) { }
// Good
if (isValid) { }
if (collection.length) { }

// Explicit for strings and numbers
// Bad
if (name) { }   // truthy — but name could be '0', 0, etc.
// Good
if (name !== '') { }
if (count > 0) { }

// switch — use braces for lexical declarations
switch (foo) {
  case 1: {
    const x = 1;
    break;
  }
  default:
    break;
}

// No nested ternaries
// Bad
const foo = maybe1 > maybe2 ? 'bar' : value1 > value2 ? 'baz' : null;
// Good
const maybeNull = value1 > value2 ? 'baz' : null;
const foo = maybe1 > maybe2 ? 'bar' : maybeNull;

// Avoid unneeded ternary
// Bad
const foo = a ? a : b;
const bar = c ? true : false;
// Good
const foo = a || b;
const bar = !!c;

// Parentheses for mixed operators
const foo = (a && b) || c || (d && e);
```

---

## 15. Blocks

```js
// Braces for multiline
// Bad
if (test) return false;
// Good
if (test) {
  return false;
}

// else on same line as }
// Bad
if (test) {
  thing1();
}
else {
  thing2();
}
// Good
if (test) {
  thing1();
} else {
  thing2();
}

// No else after return
// Bad
function foo() {
  if (x) { return x; }
  else { return y; }
}
// Good
function foo() {
  if (x) { return x; }
  return y;
}
```

---

## 16. Comments

```js
/** Multiline JSDoc */
/**
 * Make() returns a new element based on the passed-in tag name
 */
function make(tag) { ... }

// Single-line: above the subject, not end of line
// Bad
const active = true; // is current tab

// Good
// is current tab
const active = true;

// Start with a space
// Bad
//is active
// Good
// is active

// Annotations
// TODO: implement cleanup
// FIXME: causes crash when value is null
```

---

## 17. Whitespace

```js
// 2-space indentation
function foo() {
  const bar = 1;
}

// Space before {
// Bad
function test(){return true;}
if(x){foo();}
// Good
function test() { return true; }
if (x) { foo(); }

// Space after keywords
if (x) { }
while (y) { }
for (let i = 0; i < 10; i += 1) { }

// No space before function call parens
// Bad
foo ();
// Good
foo();

// Operators
// Bad
const x=1+2;
// Good
const x = 1 + 2;

// End of file: single newline
// No multiple blank lines
// No blank lines at start/end of blocks

// Method chaining: leading dot, indented
$('#items')
  .find('.selected')
  .highlight()
  .end();

// Space after blocks before next statement
const obj = { foo: 1 };
const result = bar();  // blank line between if {} and this

// Spaces inside { } for objects and imports
import { foo, bar } from 'module';
const obj = { key: 'value' };

// No spaces inside () or []
foo(bar);
const arr = [1, 2, 3];
```

---

## 18. Commas

```js
// No leading commas
// Bad
const story = [
    once
  , upon
  , aTime
];

// Trailing commas (multi-line only)
// Good
const story = [
  once,
  upon,
  aTime,
];

const hero = {
  firstName: 'Ada',
  lastName: 'Lovelace',
  birthYear: 1815,
  superPower: 'computers',
};
```

---

## 19. Semicolons

```js
// Always use semicolons
const luke = {};
const leia = {};
[luke, leia].forEach((jedi) => { jedi.father = 'vader'; });

// Without semicolons, the above would be read as:
// const luke = {}; const leia = {}[luke, leia].forEach(...)
// which throws a TypeError
```

---

## 20. Type Casting & Coercion

```js
// Strings
// Bad
const totalScore = new String(value);
const totalScore = '' + value;
// Good
const totalScore = String(value);
const totalScore = `${value}`;

// Numbers
// Bad
const val = new Number(str);
// Good
const val = Number(str);
const val = parseInt(str, 10);  // always include radix

// Booleans
// Bad
const hasAge = new Boolean(age);
// Good
const hasAge = Boolean(age);
const hasAge = !!age;
```

---

## 21. Naming Conventions

```js
// Descriptive names — no single letters except loop counters
// Bad
function q() {}
// Good
function query() {}

// camelCase for variables, functions, instances
const thisIsMyObject = {};
function thisIsMyFunction() {}
const myInstance = new MyClass();

// PascalCase for classes and constructors
class UserAccount {}
const account = new UserAccount();

// No leading/trailing underscores
// Bad
const _privateField = 1;
this.__super__ = 'test';
// Good: use WeakMap or Symbol for "private"
const _private = new WeakMap();

// No saving this — use arrow functions
// Bad
const self = this;
function foo() { self.doSomething(); }
// Good
const foo = () => { this.doSomething(); };

// Filename matches default export
// File: CheckBox.js
export default function CheckBox() { ... }

// camelCase when exporting default functions
// File: makeStyleGuide.js
export default function makeStyleGuide() { ... }

// PascalCase when exporting class or constructor
// File: StyleGuide.js
export default class StyleGuide { ... }

// SCREAMING_SNAKE_CASE for module-level constants
const MAPPING = {};
```

---

## 22. Accessors

Accessor functions (getters/setters) are not required. If you use them:

```js
// Bad
dragon.age();         // is this a getter or method?
dragon.age(25);       // is this a setter or method?

// Good — be consistent
dragon.getAge();
dragon.setAge(25);

// For boolean, use isVal / hasVal
// Bad
if (!dragon.age()) { ... }
// Good
if (!dragon.isAdult()) { ... }
```

---

## 23. Events

```js
// Pass object with data, not raw value — allows adding fields later
// Bad
$(this).trigger('listingUpdated', listing.id);
$(this).on('listingUpdated', (e, listingId) => { ... });

// Good
$(this).trigger('listingUpdated', { listingId: listing.id });
$(this).on('listingUpdated', (e, { listingId }) => { ... });
```
