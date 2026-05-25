# Airbnb JSX / React Style Rules (Detailed)

Source: https://github.com/airbnb/javascript/tree/master/react

---

## Basic Rules

- **One React component per file.** Multiple stateless/functional components are allowed in one file.
- **Always use JSX syntax.** Avoid `React.createElement` unless initializing app from a non-JSX file.
- Use `React.Fragment` or `<>...</>` instead of wrapper `<div>` when no DOM element is needed.

---

## Component Type

- Prefer functional components with hooks over class components for all new code.
- Use `class extends React.Component` if you need error boundaries (no hooks equivalent).

```jsx
// Bad — class component where function works
class Listing extends React.Component {
  render() {
    return <div>{this.props.hello}</div>;
  }
}

// Good
function Listing({ hello }) {
  return <div>{hello}</div>;
}
```

---

## Naming

```jsx
// File extension: .jsx for React components, .tsx for TypeScript components
// File name: PascalCase — ReservationCard.jsx

// Component reference: PascalCase
import ReservationCard from './ReservationCard';

// Instance: camelCase
const reservationItem = <ReservationCard />;

// Use directory name when index file
// components/CheckBox/index.jsx
export default function CheckBox() { ... }

// HOC displayName
// Bad
export default function withFoo(WrappedComponent) {
  return function WithFoo(props) { ... };
}
// Good
export default function withFoo(WrappedComponent) {
  function WithFoo(props) { ... }
  WithFoo.displayName = `WithFoo(${getDisplayName(WrappedComponent)})`;
  return WithFoo;
}

// Don't repurpose DOM prop names
// Bad
<Component style="fancy" />
<Component className={styles.foo} />  // fine — this is JSX, not HTML
// Bad (semantic clash)
<MyComponent style="primary" />     // 'style' means inline styles in DOM
// Good
<MyComponent variant="primary" />
```

---

## Quotes

```jsx
// Double quotes for JSX attributes
<Foo bar="bar" />
<Foo style={{ left: '20px' }} />   // single quotes inside JS expression

// Single quotes for all other JS
const foo = 'bar';
```

---

## Spacing

```jsx
// Single space before self-closing />
// Bad
<Foo/>
<Foo                 />
// Good
<Foo />

// No padding in JSX curly braces
// Bad
<Foo bar={ baz } />
// Good
<Foo bar={baz} />
```

---

## Props

```jsx
// camelCase prop names
// Bad
<Foo UserName="hello" phone_number={12345} />
// Good
<Foo userName="hello" phoneNumber={12345} />

// PascalCase if prop value is a React component
<MyComponent InputComponent={InputField} />

// Omit value when explicitly true
// Bad
<Foo hidden={true} />
// Good
<Foo hidden />

// Always include alt on <img>
// Bad
<img src="foo.jpg" />
// Good
<img src="foo.jpg" alt="A photo of a horse" />
// No "image" or "photo" in alt text
// Bad
<img alt="Photo of horse" src="horse.jpg" />
// Good
<img alt="A brown horse in a field" src="horse.jpg" />

// Valid, non-abstract ARIA roles
// Bad
<div role="datepicker" />
<div role="range" />
// Good
<div role="button" />

// No accessKey
// Bad
<div accessKey="h" />

// Stable keys — no array index
// Bad
{todos.map((todo, index) => <Todo key={index} />)}
// Good
{todos.map((todo) => <Todo key={todo.id} />)}

// Define defaultProps for all non-required props
function Foo({ name, age }) {
  return <div>{name}: {age}</div>;
}
Foo.defaultProps = {
  age: 0,
};
Foo.propTypes = {
  name: PropTypes.string.isRequired,
  age: PropTypes.number,
};

// Use spread sparingly — avoid passing unknown props to DOM
// Bad
function App({ irrelevant, ...rest }) {
  return <Component {...rest} />;  // may pass irrelevant props
}
// Good — explicitly pass what you need
function App({ className, style }) {
  return <Component className={className} style={style} />;
}
```

---

## Refs

```jsx
// Use createRef or callback ref. No string refs.
// Bad
<Foo ref="myRef" />

// Good (React.createRef)
class MyComponent extends React.Component {
  constructor(props) {
    super(props);
    this.myRef = React.createRef();
  }
  render() { return <div ref={this.myRef} />; }
}

// Good (callback ref)
<Foo ref={(ref) => { this.myRef = ref; }} />

// Good (useRef hook — preferred in function components)
function MyComponent() {
  const myRef = React.useRef(null);
  return <div ref={myRef} />;
}
```

---

## Parentheses

```jsx
// Wrap multi-line JSX in parentheses
// Bad
render() {
  return <MyComponent className="long className"
           foo="bar">
           <MyChild />
         </MyComponent>;
}

// Good
render() {
  return (
    <MyComponent
      className="long className"
      foo="bar"
    >
      <MyChild />
    </MyComponent>
  );
}
```

---

## Tags

```jsx
// Self-close tags without children
// Bad
<Foo className="stuff"></Foo>
// Good
<Foo className="stuff" />

// Multi-line: closing bracket on new line
// Bad
<Foo bar="bar"
     baz="baz" />
// Good
<Foo
  bar="bar"
  baz="baz"
/>
```

---

## Methods

```jsx
// Arrow functions for passing data to event handlers in render
class ItemList extends React.Component {
  render() {
    return (
      <ul>
        {this.props.items.map((item, index) => (
          <Item
            key={item.key}
            onClick={(e) => this.handleClick(e, index)}
          />
        ))}
      </ul>
    );
  }
}

// Bind event handlers in constructor, not render
// Bad
class Foo extends React.Component {
  render() {
    return <button onClick={this.handleClick.bind(this)}>Click</button>;
  }
}
// Good
class Foo extends React.Component {
  constructor(props) {
    super(props);
    this.handleClick = this.handleClick.bind(this);
  }
  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}
// Also Good — class property arrow function
class Foo extends React.Component {
  handleClick = () => { ... };
  render() {
    return <button onClick={this.handleClick}>Click</button>;
  }
}

// No underscore prefix for internal methods
// Bad
_handleClick() { ... }
// Good
handleClick() { ... }

// Always return from render
// Bad
render() {
  (<div />);  // missing return
}
// Good
render() {
  return (<div />);
}
```

---

## Ordering for Class Components

1. `static` methods and properties
2. `constructor`
3. `getChildContext`
4. `static getDerivedStateFromProps`
5. `componentDidMount`
6. `shouldComponentUpdate`
7. `getSnapshotBeforeUpdate`
8. `componentDidUpdate`
9. `componentWillUnmount`
10. Click handlers or event handlers: `handleClick`, `handleSubmit`
11. Getter methods: `getStyle`, `getValue`
12. Optional render methods: `renderNavigation`, `renderProfilePicture`
13. `render`

Place `propTypes`, `defaultProps`, `contextTypes`, `childContextTypes` outside the class, then assign to component after definition:

```jsx
function Foo({ name }) {
  return <div>{name}</div>;
}
Foo.propTypes = {
  name: PropTypes.string.isRequired,
};
Foo.defaultProps = {};
```
