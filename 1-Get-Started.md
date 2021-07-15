# Book 1, ed2

## Ch 2. Surveying JS

**values** are a fundamental unit of information. In JS this comes in 2 forms: **primitive** and **object**

primitive literals:

- strings
- numbers
- booleans
- null
- undefined
- symbol

objects value:

- arrays
- objects

arrays are actually a special type of object.

### declaring and using values

- `let` vs `var`
  - let allows a more limited access
  - this is called block scoping as opposed to function scoping

example:

```js
var adult = true;

if (adult) {
  var myName = "Kyle";
  let age = 39;
  console.log("Shhh, this is a secret!");
}

console.log(myName);
// Kyle

console.log(age);
// Error!
```

### functions

- functions are values that can be assigned and passed around
- functions are a special object value type
- not all languages treat functions as values, but it's essential for a language to support the functional programming pattern

### comparisons

- `===`'s are oftened described as 'checking both the value and the type'
- more specifically, this type of comparison disallows type coercion

the `===` operator does lie in two special cases

```js
NaN === NaN; // false
0 === -0; // true
```

- JS does not define `===` as structural equality for object values.
- instead JS defines `===` as identity equality. `===` compares objects by reference, not value
- in JS all objects are held be reference

```js
[ 1, 2, 3 ] === [ 1, 2, 3 ];    // false
{ a: 42 } === { a: 42 }         // false
(x => x * 2) === (x => x * 2)   // false

```

- JS doesn't cover structural comparisons because it's almost intractable to cover all the corner cases (e.g. closures).
- this is why even stringifying and comparing the source code doesn't take everything into account

## ch 3 digging into the roots of js

- functions have another characteristic that determines the scope of the variables that they can access (other than closure).

- this characteristic is best described as the _execution context_.

- the characteristic is exposed to the function by its `this` keyword

- scope is static. the variables available are defined only when the function is defined.
- the execution context is dynamic, it depends on the context in which the function is called.

examples:

```js
function classroom(teacher) {
  function study() {
    console.log(`${teacher} says to study ${this.topic}`);
  }
}

var assignment = classroom("melvin");

assignment();
// melvin says to study undefined
```

- `study()` is a context aware function.
- since this isn't in `strict` mode, with no context specified, the default context is the window.
- since no global variable `topic` is defined, `this.topic` resolves to `undefined`

```js
const homework = {
  topic: "js",
  assignment: assignment,
};

homework.assignment();
// melvin says to study js
```

```js
const otherHomework = {
  topic: "math",
};

assignment.call(otherHomework);
// melvin says to study math
```

- the `call(...)` method takes an object and uses it as the execution context to be referenced by `this`
- the benefit of `this` aware functions and dynamic contexts is that we can flexibly use the same functions by using the data from different objects.

### Prototypes

- a protoype is a characteristic of an object, the resolution of a property access
- think about a prototype as a linkage between two objects. the linkage is hidden, but can be exposed
- this linkage occurs when an object is created, it's linked to another object that already exists

- the purpose of a protoype linkage (e.g. from object B to object A) is that accesses against B for properties/methods that B does not have are delegated to A

consider the following example:

```js
const homework = {
  topic: "JS",
};

homework.toString();
// [object Object]
```

this works even if `homework` doesn't have a `toString()` method because the delegation invokes `Object.prototype.toString()` instead

#### Object Linkage

- an object linkage can be created using the `Object.create(...)` utility

```js
const homework = {
  topic: "js",
};

const otherHomework = Object.create(homework);

otherHomework.topic; // "js"
```

#### `this` revisited

- true importance of `this` is apparent when considering how it powers prototype-delegated function calls

- `this` supports dynamic contexts because method calls on objects that are delegated through the prototype chain still maintain the expected `this`

```js
const homework = {
  study() {
    console.log(`Please study ${this.topic}`);
  },
};

const jsHomework = Object.create(homework);
jsHomework.topic = "JS";
jsHomework.study();
// Please study JS

const mathHomework = Object.create(homework);
mathHomework.topic = "math";
mathHomework.study();
// Please study math
```
