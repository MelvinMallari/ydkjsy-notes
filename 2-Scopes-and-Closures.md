# Book 2 Scope Closures

is Javascript compiled or interpreted?

- what is compilation?

  - the process of taking source code an turning it into an executable program
  - there are three steps:
    - `tokenizing/lexing`: breaking up the code into smaller meaningfully understood chunks, called tokens
      - the difference between the two is subtle. tokenizing uses _stateful_ parsing rules to determine whether a variable (e.g. `a`) should be considered a distinct token or just part of another token (lexing).
    - `parsing`: taking the array of tokens and turning it into a tree of nested elements. This tree represents the grammatical structure of the program. It is known as an Abtract Syntax Tree (AST)
    - `code generation`: take the parsed AST and convert it into executable code. In this step, the JS engine takes the AST and turns it into machin executable code

- what is interpretation?

  - interpretation transforms your program into machine-understandable instruction. with interpretation, your source code is transformed and executed line by line before proceeding to the next line.

- interpretation & compilation processing models are generally mutually exclusive
- where things get muddy is that interpretation can take other forms than line by line operation
- the JS engine uses variations of both compilation and interpretation to handle JS programs
- kyle simpson argues that JS is closer to being compiled then interpreted

## chapter 2: illustrating lexical scope

the JS engine can be thought of as three members having a conversation

- Engine: responsible for start-to-finish compilation & execution
- Compilers: handles parsing and code generation
- Scope Manager: collects and maintains a lookup list of all declared variables/identifiers. enforces rules as to how these are accessible to currently executing code.

take the following example code:

```js
var students = [
  { id: 14, name: "Kyle" },
  { id: 73, name: "Suzy" },
  { id: 112, name: "Frank" },
  { id: 6, name: "Sarah" },
];

function getStudentName(studentID) {
  for (let student of students) {
    if (student.id == studentID) {
      return student.name;
    }
  }
}

var nextStudent = getStudentName(73);

console.log(nextStudent);
// Suzy
```

### how a statement like `var students = [ ... ]` is processed by the Compiler:

1. Compiler sets up the declaration of the scope variable (since it wasn't previously declared in current scope)
2. while Engine is executing, to process the assignment part of the statement, Engine asks Scope Manager to look up variable, initializes it to `undefined` so it's ready to use, then assigns array value to it.

what does this look like in conversation form?

- the conversation is in a Q&A form.
- the Compiler, when it runs into an identifier, asks the scope manager if an identifier declaration has already been encountered
  - if "no" the Scope Manager creates that variable in that scope
  - if "yes" then it is skipped, because there's nothing the Scope Manager can do
- similar 'conversations' occur when the Compiler runs into functions or block scopes and new scope buckets/Scope Managers need to be instantiated

### how the code is executed by the Engine:

what does this look like in conversation form?

- the conversation is in a Q&A form.
- the Engine asks the Scope Manager to look up hoisted identifiers or target rferences to associate the correct thing to it

### Nested Scope

- Scopes can be arbitrarily nested
- Each scope gets its own "Scope Manager" instance each time that scope is executed
- Each scope as all of its identifiers registered at the start of the scope being executed (this is known as 'variable hoisting')

  - if an identifier came from a `function` declaration, that variable is immediately initialized to its associated function reference
  - if an identifier comes from a `var` declaration, it is immediately initialized to `undefined` so it may be used
  - otherwise the identifier remains uninitialized (aka in its "Temporal Dead Zone") and cannot be used until its declaration

- Any time an identifier reference cannot be found in the current scope, the next outer scope is consulted. this is repeated until found or there are no more scopes to consult.

### Lookup Failures

- when the Engine exhausts all lexically available scopes, an error conditions exists. Depending on program mode (strict or not) & variable role (target or source), the error is handled differently

- if variable is a **source**, it is considered undeclared -> `ReferenceError`
- if variable is a **target**, and program is in strict-mode it is considered undeclared -> `ReferenceError`

- if the variable is a **target** and strict mode is off, a legacy behavior kicks in where the global scope's Scope Manager will create a global variable that fulffills the target assignment.

```js
function getStudentName() {
  // assignment to an undeclared variable :(
  nextStudent = "Suzy";
}

getStudentName();

console.log(nextStudent);
// "Suzy" -- oops, an accidental-global variable!
```

this is a behavior leads to nasty bugs, so always use strict-mode!!

## chapter 3 The Scope Chain

### "lookup"

- the lookup process to determine which scope a variable belongs to does not actually happen on runtime. The meta information of what scope a variable originates from is usually determined during the initial compilation processing.
- Since a variable's scope is known during compilation and is immutable, this information is stored in the variable's entry in the AST.
- 'usually determined..?' consider a reference to a variable not declared in any lexically available scope in the file. if not declaration is found in the file, another file may have declared that variable in a shared global scope.
  - so the ultimately, there may need to be a runtime lookup process to determine if a variable was appropriately declared

### shadowing

- where having different lexical scopes matter if when you have two or more variables, each in different scopes, with the same lexical name.

  consider the following example code:

```js
var studentName = "Suzy";

function printStudent(studentName) {
  studentName = studentName.toUpperCase();
  console.log(studentName);
}

printStudent("Frank");
// FRANK

printStudent(studentName);
// SUZY

console.log(studentName);
// Suzy
```

- `studentName` within the function `printStudent(..)` scope \*_shadows_ `studentName` in the global scope.
- by shadowing the variable with same name in the outerscope, it becomes impossible reference the outer `studentName` within the function `printStudent(..)`'s scope.

### Global Unshadowing Trick

- loosely `var` declarations also expose themselves as properties in the global object.
- you can use this property to access variables that have been shadowed
- note this only works for the global scope. any intermediate scopes are still impossible to access.

```js
var studentName = "Suzy";

function printStudent(studentName) {
  console.log(studentName);
  console.log(window.studentName);
}

printStudent("Frank");
// "Frank"
// "Suzy"
```

### Illegal Shadowing

- `let` can shadow `var`, but `var` cannot shaddow `let`

```js
function something() {
  var special = "JavaScript";

  {
    let special = 42; // totally fine shadowing

    // ..
  }
}

function another() {
  // ..

  {
    let special = "JavaScript";

    {
      var special = "JavaScript";
      // ^^^ Syntax Error

      // ..
    }
  }
}
```

- the reason you can't do this is because the `var` declaration is attempting to "cross the boundary" or "hop" over the `let` declaration of the same name.

### Function Name Scope

consider a function declaration:

```js
function askQuestion() {
  // ..
}
```

- the function declaration creates an identifier in the enclosing (in this case global) scope named `askQuestion`

consider a function expression

```js
var askQuestion = function () {
  // ..
};
```

- the function expression also creates an identifier in the enclosing (in this case global) scope named `askQuestion`. Since it is not a function declaration, the function will not 'hoist'.
- another difference between function declarations vs expressions is what happens to the name identifier of the function
- consider a named expression:

```js
var askQuestion = function ofTheTeacher() {
  // ..
};
```

- `askQuestion` ends up in the outer scope. but `ofTheTeacher` is declared as an identifier **inside the function itself**. it is only defined as read only.

```js
var askQuestion = function ofTheTeacher() {
  console.log(ofTheTeacher);
};

askQuestion();
// function ofTheTeacher()...

console.log(ofTheTeacher);
// ReferenceError: ofTheTeacher is not defined
```

### Arrow Functions

- anonymous functions are often claimed to behave differently with respect to lexical scoping from the standard `function` functions. this is not true.

# Chapter 4: Around the Global Scope

## Why Global Scope?

how do seperate files get stitched together in a single runtime context by the JS engine? 3 ways:

1. Using ES modules. Each module then `import`s references to other modules they need access to. Seperate module files cooperate with each other through these shared imports, without needing a shared outer scope.

2. Using a bundler. All the files are typically concatenated together before delivery to the broswer and JS engine, which then processes a single big file.
   In some build setups, all of the contents of a file are wrapped in a single enclosing scope like a wrapper function, universal module. Each piece accesses other pieces by using local variables in that shared scope.

3. either using a bundler, or if they are loaded in the browser individually, if there is no surrounding scope encompassing all these pieces, the global scope is the only way for them to cooperate with each other.

a bundled field often looks like this:

```js
var moduleOne = (function one() {
  // ..
})();
var moduleTwo = (function two() {
  // ..

  function callModuleOne() {
    moduleOne.someMethod();
  }

  // ..
})();
```

In addition to accounting where an application's code resides during runtime and how each piece is able to access other pieces to cooperate, the global scope is where:

- JS exposes its built in:

  - primitives: `undefined`, `null`, `Infinity`, `Nan`
  - natives: `Date()`, `Object()`, `String()`
  - global functions: `eval()`, `parseInt()`
  - namespaces: `Math`, `Atomics`, `JSON`
  - friends of JS: `Intl`, `WebAssembly`

- the environment hosting the JS engine exposes its own built-ins
  - `console` & methods
  - the DOM (`window`, `document`)
  - timers (`setTimeout(..)`)
  - web platform API's: `navigator`, `history`, `geolocation`, `WebRTC` etc

Most developers agree that the global scope shouldn't be a variable dumping ground. This leads to confusing bugs. But the global scope is an important glue for every JS application.

## Where Exactly is this Global Scope?

Different JS env's handle scopes of your programs, especially global scope, differently.

### Browser "Window"

the "purest" (least amount of intrusion) environment JS can be run is is a standalone .js file in a web page in a browser.

consider the following code loaded using a `<script>`, `<sript src =..>` or dynamically created `<script>` DOM element.

```js
var studentName = "Kyle";

function hello() {
  console.log(`Hello, ${window.studentName}!`);
}

window.hello();
// Hello, Kyle!
```

note, global object property can be shadowed by a global variable:

```js
window.something = 42;

let something = "Kyle";

console.log(something);
// Kyle

console.log(window.something);
// 42
```

the `let` declaration adds a `something` global variable, but not a global object property. It is probably a bad idea to create a a divergence between the global object and global scope. A simple way to avoid is is to always use `var` for globals and reserve `let` and `const` for block scopes.

### Web Workers

Web Workers are a web platform extension on top of browser-JS behavior. They allow a JS file to run in a completely seperate thread (operating system wise) from a thread that's running the main JS program.

Since Web Workers run on a separate thread, they are restricted in their communications with the main application thread to avoid/limit race conditions. e.g. Web Workers do not have access to the DOM. Some web API's are made avaiable to the worker, e.g. `navigator`

Web Workers are a entirely separate program and does not share global scope with the main JS program. In a Web Worker, the global object reference is made using `self`.

```js
var studentName = "Kyle";
let studentID = 42;

function hello() {
  console.log(`Hello, ${self.studentName}!`);
}

self.hello();
// Hello, Kyle!

self.studentID;
// undefined
```

### Developer Tools Console/REPL

These tools optimize for developer experience. They emulate the global scope to an extent. Divergence may include:

- behavior for the global scope
- hoisting
- block scoping declarators (`let` / `const`)

### ES Modules (ESM)

- Within ES modules, global variables are not created when declaring variables at the top-level scope.
- The module's top-level scope is descended from the global scope.
- This also means that the all global variables are available from inside the module's scope.
- There are still plenty of JS and web globals that you can/will access from the global scope.

### Node

Node treats every single .js file that it loads, including the main one you start the Node process with as a module. This means that the top level of your Node program is never the global scope.

# Ch 5. The (Not So) Secret Lifecycle of Variables

JS's handling how variables come into existence and when they become available to the program is rich with nuance.

## When Can I Use a Variable?

Consider that this code works fine:

```js
greeting();
// Hello!

function greeting() {
  console.log("Hello!");
}
```

Why can you acces the identifier `greeting` from line 1, even though the function declaration doesn't occur until line 4.

- remember that all identifiers are reigstered to their respective scope during compile time.
  - every identifier is **created** at the beginning of the scope it belongs to, every time that scope is entered.
- a variable being visible from the beginning of its enclosing scope, even though declaration is further down is called **hoisting**
- still, how can well call `greeting` before it's been declared?

  - in formal `function` declarations, when a name identifier is registered at the top of its scope, it's also auto-initialized. this is called function hoisting

- note function hosting and `var` variable hoisting attach their name identifier to the nearest enclosing **function scope**, not block scope.

## Variable Hoisting

consider:

```js
greeting = "Hello!";
console.log(greeting);
// Hello!

var greeting = "Howdy!";
```

how?

- the identifier is hoisted
- it's automatically initialized to the value `undefined` from the top of the scope.

## Hoisting: Yet Another Metaphor

conceptually hoisting can be thought of as reorganizing the code before executing

e.g.

```js
studentName = "Suzy";
greeting();
// Hello Suzy!

function greeting() {
  console.log(`Hello ${studentName}!`);
}
var studentName;
```

```js
function greeting() {
  console.log(`Hello ${studentName}!`);
}
var studentName;

studentName = "Suzy";
greeting();
// Hello Suzy!
```

however, this is mental model innacurate! the JS engine does not rearrange the code. It must fully parse the code to implement this behavior.

In this regard, hoisting is more of a compile-time operation for generating runtime instructions for auto variable registration at the beginning of its scope, each time that scope is entered

### Re-declaration?

consider:

```js
var greeting;

function greeting() {
  console.log("Hello!");
}

// basically, a no-op
var greeting;

typeof greeting; // "function"

var greeting = "Hello!";

typeof greeting; // "string"
```

things to note:

- `var studentName;`, doesn't mean `var studentName = undefined;`
- the second `var greeting` is essentially a noop

what about repeating declaration within a scope using `let` or `const`?

```js
let studentName = "Frank";

console.log(studentName);

let studentName = "Suzy";
```

the program will not execute, and will instead throw a `SyntaxError`.

- this goes for any chronological combination of `var`, `let`, or `const`
- why is this disallowed for `let`? because of stylistic opinions rather than technical reasons.

what about `const`?

- the `const` keyword requires a variable to be initialized, so ommitting an assignment results in `SyntaxError`
- const declarations cannot be re-assigned

### Loops

- clearly JS doesn't want us to "re-declare" variables within the same scope

what about loops? consider:

```js
var keepGoing = true;
while (keepGoing) {
  let value = Math.random();
  if (value > 0.5) {
    keepGoing = false;
  }
}
```

is `value` re-declared repeatedly in this program? No.

all the rules of scope are applied **per scope instance**. Each time a scope is entered during execution, everything resets.

### Uninitialized Variables (TDZ)

- `var` declarations are "hoisted" to the top of its scope. it is also automatically initialized to the `undefined` value.
- `let` and `const` do not behave this way.

consider:

```js
console.log(studentName);
// ReferenceError

let studentName = "Suzy";
```

also:

```js
studentName = "Suzy"; // let's try to initialize it!
// ReferenceError

console.log(studentName);

let studentName;
```

the only way to initialized an uninitialized variable for `let`/`const` is with an assignment attached to the declaration statement. e.g.

```js
let studentName = "Suzy";
console.log(studentName); // Suzy

// alternatively
let studentName;
// or:
// let studentName = undefined;

// ..

studentName = "Suzy";

console.log(studentName);
// Suzy
```

- so we see that we cannot use the variable prior to when it is initialized for `const` and `let`
- the term coined by TC39 referred to the period of time from entering ascope to where auto-initialization of the variable occurs is **Temporal Dead Zone (TDZ)**.
- a `var` also technically has a TDZ, but it's length is zero and therefore unobservable.

- debate about hoisting:
  - is auto-initialization part of hoisting?
  - here we are defining hoisting as auto-registration of a variable at the top of the scope
  - auto-initialization at the top of the scope (to `undefined`) is defined here as a separate operation

distilled

- `var` hoists & auto-initializes
- `const` & `let` hoist but do not auto-initialize

- `const` & `let` hoisting can be displayed in the following code:

```js
var studentName = "Kyle";

{
  console.log(studentName);
  // ???

  // ..

  let studentName = "Suzy";

  console.log(studentName);
  // Suzy
}
```

how to avoid TDZ errors? -> always put `let` and `const` declarations at the top of any scope to minimize TDZ window length.
