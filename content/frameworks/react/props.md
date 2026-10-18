# Props

## Key-prop

A key is a special attribute you should include when mapping over arrays to render data. Key prop helps React identify which items have changed, are added, or are removed.

```jsx
const todoItems = todos.map((todo) => <li key={todo.id}>{todo.text}</li>);
```