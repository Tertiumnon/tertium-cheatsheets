# Redux - React Library

Redux is a predictable state container for JavaScript apps based on the Flux design pattern.

Redux follows three fundamental principles:

- **Single source of truth**: The state of your whole application is stored in an object tree within a single store. The single state tree makes it easier to keep track of changes over time and debug or inspect the application.

- **State is read-only**: The only way to change the state is to emit an action, an object describing what happened. This ensures that neither the views nor the network callbacks will ever write directly to the state.

- **Changes are made with pure functions**: To specify how the state tree is transformed by actions, you write reducers. Reducers are just pure functions that take the previous state and an action as parameters, and return the next state.

## Concepts

### Flux

Flux is an application design paradigm used as a replacement for the more traditional MVC pattern. It is not a framework or a library but a new kind of architecture that complements React and the concept of Unidirectional Data Flow. Facebook uses this pattern internally when working with React.

The workflow between dispatcher, stores and views components with distinct inputs and outputs as follows:

![flux](flux.png)

### Props

*mapStateToProps()* is a utility which helps your component get updated state (which is updated by some other components):

```js
const mapStateToProps = (state) => {
  return {
    todos: getVisibleTodos(state.todos, state.visibilityFilter),
  };
};
```

*mapDispatchToProps()* is a utility which will help your component to fire an action event (dispatching action which may cause change of application state):

```js
const mapDispatchToProps = (dispatch) => {
  return {
    onTodoClick: (id) => {
      dispatch(toggleTodo(id));
    },
  };
};
```
