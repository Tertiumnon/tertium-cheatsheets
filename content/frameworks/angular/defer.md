# @defer

Deferrable views, also known as @defer blocks, reduce the initial bundle size of your application by deferring the loading of code that is not strictly necessary for the initial rendering of a page.

```ts
@defer {
  <large-component />
} @loading {
  <img alt="loading..." src="loading.gif" />
} @placeholder {
  <p>Placeholder content</p>
}
```

With triggers:

- idle
- interaction
- viewport
- hover
- timer
