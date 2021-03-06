# Async & Performance

![Async & Performance](https://bs-uploads.toptal.io/blackfish-uploads/blog/post/seo/og_image_file/og_image/19504/0425-ToPromiseOrNotToPromise-Waldek_Social-e67f662e816296781823c9a99d12c7b7.png)

## Preface

- Promises are now the official way to provide async return values in both **JavaScript** and the **DOM**.
- The relationship between the **now** and **later** part of your program is at the heart of asynchronous programming.

## Parallel Threading

- Remember, async is about the gap between now and later. But parallel is about things being able to occur simultaneously.
- **JavaScript** never shares data across threads, which means that level of nondeterminism isn't a concern.
- Consider:

    ```js
    var a = 1;
    var b = 2;

    function foo() {
        a++;
        b = b * a;
        a = b + 3;
    }

    function bar() {
        b--;
        a = 8 + b;
        b = a * 2;
    }

    // ajax(..) is some arbitrary Ajax function given by a library
    ajax("http://some.url.1", foo);
    ajax("http://some.url.2", bar);
    ```

    Chunk 1 is synchronous (happens now), but chunk 2 and 3 are asynchronous (happen later), which means their execution will be separated by a gap of time. Chunk 1:

    ```js
    var a = 1;
    var b = 2;
    ```

    Chunk 2 (`foo()`):

    ```js
    a++;
    b = b * a;
    a = b + 3;
    ```

    Chunk 3 (`bar()`):

    ```js
    b--;
    a = 8 + b;
    b = a * 2;
    ```

## Interaction

- To address such a race condition, you can coordinate ordering interaction. For example:

    ```js
    var res = [];

    function response(data) {
        if (data.url == "http://some.url.1") {
            res[0] = data;
        }
        else if (data.url == "http://some.url.2") {
            res[1] = data;
        }
    }

    // ajax(..) is some arbitrary Ajax function given by a library
    ajax("http://some.url.1", response);
    ajax("http://some.url.2", response);
    ```

    Regardless of which Ajax response comes back first, `res[0]` will always hold the `"http://some.url.1"` results and `res[1]` will always hold the `"http://some.url.2"` results. Through simple coordination, we eliminated the **race condition** nondeterminism.

## Jobs

- The **Job queue** is a queue hanging off the end of every tick in the event loop queue.
- Certain async-implied actions that may occur during a tick of the event loop will not cause a whole new event to be added to the event loop queue, but will instead add an item (aka Job) to the end of the current tick's Job queue.
- The event loop queue is like an amusement park ride, where once you finish the ride, you have to go to the back of the line to ride again. But the Job queue is like finishing the ride, but then cutting in line and getting right back on.
- Jobs are kind of like the spirit of the `setTimeout(..0)` hack, but implemented in such a way as to have a much more well-defined and guaranteed ordering: **later, but as soon as possible**.
- Consider:

    ```js
    console.log("A");

    setTimeout(function () {
        console.log("B");
    }, 0);

    // theoretical "Job API"
    schedule(function () {
        console.log("C");

        schedule(function () {
            console.log("D");
        });
    });
    ```

    It would print out `A C D B`, because the Jobs happen at the end of the current event loop tick, and the timer fires to schedule for the next event loop tick (if available!).

## Callbacks

- Callbacks are by far the most common way that asynchrony in **JavaScript** programs is expressed and managed. Indeed, the callback is the most fundamental async pattern in the language.
- Callback function wraps or encapsulates the continuation of the program.
- Consider:

    ```js
    doA(function () {
        doB();

        doC(function () {
            doD();
        })

        doE();
    });

    doF();
    ```

    The operations will happen in this order:

    ```js
    `doA()`
    `doF()`
    `doB()`
    `doC()`
    `doE()`
    `doD()`
    ```

    So imagine this snippent working like this:

    ```js
    doA(function () {
        doC();

        doD(function () {
            doF();
        })

        doE();
    });

    doB();
    ```

- **Inversion of control** is when you take part of your program and give over control of its execution to another third party.

## Trying to Save Callbacks

- Split-callback design:

    ```js
    function success(data) {
        console.log(data);
    }

    function failure(data) {
        console.log(data);
    }

    ajax("http://some.url.1", success, failure);
    ```

- Error-first style (or **Node** style) design:

    ```js
    function response(err, data) {
        // error?
        if (err) {
            console.log(err);
        }
        // otherwise, assume success
        else {
            console.log(data);
        }
    }

    ajax("http://some.url.1", response);
    ```

- Always invoke callbacks asynchronously, even if that's **right away** on the next turn of the event loop, so that all callbacks are predictably async.

## Promises

- Most new async APIs being added to **JavaScript**/DOM platform are being built on Promises.
- Promises can have, at most, one resolution value (fulfillment or rejection).
- With Promises, the `then(..)` call can actually take two functions, the first for fulfillment, and the second for rejection. For example:

    ```js
    add(fetchX(), fetchY())
    .then(
        // fulfillment handler
        function (sum) {
            console.log(sum);
        },
        // rejection handler
        function (err) {
            console.log(err); // bummer!
        }
    );
    ```

- Consider:

    ```js
    function foo(x) {
        // start doing something that could take a while

        // make a `listener` event notification
        // capability to return

        return listener;
    }

    var evt = foo(42);

    evt.on("completion", function () {
        // now we can do the next step!
    });

    evt.on("failure", function () {
        // oops, something went wrong with `foo(..)`
    });
    ```

    `foo(..)` expressly creates an event subscription capability to return back, and the calling code receives and registers the two event handlers against it.
- In Promises we use `then(..)` to register a **then** event. Or perhaps more precisely, `then(..)` registers **fulfillment** and/or **rejection** event(s), though we don't see those terms used explicitly in the code.
- Consider:

    ```js
    function foo(x) {
        // start doing something that could take a while

        // construct and return a promise
        return new Promise(function (resolve, reject) {
            // eventually, call `resolve(..)` or `reject(..)`,
            // which are the resolution callbacks for
            // the promise.
        });
    }

    var p = foo(42);

    bar(p);

    baz(p);
    ```

    The internal of `bar(..)` and `baz(..)` might look like:

    ```js
    function bar(fooPromise) {
        // listen for `foo(..)` to complete
        fooPromise.then(
            function () {
                // `foo(..)` has now finished, so do `bar(..)`'s task
            },
            function () {
                // oops, something went wrong in `foo(..)`
            }
        );
    }

    // ditto for `baz(..)`
    ```

- Given that Promises are constructed by the `new Promise(..)` syntax.
- Consider:

    ```js
    Object.prototype.then = function () {};
    Array.prototype.then = function () {};

    var v1 = { hello: "world" };
    var v2 = { "Hello", "World"};
    ```

    Both `v1` and `v2` will be assumed to be thenables. But they're don't.
- Thenable duck typing can be hazardous if it incorrectly identifies something as a Promise that isn't.

## Promise Trust

- When you call `then(..)` on a Promise, even if that Promise was already resolved, the callback you provide to `then(..)` will **always** be called asynchronously. No more need to insert your own `setTimeout(..,0)` hacks. Promises prevent Zalgo **automatically**.
- When a Promise is resolved, all `then(..)` registered callbacks on it will be called, in order, immediately at the next asynchronous opportunity, and nothing that happens inside of one of those callbacks can affect/delay the calling of the other callbacks. For example:

    ```js
    p.then( function () {
        p.then(function () {
            console.log("C");
        });
        console.log("A");
    });
    p.then(function () {
        console.log("B");
    });
    // A B C
    ```

- If you register both fulfillment and rejection callbacks from a Promise, and the Promise get resolved, one of the two callbacks will always be called.
- If for some reason the Promise creation code tries to call `resolve(..)` or `reject(..)` multiple times, or tries to call both, the Promise will accept only the first resolution, and will silently ignore any subsequent attempts. Because a Promise can only be resolved once, any `then(..)` registered callbacks will only ever be called once (each).
- If you register the same callback more than once, (e.g., `p.then(..); p.then(..);`), it'll be called as many times as it was registered.
- If you don't explicitly resolve with a value either way, the value is `undefined`, as is typical in **JavaScript**.
- If you call `resolve(..)` or `reject(..)` with multiple parameters, all subsequent parameters beyond the first will be silently ignored.
- If you reject a Promise with a reason (aka error message), that value is passed to the rejection callback(s).
- If you pass an immediate, non-Promise, non-thenable value to `Promise.resolve(..)`, you get a promise that's fulfilled with that value. In other words, these two promises `p1` and `p2` will behave basically identically:

    ```js
    var p1 = new Promise(function (resolve, reject) {
        resolve(42);
    });

    var p2 = Promise.resolve(42);
    ```

    But if you pass a genuine Promise to `Promise.resolve(..)`, you just get the same promise back:

    ```js
    var p1 = Promise.resolve(42);

    var p2 = Promise.resolve(p1);

    p1 === p2; // true
    ```

- `Promise.resolve(..)` will accept any thenable, and will unwrap it to its non-thenable value. But you get back from `Promise.resolve(..)` a real, genuine Promise in its place, **one that you can trust**.

## Chain Flow

- We can string multiple Promises together to represent a sequence of async steps.
- Every time you call `then(..)` on a Promise, it creates and returns a new Promise, which we can chain with.
- Whatever value you return from the `then(..)` call's fulfillment callback (the first parameter) is automatically set as the fulfillment of the chained Promise (from the first point).

- Consider:

    ```js
    var p = Promise.resolve(21);

    var p2 = p.then(function (v) {
        console.log(v); // 21

        // fulfill `p2` with value `42`
        return v * 2;
    });

    // chain off `p2`
    p2.then(function (v){
        console.log(v); // 42
    });
    ```

    Thankfully, we can easily just chain these together:

    ```js
    var p = Promise.resovle(21);

    p.then(function (v) {
        console.log(v); // 21

        // fulfill the chained promise with value `42`
        return v * 2;
    }).then(function (v) {
        console.log(v); // 42
    });
    ```

- Each Promise resolution is thus just a signal to proceed to the next step.
- Consider:

    ```js
    function delay(time) {
        return new Promise(function (resolve, reject) {
            setTimeout(resolve, time);
        });
    }

    delay(100) // step 1
        .then(function STEP2() {
            console.log("step 2 (after 100ms)");
            return delay(200);
        })
        .then(function STEP3() {
            console.log("step 3 (after another 200ms)");
        })
        .then(function STEP4() {
            console.log("step 4 (next Job)");
            return delay(50);
        })
        .then(function STEP5() {
            console.log("step 5 (after another 50ms)");
        });
    ```

    Calling `delay(200)` creates a promise that will fulfill in 200ms, and then we return that from the first `then(..)` fulfillment callback, which causes the second `then(..)`'s promise to wait on that 200ms promise.
- If you call `then(..)` on a promise, and you only pass a fulfillment handler to it, an assumed rejection handler is substitute. For example:

    ```js
    var p = new Promise(function (resolve, reject) {
        reject("Oops");
    });

    var p2 = p.then(
        function fulfilled() {
            // never gets here
        }
        // assumed rejection handler, if omitted or
        // any other non-function value passed
        // function(err) {
        // throw err;
        // }
    );
    ```

    In essence, this allows the error to continue propagating along a Promise chain until an explicitly defined rejection handler is encountered.
- If a proper valid function is not passed as the fulfillment handler parameter to `then(..)`, there's also a default handler substituted. For example:

    ```js
    var p = Promise.resolve(42);

    p.then(
        // assumed fulfillment handler, if omitted or
        // any other non-function value passed
        // function (v) {
        //     return v;
        // }
        null,
        function rejected(err) {
            // never gets here
        }
    );
    ```

## Terminology: Resovle, Fulfill, and Reject

- Consider:

    ```js
    var p = new Promise(function (X, Y) {
        // X() for fulfillment
        // Y() for rejection
    });
    ```

    The first parameter is usually used to mark the Promise as fulfilled, and the second parameter always marks the Promise as rejected. Almost all literature uses `reject(..)` as second parameter name, and because that's exactly (and only!) what is does, that's a very good choice for the name. I'd strongly recommand you always use `reject(..)`. For eaxmple:

    ```js
    var p = new Promise(function (resolve, reject) {
        // resolve() for fulfillment
        // reject() for rejection
    });
    ```

- Consider:

    ```js
    var rejectedPr = new Promise(function (resolve, reject) {
        // resolve this promise with a rejected promise
        resolve(Promise.reject("Oops"));
    });

    rejectedPr.then(
        function fulfilled() {
            // never gets here
        },
        function rejected(err) {
            console.log(err); // "Oops"
        }
    );
    ```

    It should be clear now that `resolve(..)` is the appropriate name for the first callback parameter of the `Promise(..)` constructor.

## Error Handling

- Consider:

    ```js
    function foo(){
        setTimeout(function () {
            baz.bar();
        }, 100);
    }

    try {
        foo();
        // later throws global error from `baz.bar()`
    }
    catch (err) {
        // never gets here
    }
    ```

    `try..catch` would certainly be nice to have, but it doesn't work across async operations. Now Consider:

    ```js
    var p = Promise.resolve(42);

    p.then(
        function fulfilled(msg) {
            // numbers don't have string functions, so will throw an error
            console.log(msg.toLowerCase());
        },
        function rejected(err) {
            // never gets here
        }
    )
    ```

    To avoid losing an error to the silence of a forgotten/discarded Promise, always end your chain with a final `catch(..)`, like:

    ```js
    var p = Promise.resolve(42);

    p.then(
        function fulfilled(msg) {
            // numbers don't have string functions, so will throw an error
            console.log(msg.toLowerCase());
        }
    ).catch(handleErrors);
    ```

## Promise Patterns

- Consider:

    ```js
    var p1 = request("http://some.url.1/");
    var p2 = request("http://some.url.2/");

    Promise.all([p1, p2])
        .then(function (msgs) {
            // both `p1` and `p2` fulfill and pass in
            // their messages here
            return request(
                "http://some.url.3/?v=" + msgs.join(",")
            );
        })
        .then(function (msg) {
            console.log(msg);
        });
    ```

    `Promise.all([..])` expects a single arguments, and `array`, consisting generally of Promise instances. The promise returned from the `Promise.all([..])` call will receive a fulfillment message (`msg` in this snippet) that is an `array` of all the fulfillment messages from the passed in promises, in the same order as specified (regardless of fulfillment order).
- The main promise returned from `Promise.all([..])` will only be fulfilled if and when all its constituent promises are fulfilled. If any one of those promises instead is rejected, the main `Promise.all([ .. ])` promise is immediately rejected, discarding all results from any other promises.
- `Promise.race([ .. ])` expects a single `array` argument, containing one or more Promises, thenables, or immediate values.
- Similar to `Promise.all([..])`, `Promise.race([..])` will fulfill if any Promise resolution is a fulfillment, and it will reject if and when any Promise resolution is a rejection.
- If you pass an empty `array`, instead of immediately resolving, the main `race([..])` Promise will never resolve. So be careful never to send in an empty `array`.
- Consider:

    ```js
    var p1 = request("http://some.url.1/");
    var p2 = request("http://some.url.2/");

    Promise.race([p1, p2])
        .then(function (msg) {
            // either `p1` or `p2` will win the race
            return request(
                "http://some.url.3/?v=" + msg
            );
        })
        .then(function (msg) {
            console.log(msg);
        });
    ```

    Because only one promise wins, the fulfillment value is a single message, not an `array` as it was for `Promise.all([..])`.
- Promises cannot be canceled. So they can only be silently ignored.

## Promise API Recap

- The revealing constructor `Promise(..)` must be used with `new`, and must be provided a function callback that is synchronously/immediately called. This function is passed two function callbacks that acts as resolution capabilities for the promise. We commonly label these `resolve(..)` and `reject(..)`. For example:

    ```js
    var p = new Promise(function (resolve, reject) {
        // `resolve(..)` to resolve/fulfill the promise
        // `reject(..)` to reject the promise
    });
    ```

    If `resolve(..)` is passed an immediaten non-Promise, non-thenable value, then the promise is fulfilled with that value. But if `resolve(..)` is passed a genuine Promise or thenable value, that value is unwrapped recursively, and whatever its final resolution/state is will be adopted by the promise.
- There are two shortcuts for creating an already-rejected and an already-fulfilled Promises. For example:

    ```js
    var p1 = new Promise(function (resolve, reject) {
        reject("Oops");
    });

    var p2 = new Promise(function (resolve, reject) {
        resolve("Done");
    });

    // an already-rejected Promise
    var p3 = Promise.reject("Oops");

    // an already-fulfilled Promise
    var p4 = Promise.resolve("Done");
    ```

    However, `Promise.resolve(..)` also unwraps thenable values. The return value could be a fulfillment or a rejection. For example:

    ```js
    var fulfilledTh = {
        then: function (cb) { c(42); }
    };
    var rejectedTh = {
        then: function (cb, errCb) {
            errCb("Oops");
        }
    };

    var p1 = Promise.resolve(fulfilledTh);
    var p2 = Promise.resolve(rejectedTh);

    // `p1` will be a fulfilled promise
    // `p2` will be a rejected promise
    ```

- Remember to `Promise.resolve(..)` doesn't do anything if what you pass is already a genuine Promise. It just returns the value directly.
- Each Promise instance has `then(..)` and `catch(..)` methods, which allow registering of fulfillment and rejecion handlers for the Promise. Once the Promise is resolved, one or the other of these handlers will be called, but not both, and it will always be called asynchronously.
- `catch(..)` takes only the rejection callback as a parameter, and automatically substitutes the default fulfillment callback. In other word, it's equivalent to `then(null, ..)`. For example:

    ```js
    p.then(fulfilled);

    p.then(fulfilled, rejected);

    p.catch(rejected); // or `p.then(null, rejected)`
    ```

- `then(..)` and `catch(..)` also create and return a new promise, which can be used to express Promise chain flow control.
- The static helpers `Promise.all([..])` and `Promise.race([..])` on the ES6 `Promise` API both create a Promise as their return value.
- For `Promise.all([..])`, all the promises you pass in must fulfill for the returned promise to fulfill. If any promise is rejected, the main returned promise is immediately rejected, too. For fulfillment, you receive an `array` of all the passed in promises fulfillment values. For rejection, you receive just the first promise rejection reason value. This pattern is classically called a **gate**: all must arrive before the gates opens.
- For `Promise.race([..])`, only the first promise to resolve (fulfillment or rejection) **wins**. This pattern is classically called a **latch**: first one to open the latch gets through.
- See these examples of `Promise.all([..])`and `Promsie.race([..])`:

    ```js
    var p1 = Promise.resolve(42);
    var p2 = Promise.resolve("Hello World");
    var p3 = Promise.reject("Oops");

    Promise.race([p1, p2, p3])
        .then(function (msg) {
            console.log(msg); // 42
        });

    Promise.all([p1, p2, p3])
        .then(function (err) {
            console.log(err); // "Oops"
        });

    Promise.all ([p1, p2])
        .then(function (msgs) {
            console.log(msgs); // [42, "Hello World"]
        });
    ```

    Be careful! If an empty `array` is passed to `Promise.all([..])`, it will fulfill immediately, but `Promise.race([..])` will hang forever and never resolve.

## Unwrap/Spread Arguments

- Consider:

    ```js
    function spread(fn) {
        return Function.apply.bind(fn, null);
    }
    Promise.all(foo(10, 20))
        .then(
            spread(function (x, y) {
                console.log(x, y); // 200 599
            })
        );
    ```

    You could inline the functional magic to avoid the extra helper:

    ```js
    Promise.all(foo(10, 20))
        .then(Function.apply.bind(
            function (x, y) {
                console.log(x, y); // 200 599
            },
            null
        ));
    ```

    ES6 has an even better answer for us: **destructuring**. The array destructuring assignment form looks like this:

    ```js
    Promise.all(foo(10, 29))
        .then(function (msgs) {
            var [x, y] = msgs;

            console.log(x, y); // 200 599
        })
    ```

    But best of all, ES6 offers the array parameter destructuring form:

    ```js
    Promise.all(foo(10, 20))
        .then(function ([x,y]) {
            console.log(x, y); // 200 599
        });
    ```

## Promise Uncancelable

- No individual Promise should be cancelable, but it's sensible for a sequence to be cancelable, because you don't pass around a sequence as a single immutable value like you do with a Promise.

## Promise Performance

- Promises are a bit slower than non-Promises. But in exchange you're getting a lot of trustability, non-Zalgo predictability, and composability built in.
- Promises are awesome. Use them. They solve the *inversion of control* issues that plague us with callbacks-only code.

## Generators

- Consider:

    ```js
    var x = 1;

    function *foo() {
        x++;
        yield; // pause!
        console.log("x: ", x);
    }

    function bar() {
        x++;
    }
    ```

    You will likely see most other **JavaScript** documentation/code that will format a generator declaration as `function* foo() { .. }` instead of as I've done here with `function *foo() { .. }`. The only difference being the stylistic positioning of the `*`. The two forms are functionally/syntactically identical, as is a third `function*foo() { .. }` (no space) form. I think `function *foo() { .. }` form is better.
- A generator is a special kind of function that can start and stop one or more times, and doesn't necessarily ever have to finish.
- Consider:

    ```js
    function *foo(x, y) {
        return x + y;
    }

    var it = foo(10, 20);
    var res = it.next();

    console.log(res.value); // 30
    console.log(res.done); // true
    ```

    The result of that `next(..)` call is an `object` with a `value` property on it holding whatever value (if anything) was returned from `*foo()`.
- Consider:

    ```js
    function *foo(x) {
        var y = x * (yield);
        return y;
    }

    var it = foo(5);

    // start `foo(..)`
    it.next();

    var res = it.next(2);

    res.value; // 10
    ```

    You can set a value for `yield` with `next(..)` function. So, at this point, the assignment statement is essentially `var y = 5 * 2`. Now `return y` returns that `10` value back as the result of the `it.next(2)` call.
- There's a mismatch between the `yield` and the `next(..)` call. In general, you're going to have one more `next(..)` call than you have `yield` statement. Why the mismatch? Because the first `next(..)` always starts a generator, and runs to the first `yield`.
- The `yield` is basically asking a question: **What value should I insert here?** you sould answer this question with `next(..)`.
- Always start a generator with an argument-free `next()`. Because the specification and all compliant browsers just silently discard anything passed to the first `next()`.

## Generatoring Values

- The `for..of` loop automatically calls `next()` for each iteration (it does't pass any values in to the `next()`) and it will automatically terminate on receiving a `done: true`. It's quite handy for looping over a set of data.
- Many built-in data structures in **JavaScript** (as of ES6), like `array`s , have default iterators. For example:

    ```js
    var a = [1, 3, 5, 7, 9];

    for (var v of a) {
        console.log(v);
    }
    // 1 3 5 7 9
    ```

    The `for..of` loop asks `a` for its iterator, and automatically uses it to iterate over `a`'s values.
- You can manually itrate an `array` like generators using ES6 `Symbol`. For example:

    ```js
    var a = [1, 3, 5, 7, 9];

    var it = a[Symbol.iterator]();

    it.next().value; // 1
    it.next().value; // 3
    it.next().value; // 5
    ..
    ```

- In a generator, such a loop is generally totally OK if it has a `yield` in it, as the generator will pause at each iteration, `yield`ing back to the main program and/or to the event loop queue.
- Consider:

    ```js
    function *something() {
        var nextVal;

        while (true) {
            if (nextVal === undefined) {
                nextVal = 1;
            }
            else {
                nextVal = (3 * nextVal) + 6;
            }

            yield nextVal;
        }
    }

    for (var v of something()) {
        console.log(v);

        // don't let the loop run forever!
        if (v > 500) {
            break;
        }
    }
    // 1 9 33 105 321 969
    ```

    Why couldn't we say `for (var v of something)..`? Because `something` here is a generator, which is not an iterable. We have to call `something()` to construct a producer for the `for..of` loop to iterate over. The `something()` call produces an iterator, but the `for..of` loop wants an iterable, right? Yep. The generator's iterator also has a `Symbol.iterator` function on it, which basically does a `return this`. In other words, **a generator's iterator is also an iterable**.
- You can use `try..finally` in generators. It will always be run even when the generator is externally completed. This is useful if you need to clean up resources (database, connections, etc). For example:

    ```js
    function *something() {
        var nextVal;

        try {
            while (true) {
                if (nextVal === undefined) {
                    nextVal = 1;
                }
                else {
                    nextVal = (3 * nextVal) + 6;
                }

                yield nextVal;
            }
        }
        // cleanup clause
        finally {
            console.log("cleaning up!");
        }
    }

    for (var v of something()) {
        console.log(v);

        // don't let the loop run forever!
        if (v > 500) {
            break;
        }
    }
    // 1 9 33 105 321 969 cleaning up!
    ```

- You can terminate a generator's iterator manually with `return()` function. For example:

    ```js
    var it = something();
    for (var v of it) {
        console.log(v);

        // don't let the loop run forever!
        if (v > 500) {
            console.log (
                // complete the generator's iterator
                it.return("Hello World").value;
            );
            // no `break` needed here
        }
    }
    // 1 9 33 105 321 969
    // cleaning up!
    // Hello World
    ```

    The `return()` function sets the generator's iterator to `done: true`, so the `for..of` loop will terminate on its next iteration.

## Iterating Generators Asynchronously

- The `yield` allows the generator to `catch` an error. For example:

    ```js
    if (err) {
        // throw an error into `*main()`
        it.throw(err);
    }
    ```

- The `yield`-pause nature of generators means that not only do we get synchronous-looking `return` values from async function calls, but we can also synchronously `catch` errors from those async function calls.
- We can even `catch` the same error that we `throw()` into the generator, essentially giving the generator a chance to handle it but if it doesn't, the iterator code must handle it. For example:

    ```js
    function *main() {
        var x = yield "Hello World";

        // never gets here
        console.log(x);
    }

    var it = main();

    it.next();

    try {
        // will `*main()` handle this error? we'll see!
        it.throw("Oops");
    }
    catch (err) {
        // nope didn't handle it!
        console.log(err); // Oops
    }
    ```

    Synchronous-looking error handling (via `try..catch`) with async code is a huge win for readability and reason-ability.

## Generators + Promises

- The best of all worlds in ES6 is to combine generators (synchronous-looking async code) with Promises (trustable and composable).
- Consider:

    ```js
    function foo(x, y) {
        return request(
            "http://some.url.1/?x=" + x + "&y=" + y
        );
    }

    async function main() {
        try {
            var text = await foo(11, 31);
            console.log(text);
        }
        catch (err) {
            console.log(err);
        }
    }

    main();
    ```

    Here instead of `yield`ing a Promise, we `await` for it to resolve.
- The `async function` automatically knows what to do if you `await` a Promise. It will pause the function (just like with generators) until the Promise resolves.
- The `async/await` is like combining Promises with sync-looking flow control code.
- All of the concurrency capabilities of Promises are available to us in the generator + Promise approach. For example:

    ```js
    function *foo() {
        // make both requests "in parallel," and
        // wait until both promises resolve
        var results = yield Promise.all([
            request("http://some.url.1"),
            request("http://some.url.2")
        ]);

        var r1 = results[0];
        var r2 = results[1];

        var r3 = yield request(
            "http://some.url.3/?v=" + r1 + "," + r2
        );

        console.log(r3);
    }

    run(foo);
    ```

    We can even use ES6 destructuring assignment to simplify the `var r1 = .. var r2 = ..` assignments, with `var [r1, r2] = results`.

## Generator Delegation

- Consider:

    ```js
    function *foo() {
        console.log("`*foo()` starting");
        yield 3;
        yield 4;
        console.log("`*foo()` finished");
    }

    function* bar() {
        yield 1;
        yield 2;
        yield *foo(); // `yield`-delegation!
        yield 5;
    }

    var it = bar();

    it.next().value; // 1
    it.next().value; // 2
    it.next().value; // `*foo()` starting
                     // 3
    it.next().value; // 4
    it.next().value; // `*foo()` finished
                     // 5
    ```

    First, calling `foo()` creates an iterator. Then, `yield *` delegates/transfers the iterator instance control (of the present `*bar` generator) over to this other `*foo()` iterator. So, the first two `it.next()` calls are controlling `*bar()`, but when we make the third `it.next()` call, now `*foo()` starts up, and now we're controlling `*foo()` instead of `*bar()`. That's why it's called delegation (`*bar()` delegated its iteration control to `*foo()`).
- You could even use `yield`-delegation for async-capable generator **recursion** (a generator `yield`-delegating to itself). For example:

    ```js
    function *foo(val) {
        if (val > 1) {
            // generator recursion
            var = yield *foo(val -1);
        }

        return yield request("http://some.url/?v=" + val);
    }

    function *bar() {
        var r1 = yield *foo(3);
        console.log(r1);
    }

    run(bar);
    ```

- Thunks do not in and of themselves have hardly any of the trustability or composability guarantees that Promises are designed with.

## Web Workers

- Imagine splitting your program into two pieces, and running one of those pieces on the main UI thread, and running the other piece on an entirely separate thread.
- **JavaScript** does not currently have any features that support threaded execution. But an environment like your browser can easily provide multiple instances of the **JavaScript Engine**, each on its own thread, and let you run a different program in each thread. Each of those separate threaded pieces of your program is called a **(Web) Worker**.
- Consider:

    ```js
    var w1 = new Worker("http://some.url.1/mycoolworker.js");
    ```

    The URL should point to the location of a **JavaScript** file (not an HTML page!) which is intended to be loaded into a Worker.
- Workers do not share any scope or resources with each other or the main program. But instead have a basic event messaging mechanism connecting them. Here's how to listen for events (actually, the fixed `"message"` event):

    ```js
    var w1 = new Worker("http://some.url.1/mycoolworker.js");

    w1.addEventListener("message", function (evt) {
        // evt.data
    });
    ```

    And you can send the `"message"` event to the Worker:

    ```js
    w1.postMessage("something cool to say");
    ```

    Inside the Worker, the messaging is totally symmetrical:

    ```js
    // "mycoolworker.js"

    addEventListener("message", function (evt) {
        // evt.data
    });

    postMessage("a really cool reply");
    ```

- To kill a Worker immediately from the program that created it, call `terminate()` on the Worker object.
- Terminating a Worker thread does not give it any chance to finish up its work or clean up any resources. It's akin to you closing a browser tab to kill a page.
- Worker has access to its own copy of several important global variables/features, including `navigator`, `location`, `JSON` and `applicationCache`.
- You can load extra **JavaScript** scripts into your Worker, using `importScripts(..)`. For example:

    ```js
    // inside the Worker
    importScripts("foo.js", "bar.js");
    ```

- You can use Web Workers for:
  - Processing intensive math calculations.
  - Sorting large data sets.
  - Data Operations (compression, audio analysis, image pixel manipulations, etc).
  - High-traffic network communications.
- You can create a single centralized Worker that all the page instances of your site or app can share is quite useful. That's called a `SharedWorker`. For example:

    ```js
    var w1 = new SharedWorker("http://some.url.1/mycoolworker.js");
    ```

- Because a shared Worker can be connected to or from more than one program instance or page on your site, the Worker needs a way to know which program a message comes from. So the calling program must use the `port` object of the Worker for communication. For example:

    ```js
    w1.port.addEventListener( "message", handleMessages );

    // ..

    w1.port.postMessage("something cool");
    ```

    Also, the port connection must be initialized, as:

    ```js
    w1.port.start();
    ```

- Shared Workers survive the termination of a port connection if other port connections are still alive, whereas dedicated Workers are terminated whenever the connection to their initiating program is terminated.
- Workers are an API and not a syntax, they can be polyfilled, to an extent.
- If a browser doesn't support Workers, there's simply no way to fake multithreading from the performance perspective.

## SIMD

- With SIMD (**S**ingle **I**nstruction, **M**ultiple **D**ata), threads don't provide the parallelism. Instead, modern CPUs provide SIMD capability with **vectors** of numbers.
- The performance benefits for data-intensive applications (signal analysis, matrix operations on graphics, etc) with such parallel math processing are quite obvious.
- SIMD first provided by **Intel**, by **Mohammad Haghighat**, in cooperation with Firefox and Chrome teams.

## asm.js

- Early versions of the asm.js experiment required a `"use asm";` pragma (similar to strict mode's `"use strict";`) to help clue the **JavaScript** engine to be looking for asm.js optimization opportunities and hints.
- asm.js is intended to provide an optimized way of handling specialized tasks such as intensive math operations (e.g., those used in graphics processing for games).
- asm.js could be hand authored, but that's extremely tedious and error prone, akin to hand authoring assembly language (hence the name).
- asm.js would be a good target for cross-compilation from other highly optimized program languages. For example, Emscripten transpiling C/C++ to **JavaScript**.

## Benchmarking

- The vast majority of **JavaScript** developers, if asked to benchmark the speed (execution time) of a certain operation, would initially go about it something like this:

    ```js
    var start = (new Date()).getTime(); // or `Date.now()`

    // do some operation

    var end = (new Date()).getTime();

    console.log("Duration: ", (end - start));
    ```

    Don't do like this.
- If you repeat an operation 100 times, and that whole loop reportedly takes a total of 137ms, then you can just divide by 100 and get an average duration of 1.37ms for each operation, right? Well, not exactly. Don't do it.
- You are not actually qualified to write your own benchmarking logic.
- Use **Benchmark.js** tool to test your program execution speed. Here's how you could use Benchmark.js to run a quick performance test:

    ```js
    function foo() {
        // operation(s) to test
    }

    var bench = new Benchmark(
        "foo test",      // test name
        foo,             // function to test (just contents)
        {
            // ..        // optional extra options (see docs)
        }
    );

    bench.hz; // number of operations per second
    bench.stats.moe; // margin of error
    bench.stats.variance; // variance across samples
    ```

    If you're going to try to test and benchmark your code, this library is the first place you should turn.
- Consider:

    ```js
        // Case 1
        var x = false;
        var y = x ? 1 : 2;

        // Case 2
        var x;
        var y = x ? 1 : 2;
    ```

    There is extra work to do the coercion in the second case. So always use first case.
- Consider these three `for` loops:

    ```js
    // Option 1
    for (var i = 0; i < 10; i++) {
        console.log(i);
    }

    // Option 2
    for (var i = 0; i < 10; ++i) {
        console.log(i);
    }

    // Option 3
    for (var i = -1; ++i < 10; ) {
        console.log(i);
    }
    ```

    These sort of obsession is basically nonsense in modern **JavaScript**. But you might do some tricks to optimize the performance. For example:

    ```js
    var x [..];

    // Option 1
    for (var i = 0; i < x.length; i++) {
        // ..
    }

    // Option 2
    for (var i = 0, len = x.length; i < 0; i++) {
        // ..
    }
    ```

    If you run performance benchmarks around `x.length` usage compared to caching it in a `len` variable, you'll find that while the theory sounds nice. But don't try to outsmart your **JavaScript Engine**, you'll probably lose when it comes to performance optimizations.
- Don't pass the `arguments` variable from one function to any other function, as such **leakage** slows down the function implementation.
- You shouldn't necessarily change a piece of code to work around one engine's difficulty with running a piece of code in an acceptably performant way.
- Which one if any is the fastest?

    ```js
    var x = "42"; // need number `42`

    // Option 1: let implicit coercion automatically happen
    var y = x / 2;

    // Option 2: use `parseInt(..)`
    var y = parseInt(x, 0) / 2;

    // Option 3: use `Number(..)`
    var y = Number(x) / 2;

    // Option 4: use `+` unary operator
    var y = +x / 2;

    // Option 5: use `|` unary operator
    var y = (x | 0) / 2;
    ```

    If `x` can ever be a value that needs parsing, shuch as `"42px"` (like from a CSS style lookup), then `parseInt(..)` really is the only suitable option. `Number(..)` is also a function call. From a behavioral perspective, it's identical to the `+` unary operator option, but it may in fact be a little slower, requiring more machinery to execute the function.
- If you write recursive functions without tail calls, the performance will still fall back to normal stack frame allocation, and the **Engines**' limits on such recursive call stacks will still apply.
- jsPerf.com is a fantastic website for crowdsourcing performance benchmark test runs.
- TCO (**T**ail **C**all **O**ptimization) allows a function call in the tail position of another function to execute without needing any extra resources, which means the engine no longer needs to place arbitrary restrictions on call stack depth for recursive algorithms.
