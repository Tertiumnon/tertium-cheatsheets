# Lifecycle

Angular component lifecycle hooks that allow you to tap into key events during the component lifecycle.

## constructor

Called when the class is instantiated, before any Angular-specific functionality. Use for dependency injection.

```typescript
constructor(private service: MyService) { }
```

## ngOnChanges

Called before `ngOnInit` and whenever input properties change. Receives `SimpleChanges` object.

```typescript
ngOnChanges(changes: SimpleChanges) {
  if (changes['count']) {
    console.log('Count changed:', changes['count'].currentValue);
  }
}
```

## ngOnInit

Called once after first `ngOnChanges`. Use for initialization logic and fetching data.

```typescript
ngOnInit() {
  this.data = this.service.getData();
}
```

## ngDoCheck

Called during every change detection run. Use for custom change detection logic (expensive).

```typescript
ngDoCheck() {
  console.log('Change detection ran');
}
```

## ngAfterContentInit

Called after content projection (ng-content) is initialized.

```typescript
ngAfterContentInit() {
  console.log('Content initialized');
}
```

## ngAfterContentChecked

Called after content is checked during change detection.

```typescript
ngAfterContentChecked() {
  console.log('Content checked');
}
```

## ngAfterViewInit

Called after component's view is initialized. Use for interacting with template elements.

```typescript
@ViewChild('myElement') element: ElementRef;

ngAfterViewInit() {
  this.element.nativeElement.focus();
}
```

## ngAfterViewChecked

Called after component's view and child views are checked.

```typescript
ngAfterViewChecked() {
  console.log('View checked');
}
```

## ngOnDestroy

Called just before component is destroyed. Use for cleanup (unsubscribe, cancel timers).

```typescript
ngOnDestroy() {
  this.subscription.unsubscribe();
  clearInterval(this.timer);
}
```

## Lifecycle Sequence

1. constructor
2. ngOnChanges
3. ngOnInit
4. ngDoCheck
5. ngAfterContentInit
6. ngAfterContentChecked
7. ngAfterViewInit
8. ngAfterViewChecked
9. ngOnDestroy (when component is destroyed)
