# switchMap

```ts
this.searchField.valueChanges
  .debounceTime(400)
    .switchMap(searchString => this.searchService.search(searchString))
    .subscribe((result) => {
        this.result = result.artists.items
    });
```
