# React Elements

An Element is a plain object describing what you want to appear on the screen in terms of the DOM nodes or other components.

React-way to create elements:

```jsx
const Button = ({ handleLogin }) => (
  <div id={"login-btn"} onClick={handleLogin}>
    Login
  </div>
);
```

Transpiled function (JS-representation):

```jsx
const Button = ({ handleLogin }) =>
  React.createElement(
    "div",
    { id: "login-btn", onClick: handleLogin },
    "Login"
  );
```
