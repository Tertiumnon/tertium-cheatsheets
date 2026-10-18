# Change Detection Strategy

## Default

Every change detection cycle.

## OnPush

When inputs or properties change.

The "OnPush" change detection strategy should be used for components that are not constantly changing. This includes components that

- Display static data
- Only receive data from their parent components
- Do not perform any calculations or transformations on their data

```ts
import { Component, Input, ChangeDetectionStrategy, ChangeDetectorRef } from '@angular/core';

@Component({
  selector: 'app-onpush-dev',
  templateUrl: './onpush-dev.component.html',
  changeDetection: ChangeDetectionStrategy.OnPush
})
export class OnPushDevComponent {
  @Input() data: any;

  constructor(private cdr: ChangeDetectorRef) {}

  // This method marks the component for change detection.
  triggerChangeDetection() {
    this.cdr.markForCheck();
  }

  // Event handler that updates the input property.
  updateData() {
    this.data = { newValue: 'Updated value' };
  }
}
```

### ChangeDetectionRef

So you could try ref.detectChanges() to run change detection on the component and his children or ref.markForCheck() which does not trigger change detection and instead it will mark all onPush ancestors as to be checked once.
