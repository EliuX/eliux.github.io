---
layout: single
classes: wide
title:  "Why this gets undefined in high order functions in Javascript"
date:   2021-01-26 23:15 -0500
tags:
  - Javascript
  - programming
  - common-errors
categories: javascript common-errors
excerpt: "Understanding why in Javascript methods lose their reference to `this` when they are used as high order functions"
comments: true
---

When a function has been used as a high order function (passed as an argument) they lose their awareness of `this`. In such cases, a common way to solve this problem is by passing such function bound to `this`. E.g.

```javascript
this.myFunction.bind(this);
```

But wouldn't be great to understand the source of this problem?

## A matter of context

Why this happens? Well, in Javascript when you assign a function of an object to a variable or a parameter, you are actually detaching that method from its object:

```javascript
const myFunc = instanz.myFunction;
// or 
function myFunc2(f) {
    f();
}
```

In such case, inside `myFunction` the value of `this` is not the same as `intanz` anymore, but it will get the value of `this` from the global context, which in [strict-mode] is `undefined` instead of the global `window` object.

Well, it doesn't look very clear at the beginning, but with a short example it will be easier to understand.

### An example to test the theory

1. Let's create a test class that is aware of its context

    ```javascript
    function Soldier(numBullets) {
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
    captain = new Soldier(5);
    captain.checkStatus();

    > true
    > I am [object Object]
    > I have 5 bullets
    ```

3. Let's see what happens if we separate the function `checkStatus` from its object `captain`:

    ```javascript
    captainStatus = captain.checkStatus;
    captainStatus();
    
    > false
    > I am undefined
    > Uncaught TypeError: Cannot read property 'numBullets' of undefined
    ```

This confirms that `checkStatus` is detached from `captain`. Also, that in the global context, `this` is `undefined`.

But is that so every time or only when it is in `strict mode`? If the directive `strict mode` is removed and you repeat all previous steps, then running this in your browser should get the global object `window`, because it is the object that owns the method at the moment of this execution:

```javascript
captainStatus();

> false
> I am [object Window]
> I have undefined bullets
```

## Conclusions

It is important to understand that:

- A method can be extracted from an object into a separated variable, e.g. `const oprhan = obj.someMethod`, in which case the separated variable
is detached from the original object.
- How we invoke a function changes the context on which it is executed
- The keyword `this` is the global object in a function invocation.
- The keyword `this` is undefined in a function invocation in strict mode

It even gets more interesting when you learn that when using arrow functions you get a static context on different invokation types.

## See more

- [Gentle explanation of `this` in Javascript](https://dmitripavlutin.com/gentle-explanation-of-this-in-javascript/)
- [The Mozilla official documentation about the operator `this`](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/this)

[strict-mode]: https://www.w3schools.com/js/js_strict.asp
