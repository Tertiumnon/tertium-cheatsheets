# Zone.js

Change detection (rendering) in Angular is usually triggered completely automatically as a result of async events in a browser. This is made possible by utilizing zones implemented by zone.js library.

In general, zones provide a mechanism to intercept the scheduling and calling of asynchronous operations. Interceptor logic can execute additional code before or after the task and notify interesting parties about the event. These rules are defined individually for each zone when it’s being created.

To enable automatic change detection, Angular implements NgZone service that forks a child zone and subscribes to notifications from this zone.

Each time a component is created, Angular creates a new zone for that component. The zone tracks any changes that occur within the component and triggers the change detection mechanism when necessary.

Angular provides the NgZone service, which allows us to run code inside or outside of the Angular zone.
