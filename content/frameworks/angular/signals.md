# Signals

Signals are all about increasing the runtime performance of your application, by getting rid of Zone.js. With Signals, Angular will be able to determine exactly what parts of the page need to be updated and update only those and nothing more.

## Signal

Methods:

- set
- update

```ts
import { computed, signal } from '@angular/core';

// state/writable signal
const counter = signal(0);

// computed signal
const isEven = computed(() => (counter() & 1) == 0);

counter() // current value: 0
isEven() // computed: true

counter.set(1)

counter() // 1
isEven() // false
```

## Input
