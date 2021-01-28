---
layout: single
classes: wide
title:  "Why `this` gets undefined in high order functions in Javascript?"
date:   2021-01-26 23:15 -0500
tags:
  - Javascript
  - programming
  - common-errors
categories: javascript common-errors
excerpt: "Understanding why sometimes in Javascript a method loses its reference to `this` when it is used as a high order function"
comments: true
---
When a function has been used as a high order function (passed as an argument) they lose their awareness of `this`, even if they are in the same object. E.g:

```javascript
    > function Person(name, gender) {
        this.name = name;

        this.sayHi = function (descriptor) {
            return `Hi ${descriptor()}`;
        }

        this.getName = function() {
            return this.name;
        }
     }

    > p = new Person("Jon");
    > p.sayHi(p.getName);

    'Hi undefined'
```

A quick way to solve this issue is by passing the function bound to the object. E.g.:

```javascript
    > p.sayHi(p.getName.bind(p));

    'Hi Jon'
```

But wouldn't be great to understand the source of this problem?

## A matter of context

Why this happens? Well, in Javascript when you assign a function of an object to a variable/parameter, you are actually detaching that method from its object:

```javascript
> const myFunc = instanz.myFunction;
// or 
> function myFunc2(f) {
    f();
 }
```

In such case, inside `myFunction` the value of `this` is not the same as `intanz` anymore, but it will get the value of `this` from the global context, which in [strict-mode] is `undefined` instead of the global `window` object.

Well, it doesn't look very clear at the beginning, but with a short example it will be easier to understand.

### An example to test the theory

1. Let's create a test class that is aware of its context

    ```javascript
    > function Soldier(numBullets) {
        this.numBullets = numBullets;

        this.checkStatus = function() {
            "use strict";
            console.log(this === captain);
            console.log(`I am ${this}`);
            console.log(`I have ${this.numBullets} bullets`);
        }
     }
    ```

2. Create an instance `captain` to see whether it works as expected

    ```javascript
    > captain = new Soldier(5);
    > captain.checkStatus();

    true
    I am [object Object]
    I have 5 bullets
    ```

3. Let's see what happens if we separate the function `checkStatus` from its object `captain`:

    ```javascript
    > captainStatus = captain.checkStatus;
    > captainStatus();
    
    false
    I am undefined
    Uncaught TypeError: Cannot read property 'numBullets' of undefined
    ```

This confirms that `checkStatus` is detached from `captain` and that `this` is `undefined`.

Does this happen every time or only when it is in `strict mode`? If the directive `strict mode` is removed and you detach the function from its object again, then running this in your browser should get the global object `window`:

```javascript
> function Soldier(numBullets) {
    this.numBullets = numBullets;

    this.checkStatus = function() {
        console.log(this === captain);
        console.log(`I am ${this}`);
        console.log(`I have ${this.numBullets} bullets`);
    }
  }
> captain = new Soldier(5);
> captainStatus = captain.checkStatus;
> captainStatus(); 

false
I am [object Window]
I have undefined bullets
```

This happened because in the browser `window` was the object that owned `captainStatus` at the moment of its execution. Try these same instructions again, but this time in a nodejs console to see what happens. Who is the owner of the method this time?

## Conclusions

Each time we are going to execute functions in Javascript, it is important to remember that:

- A method can be detached from an object into a separated variable, e.g. `const oprhan = obj.someMethod`.
- How we invoke a function changes the context on which it is executed.
- During a function invokation, i.e. the invokation of a function that is not attached to an object, `this` is going to be searched in the global context.
- The keyword `this` is `undefined` in a function invocation in strict mode.

It gets even more interesting when you learn that when using arrow functions in these scenarios you actually get a static context on different invokation types.

## See more

- [Gentle explanation of 'this' in Javascript](https://dmitripavlutin.com/gentle-explanation-of-this-in-javascript/)
- [The Mozilla official documentation about the operator 'this'](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

[strict-mode]: https://www.w3schools.com/js/js_strict.asp
