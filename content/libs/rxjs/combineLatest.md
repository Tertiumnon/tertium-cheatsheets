# combineLatest

You can also use combineLatest instead of forkJoin, which instead of waiting for all observables to complete, will emit everytime one of the observables changes (but it will wait for all observables to emit at least one value first).

```ts
combineLatest([
          this.httpHandlerCached.getListsA(),
          this.httpHandlerCached.getListsB(),
          this.httpHandlerCached.getListsC(),
          this.route.params
        ]).subscribe(res => {
            this.listA = res[0];
            this.listB = res[1];
            this.listC = res[2];
            this.doSomethingWithFetchedLists();
          });
```
