# Lifecycle

## After 16.3

Mounting: The component is ready to mount in the browser DOM. This phase covers initialization.

- constructor()
- getDerivedStateFromProps()
- render()
- **componentDidMount()** // AJAX requests, DOM or state updates, and set up event listeners should occur (componentWillMount() before 16.3)

Updating: In this phase, the component gets updated in two ways, sending the new props and updating the state either from setState() or forceUpdate(). This phase covers

- getDerivedStateFromProps()
- shouldComponentUpdate()
- render()
- getSnapshotBeforeUpdate()
- **componentDidUpdate()** // capture events

Unmounting: In this last phase, the component is not needed and gets unmounted from the browser DOM. This phase includes

- **componentWillUnmount()** // cancel any outgoing network requests, or remove all event listeners
