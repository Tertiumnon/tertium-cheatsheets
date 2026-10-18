# forkJoin

```ts
forkJoin([
      this.httpHandlerCached.getListsA(),
      this.httpHandlerCached.getListsB(),
      this.httpHandlerCached.getListsC(),
      this.route.params.pipe(take(1))
    ]).subscribe(res => {
        this.listA = res[0];
        this.listB = res[1];
        this.listC = res[2];
        this.doSomethingWithFetchedLists();
      });
```
