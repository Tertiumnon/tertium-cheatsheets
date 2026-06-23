# Templates

Angular template syntax for binding and control flow.

## Conditions

### @if

Conditional rendering based on expression.

```html
@if (isLoggedIn) {
  <p>Welcome back!</p>
} @else if (isPending) {
  <p>Loading...</p>
} @else {
  <p>Please log in</p>
}
```

### @switch

Switch statement with cases.

```html
@switch (status) {
  @case ('active') {
    <span class="badge-green">Active</span>
  }
  @case ('pending') {
    <span class="badge-yellow">Pending</span>
  }
  @default {
    <span class="badge-gray">Inactive</span>
  }
}
```

## Loops

### @for

Iterate over arrays and objects.

```html
@for (item of items; track item.id) {
  <div>{{ item.name }}</div>
  <div>Index: {{ $index }}</div>
  <div>First: {{ $first }}, Last: {{ $last }}</div>
}
```

**Track function:** Uniquely identifies items for DOM reuse (improves performance)

```html
@for (item of items; track item.id) {
  {{ item.name }}
}
```

## Data Binding

### Interpolation

```html
<p>{{ message }}</p>
<p>{{ 1 + 2 }}</p>
<p>{{ obj?.property }}</p>
```

### Property Binding

```html
<img [src]="imageUrl" />
<button [disabled]="isDisabled"></button>
<div [class.active]="isActive"></div>
```

### Attribute Binding

```html
<button [attr.aria-label]="buttonLabel"></button>
<table [attr.colspan]="colCount"></table>
```

### Class & Style Binding

```html
<!-- Single class -->
<div [class.active]="isActive"></div>

<!-- Multiple classes -->
<div [ngClass]="{ active: isActive, disabled: isDisabled }"></div>

<!-- Single style -->
<div [style.color]="textColor"></div>

<!-- Multiple styles -->
<div [ngStyle]="{ color: textColor, 'font-size': fontSize }"></div>
```

### Event Binding

```html
<button (click)="handleClick()"></button>
<input (input)="onInput($event)" />
<form (ngSubmit)="handleSubmit()"></form>
```

### Two-Way Binding

```html
<input [(ngModel)]="name" />
<app-counter [(count)]="total"></app-counter>
```

## Template Variables

### Template Reference Variables

```html
<input #myInput />
<button (click)="myInput.focus()">Focus</button>
```

### Built-in Variables

```html
@for (item of items; track item.id) {
  $index   <!-- 0-based index -->
  $first   <!-- true if first item -->
  $last    <!-- true if last item -->
  $even    <!-- true if even index -->
  $odd     <!-- true if odd index -->
  $count   <!-- length of iterable -->
}
```

## Directives

### Structural Directives

```html
*ngIf, *ngFor, *ngSwitch   <!-- Modify DOM structure -->
```

### Attribute Directives

```html
<div [ngClass]="classes"></div>
<div [ngStyle]="styles"></div>
```

## Safe Navigation Operator

```html
{{ user?.name }}           <!-- null-safe -->
{{ items?.[0] }}           <!-- safe array access -->
{{ getUser()?.address?.city }}  <!-- chained -->
```

## Pipes

Transform data for display.

```html
<p>{{ date | date: 'short' }}</p>
<p>{{ price | currency: 'USD' }}</p>
<p>{{ text | uppercase }}</p>
<p>{{ obj | json }}</p>

<!-- Chaining pipes -->
<p>{{ date | date: 'short' | uppercase }}</p>
```

## Form Directives

```html
<!-- Form setup -->
<form [formGroup]="myForm" (ngSubmit)="submit()">
  <!-- Input binding -->
  <input formControlName="name" />

  <!-- Error messages -->
  @if (myForm.get('name')?.hasError('required')) {
    <span>Name is required</span>
  }

  <button type="submit" [disabled]="myForm.invalid">Submit</button>
</form>
```
