# map

## Use case

Return changed result.

## Example

```ts
const http$ : Observable<Course[]> = this.http.get('/api/courses');

http$
  .pipe(
    tap(() => console.log('HTTP request executed')),
    map(res => Object.values(res['payload']))
  )
  .subscribe(
    courses => console.log("courses", courses)
  );
```
