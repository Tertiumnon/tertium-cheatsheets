# mergeMap

When we want to access the value of one subscribed observable into another, we can use the mergeMap to pass it and get the a single cohesive stream we can use to control events.

```ts
this.homeworld = this.http
  .get('/api/people/1')
  .pipe(mergeMap(character => this.http.get(character.homeworld)));
```
