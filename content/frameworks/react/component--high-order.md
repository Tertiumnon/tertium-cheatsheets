# Higher-Order Components

A higher-order component (HOC) is a function that takes a component and returns a new component.

```js
const EnhancedComponent = higherOrderComponent(WrappedComponent);
```

HOC can be used for many use cases:

i. Code reuse, logic and bootstrap abstraction.
ii. Render hijacking.
ii. State abstraction and manipulation.
iii. Props manipulation.

```js
function HOC(WrappedComponent) {
  return class Test extends Component {
    render() {
      const newProps = {
        title: "New Header",
        footer: false,
        showFeatureX: false,
        showFeatureY: true,
      };

      return <WrappedComponent {...this.props} {...newProps} />;
    }
  };
}
```
