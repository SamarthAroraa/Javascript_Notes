# Types & Grammar

![Types & Grammar](https://res.cloudinary.com/startup-grind/image/upload/c_fill,dpr_2.0,f_auto,g_center,h_1080,q_100,w_1080/v1/gcs/platform-data-duolingo/events/grammar_BgnJsH1.jpg)

## Built-in Types

- A type is an intrinsic, built-in set of characteristics that uniquely identifies the behavior of a particular value and distinguishes it from other values, both to the **Engine** and to the developer.
- Javascript defines seven built-in types:
    1. `null`
    2. `undefined`
    3. `boolean`
    4. `number`
    5. `string`
    6. `object`
    7. `symbol` --- added in ES6!

    All of these types except `object` are called **primitives**.
- The `typeof` operator inspects the type of the given value, and always returns one of seven string values. For example:

    ```js
    typeof undefined        === "undefined"; // true
    typeof true             === "boolean";   // true
    typeof 42               === "number";    // true
    typeof "42"             === "string";    // true
    typeof { life: 42}      === "object";    // true
    typeof [1, 2, 3]        === "object";    // true
    typeof function () { }  === "function";  // true
    typeof null             === "object";    // true - buggy in JavaScript
    typeof Symbol()         === "symbol";    // true - added in ES6
    ```

- `null` is the only primitive value that is **falsy**, but that also returns `"object"` from the `typeof` check.
- Function is actually a **subtype** of object. They can have properties. For example:

    ```js
    function a(b, c) {
        /* .. */
    }
    ```

    The function object has a `length` property set to the number of formal parameters it is declared with. For instance:

    ```js
    a.length; // 2
    ```

## Values as Types

- In **JavaScript**, variables don't have types -- **values have types**. Variables can hold any value, at any time.
- A variable can, in one assignment statement, hold a `string`, and in the next hold a `number`, and so on.
- The value, like `"42"` with the `string` type, can be created from the `number` value through a process called **coercion**.
- If you use `typeof` against a variable, it's not asking "what's the type of the variable?" as it may seem, since **JavaScript** variables have to types. Instead, it's asking "what's the type of the value in the variable?". For example:

    ```js
    var a = 42;
    typeof a; // "number"

    a = true;
    typeof a; // "boolean"
    ```

- The `typeof` operator always returns a string. So:

    ```js
    typeof typeof 42; // "string"
    ```