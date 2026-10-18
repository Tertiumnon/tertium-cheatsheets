# JavaScript Classes

## ES5

### ES5 new class

```JavaScript
function Animal(name, x, y) {
  this.name = name;
  this.setLocation(x, y);
}

Animal.prototype.setLocation = function(x, y) {
    this.x = x;
    this.y = y;
};

Animal.prototype.getLocation = function() {
    return {
        x: this.x,
        y: this.y
    };
};

let myAnimal = new Animal('Beast', 50, 100);
console.log(myAnimal.getLocation());
```

### ES5 class extends

```JavaScript
function Cat(name, x, y, speed) {
    Animal.call(this, name, x, y);
    this.speed = speed;
}

Cat.prototype = Object.create(Animal.prototype);
Cat.prototype.constructor = Cat;

var myCat = new Cat('Kitty', 100, 200, 50);
console.log(myCat.getLocation());
```

## ES6

### ES6 new class

```JavaScript
class Rectangle {
  constructor(height, width) {
    this.height = height;
    this.width = width;
  }
  // Getter
  get area() {
    return this.calcArea();
  }
  // Method
  calcArea() {
    return this.height * this.width;
  }
}

const square = new Rectangle(10, 10);
console.log(square.area); // 100
```

### ES6 Inheritance

```JavaScript
class Animal {
  constructor(name) {
    this.name = name;
  }
  speak() {
    console.log(`${this.name} makes a sound`);
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);  // Call parent constructor
    this.breed = breed;
  }
  speak() {
    console.log(`${this.name} barks`);
  }
}

const dog = new Dog('Rex', 'Labrador');
dog.speak(); // 'Rex barks'
```

### Static Methods

Belong to class, not instances.

```JavaScript
class Math2 {
  static add(a, b) {
    return a + b;
  }
}

Math2.add(5, 3); // 8
```

### Private Fields

Prefix with `#` to make fields private.

```JavaScript
class Counter {
  #count = 0;  // Private field

  increment() {
    this.#count++;
  }

  getCount() {
    return this.#count;
  }
}

const counter = new Counter();
counter.increment();
console.log(counter.getCount()); // 1
console.log(counter.#count);     // SyntaxError: private field
```

### Setters & Getters

```JavaScript
class Temperature {
  #celsius = 0;

  set celsius(value) {
    if (value < -273.15) throw new Error('Invalid temp');
    this.#celsius = value;
  }

  get celsius() {
    return this.#celsius;
  }

  get fahrenheit() {
    return (this.#celsius * 9/5) + 32;
  }
}

const temp = new Temperature();
temp.celsius = 25;
console.log(temp.fahrenheit); // 77
```
