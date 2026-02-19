# Introduction

If you have spent a significant amount of time talking to Haskellers, or to PL researchers (particularly, but not only, the ones interested in effect systems), you've probably heard them extol the virtues of CPS. No, they aren't talkin about Child Protective Services. Even Haskellers don't (usually) think we should save children from all this imperative code. They are talking about Continuation Passing Style, a very interesting topic that I feel more people should at least know about: its expressive power can do everything traditional programming can[^1] and more.

Now, I hear your thoughts[^2]: "T-Dark, what in transistors _is_ CPS, actually?". Thank you for thinking about asking. CPS is what happens if you decide that `return`, and in general the operation of returning from a function, is _cringe_ and must be _eradicated_, and then go from there to figure out how you can still manage to write useful code without putting literally everything in `main`

# What even is a continuation?

Another excellent question. A continuation is a rather abstract concept. Don't worry if you don't really understand what I'm describing here, it'll probably become clearer as you keep reading this post. 

While you execute your program, at any given point in time, control flow is _somewhere_. Based on the current position of control flow, you can divide your program between its past and its future: what has already happened and what will (or could) happen later. A continuation, at a given point in time, is the future of your program based on that point. In other words, a continuation is _the rest of your program_. Everything you still have to do.

This is a rather oblique concept at first. To explain it better, as well as to prove that CPS can do everything you're used to, allow me to show you how to translate traditional code into CPS form.

Before I start, a warning: I will be showing you a very _purist_ take on CPS: Normally you wouldn't _completely eliminate_ return values, you'd just avoid using them when CPS is a better alternative. Additionally, CPS is very rarely completely handwritten: usually you'd use a macro or some compiler feature to get prettier syntax. Handwritten CPS is _really ugly_. Nonetheless, its sheer power makes it useful to know.

# Straight-line code

To start, we need to figure out how to replace code like `let x = 2 + 2;` This may _seem_ like innocent code, but in reality it's equivalent to `let x = add(2, 2);`, which only works because the `add` function _returns a value_. This, we cannot allow. So instead, let's introduce the core trick (really, the _only_ trick) of CPS. Let's pass this function its continuation. `let x = add(2, 2);` does not exist. Instead, we shall change how `add` is defined so that we can write `add(2, 2, |x| { ... })`.

See that closure? That's the continuation after `add`: that closure must contain _the entire rest of our program_. Everything that you wish to ever happen after this one addition must be inside that closure. It can't be outside: `add` does not return (and neither does the closure), so once we enter it we will never exit it.

In terms of type system (expressed using Rusty pseudocode), the signature of `add` has gone from `fn add(a: Number, b: Number) -> Number` to `fn add(a: Number, b: Number, cont: fn(Number) -> !) -> !`.  
Note that since no function will ever return, we could omit the `-> !` and let it be implicit. To avoid accidentally making it look like any of these functions return, this blog post will write `-> !` after every function.

# The entry and exit points

Every programming language needs an entry point, where control flow starts when your program begins, and an exit point, where control flow ends to terminate your program. Conventionally, the entry point is the start of `main` and the exit point is... returning from `main`.

Fear not, we have a solution. `main` will just need, like every other function, to take a continuation. The continuation passed to `main` is a function which, when invoked, ends your program. This is the noop program:
```rs
fn main(exit: fn() -> !) -> ! {
    exit()
}
```
And this is hello world:
```rs
fn main(exit: fn() -> !) -> ! {
    print("Hello, World!", exit)
}
```
(Remember that, for all zero-argument functions `f`, `|| f()` does the same thing as `f`, and in general `|args...| f(args...)` does the same thing as `f` regardless of the amount of arguments)  
We call `print`, which will of course never return, so we need to pass it `exit` as a continuation so it can call it for us.

# Return

Yes, I did just say we're removing `return` and replacing it with a continuation. However, from a point of view _inside_ a function, we have a very interesting property: the continuation looks a lot like `return`. Consider this example:
```rs
fn example(cont: () -> !) -> ! {
    print("owo", cont)
}
```
Think about it: when you (make `print`) call `cont`, you go on to executing the rest of the program after `example` is done. In particular, _what_ this program is isn't your choice: it's decided by your caller. This is _exactly_ like the `return` keyword conventionally works!  
Indeed, for all functions that take a single continuation, that continuation can be thought of as _the return keyword_. `return` is a function now. Sometimes it's helpful to think of calling a continuation as returning: it makes CPS easier to relate to conventional code.

## Practical applications 1: multi-level return

Have you ever written a function that contains a really long code block and then realized this block would work better in a function of its own?
```rs
// turn this
fn long() {
    setup()
    {
        //really long code block
    }
    teardown()
}

// into this
fn long() {
    setup()
    code_block()
    teardown()
}
fn code_block() {
    //really long code block
}
```
This refactoring is impossible if the really long code block happens to use the `return` keyword: after the refactoring, it returns from `code_block`, but we'd really rather it return from `long`. Making `return` a function fixes this problem: we can now refactor like this
```rs
// this code block is not fully written in CPS, since it's about something you might want to _actually handwrite in real code_. It does, however, use the CPS concept of replacing return with a continuation

// turn this
fn long(ret: fn() -> !) {
    setup()
    {
        // pile of code before
        ret()
        // pile of code after
    }
    teardown()
}

// into this
fn long(ret: fn() -> !) {
    setup()
    code_block(ret)
    teardown()
}
fn code_block(long_ret: fn() -> !) {
    // pile of code before
    long_ret()
    // pile of code after
}
```

# If

On the topic of nontrivial control flow, let's introduce `if`. `if` is now a function. We didn't _have_ to make it a function, but we _can_, and doing so lets me showcase a powerful feature of CPS: `if` does not need to be special syntax with unique effects on the control flow of your code that could only be implemented by the compiler. The mechanics of branching are still going to be a compiler builtin (which is why I won't show an implementation)[^3], but the control flow effects follow naturally from how _all_ functions work.

Here's the signature:
```rs
fn if(cond: Bool, then: fn() -> !, else: fn() -> !) -> !
```
In practice, you could use it like this:
```rs
fn main(exit: fn() -> !) -> !{
    read_number_from_stdin(|number| {
        eq(number, 10, |b| {
            if(b,
                || print("b is equal to 10", exit)
                || print("b is not equal to 10", exit)
            )
        })
    })
}
```
I did warn you handwritten CPS is really ugly. The fact I can't nest `number == 10` (or at least `eq(number, 10)`) inside the `if` really hurts, not to mention the indentation. That aside, you can hopefully see the principle: `if` chooses which closure to run based on whether `cond` is `true` or `false`, and then it runs it.

Another interesting aspect is that these closures both call `exit` (or rather, both tell `print` that its continuation is `exit`) This happens naturally whenever both branches of an `if` want to "merge back" into running the same code: they can't just _return_ to it, so they simply both call the same continuation.

Lastly, I'll call your attention to the fact that, despite the fact `if` is not even a generic function, this _is_ a full-fledged `if` expression: you can translate this code
```rs
let x = if cond { "owo" } else { "uwu" };
print("example: {x}");
```
into this code
```rs
let cont = |x| print("example: {x}");
if(cond, || cont("owo"), || cont("uwu"))
```
Again, I did warn you handwritten CPS is ugly. Why yes, you do have to write the continuation _before_ the `if` despite the fact it runs _after_ it. That's just how variable scoping works. In a language built around CPS, you could introduce some dedicated syntax, perhaps inspired by Haskell's `where`[^4], but otherwise you're out of luck.

# For

Have no fear, this section is long, but that's just because it's the best place to introduce a number of useful concepts. None of them is too big.

We've introduced straight line code and branching. We should now introduce loops. Oddly enough, the most interesting loop in this post shall be `for`. Before we can get to `for`, however, we need to introduce iterators. We'll only need to care about one function:
```rs
fn next<T>(iter: Iterator<T>, on_elem: fn(T, Iterator<T>) -> !, on_empty: fn() -> !) -> !
```
This is a functional programmer's take on `next`[^6]. It takes two continuations because there are two cases: either the iterator has an element or it's empty. In the first case, our continuation is passed the element and the rest of the iterator, and in the second case we get nothing.

Now, we can go ahead to invent `for`:
```rs
fn for<T>(iter: T, body: fn(T, fn() -> !) -> !, cont: fn() -> !) -> ! {
    next(iter,
        |elem, rest| body(elem, || for(rest, body, cont)),
        cont,
    )
}
```
You could call it like this
```rs
fn main(exit: fn() -> !) -> ! {
    range(0, 10, |range| { 
        for(range, |elem, cont| {
                print(elem, cont), 
            },
            exit
        )
    })
}
```
This code would print all numbers from 0 to 10.

Now that's a signature, alright. While we're here before I explain what I've just done, notice that this is a recursive function: it refers to itself inside (a lambda inside a lambda inside) itself. That's because I'm writing `for` like a functional programmer would[^5]: recursively instead of iteratively, in order to show all the interesting control flow achieved by CPS.

For starters, let's get the easy part out of the way: `cont` is the equivalent of returning from the `for` function, and if the iterator is empty `for` wants to return, so that's exactly what we set up: if the iterator's `on_empty` continuation runs, it runs `cont`, thus "returning from"`for`.

The other branch is a bit more interesting. Let's take it step by step.
- For starters, our arguments are the current element and the rest of the iterator, so we need a closure that accepts them: `|elem, rest|`
- If there is one element, the loop body should run once, giving it access to that element. `body` is the closure representing the loop body, so we get `|elem, rest| body(elem)`
- Once `body` finishes, it needs to have a way to continue to the next iteration. That's why it takes a continuation. This gives us `|elem, rest| body(elem, || ...)`
- Calling `for` once (on an iterator with an element) runs `body` once, so to run `body` again we can just call `for` again with the same `body`. The iterator needs to be replaced by _the rest_ of the iterator so we don't process `elem` again, and the "return" continuation, `cont`, is still the same: `|elem, rest, body(elem, || for(rest, body, cont))`

That's it, that's everything. We have implemented a perfectly usable `for` loop.

This is a somewhat sad `for` loop, though. Where's `break` and `continue`? Turns out, we actually already have `continue`, and `break` is just around the corner.

To see `continue`, start by imagining a language in which ending a `for`'s body without the `break` or `continue` keyword is an error:
```rs
// ERROR
for elem in iter {
    process(elem)
}
```
You can restore the behaviour you're used to by adding `continue` at the end of every loop body:
```rs
// OK
for elem in iter {
    process(elem);
    continue
}
```
Pay close attention to the `for` loop example earlier. Can you see anything interesting we do at the end of every loop body?  
Why yes, we call `cont`. Can you imagine what would happen if we sometimes called `cont` _earlier_, like this?
```rs
fn main(exit: fn() -> !) -> ! {
    range(0, 10, |range| { 
        for(range, |elem, cont| {
                eq(elem, 5, |b| {
                    if(b, cont, print(elem, cont))
                })
            }
            exit
        )
    })
}
```
(this code calls `cont` without `print` if `elem == 5`)  
Calling `cont` causes us to proceed to the next iteration of the `for`. We can call `cont` at any time, including _instead of_ running other code inside the loop body. Would you look at that, `cont` _is_ `continue`!

As for `break`, that one will actually require us to change the `for` function. Before we do that, think about `break`. Its purpose is to let the loop body skip the rest of the loop, right? Another way to say that would be that its purpose is to let the continuation of the loop body be the continuation of the loop. If thinking with continuations is still difficult, think of it as "jumping" to the code after the loop.

Huh. `for` _has_ the continuation of the loop, it's the last argument. All we need to do is to pass it to `body`, so it can be called by `body` as well. Let's rename it too, for clarity.
```rs
fn for<T>(iter: T, body: fn(T, fn() -> !, fn() -> !) -> !, break: fn() -> !) -> ! {
    next(iter,
        |elem, rest| body(elem, break, || for(rest, body, cont))
        || break(),
    )
}
```
In this implementation, the continuations we pass to `body` are `break` and `continue` in that order. We could also put them in the other order, as long as we fixed all callsites of `for` to match.

In conclusion, remember that just because `body` takes `break` and `continue` doesn't mean it's forced to call either. It could call _some other_ continuation it got as a closure capture. Here's an example
```rs
fn main(exit: fn() -> !) -> ! {
    range(0, 10, |range| { 
        for(range, |elem, break, continue| {
                eq(elem, 5, |b| {
                    if(b, exit, print(elem, continue))
                })
            },
            || print("10", exit)
        )
    })
}
```
This code never prints `6` nor `10`. It exits before that can happen.

## Practical applications 2: `break` and `continue` across functions

In the last "practical applications" section, we introduced multi-level return. I'm happy to announce this also works with `break` and `continue`, which means you can always extract loop bodies (or parts thereof) into their own functions:
```rs
// again, this is not fully CPS, but does use CPS concepts

// turn this
```rs
fn example(iter: Iterator<Foo>) {
    for(iter, |elem, break, continue| {
        // code
        if condition { break() }
        // more code
        if other_condition { continue() }
        // additional code
    })
}

// into this
fn example(iter: Iterator<Foo>) {
    for(iter, loop_body)
}
fn loop_body(elem: Foo, break: fn() -> !, continue: fn() -> !) {
    // code
    if condition { break() }
    // more code
    if other_condition { continue() }
    // additional code
}
```

# While and Loop

`for` introduced all of the interesting concepts about loops, but for completeness we might as well add the other two common kinds of loop.
```rs
fn while(condition: fn(fn(bool) -> !) -> !, body: fn(fn() -> !, fn() -> !) -> !, ret: fn() -> !) -> !{
    condition(|b| if(b, {
        || body(ret, || while(condition, body, ret)),
        ret,
    }))
}
```
Remember that `body` takes the `break` continuation and then the `continue` one. The principle is the usual one: `while` runs its condition and gets a boolean out if it. If it's `true`, it runs the body, whose `continue` runs the next iteration of `while`. If it's `false`, it just returns.
```rs
fn loop(body: fn(fn() -> !)) -> ! {
    body(loop)
}
```
`loop` is interesting: it does not accept a return continuation. This should make sense, since it's an _infinite_ loop. Likewise, the loop body only receives `continue` as an argument, not `break`. You might point out that you can `break` out of a `loop`, and you'd be right. But you don't need _the loop itself_ to give you a continuation that does that. You can define it externally and store it inside `body` as a closure capture. The other loops only need a return continuation because sometimes _they_ call it. Besides, it's not like `loop` can define it for you: you'd just end up having to pass it as an argument to `loop` just to get it back as an argument to `body`.

With that, we have introduced all the important primitives for control flow. As an exercise for the reader, you might find it interesting to look very closely at how `next` was defined in the `for` section and think about how you could replace `match` with a function that takes continuations too. Just note every enum will need its own version of this function, the rust-y typesystem I'm assuming doesn't have the features to do otherwise.

# Exceptions

What, you thought we were done? We're just getting to the interesting part! Continuations are not limited to just _traditional_ control flow! 

We've already had a taste of power when we implemented multi-level `return`, `break`, and `continue`. Now let's go even further and implement `try`, `catch`, and `throw`.

It turns out, they're all part of the same function:
```rs
fn try_catch<E, T>(try: fn(fn(E) -> !, fn(T) -> !) -> !, catch: fn(E) -> !, ret: fn(T) -> !) -> ! {
    try(catch, ret)
}
```
You would use it like this
```rs
try_catch(
    |throw, ret| {
        if(something_goes_wrong, || throw(some_value), || ret(some_other_value))
    },
    |exception| handle(exception),
    |success_value| rest_of_your_program(success_value),
)
```
Most of the complexity here is in the signature. Let's take a closer look:
- `try` is the code block we are attempting to run. It can return successfully (that's what the second continuation is for), or it can throw (by calling the first continuation)
- `catch` and `throw` are the continuation that should be called on throw. Also, yes, they are literally the same thing. You can think of a continuation as a way to "jump" to a different location in your source code. When you `throw` something, control flow should jump to the matching `catch`, so it should make sense that `throw` _is_ `catch` in CPS.
- `ret`, as usual, is the rest of your program.

Now, the traditional try-catch construct allows multiple `catch`es, and also allows you to decide whether to catch an exception based on its type. This is a slightly more primitive construct: If you want to be able to throw multiple kinds of exception and handle them differently, you're free to throw an enum and `match` it inside the `catch` continuation. If you were using CPS to implement exceptions, you would probably want to include this convenience feature in the desugaring of `try` blocks to CPS code.  
Also, normally `throw` is just something you can _do_, at any time. In this implementation, you can only throw if you have a throw continuation. This is a feature: it means you cannot throw an uncaught exception. That said, if you wanted to, you could wrap your entire program inside a `try_catch` and pass around the nearest `try_catch`'s `throw` as an implicit argument. The top-level `throw` just exits your program printing something like "uncaught exception".

Lastly, allow me to point out an interesting fact: the `try_catch` function is unnecessary.  
No, really. All it does is take three arguments and pass the last two to the first. You don't need it. You could have written the code earlier as follows:
```rs
(|throw, ret| {
    if(something_goes_wrong, || throw(some_value), || ret(some_other_value)0)
})(
    |exception| handle(exception),
    |success_value| rest_of_your_program(success_value),
)
```
This would look a lot better if, rather than calling a closure you defined on the spot, you were calling a function:
```rs
some_function(
    |exception| handle(exception),
    |success_value| rest_of_your_program(success_value),
)
```
Of course, just because a function does something really trivial doesn't make it useless. Ask your nearest Haskeller to tell you why `id = |x| x` and `apply = |f, x| f(x)` are useful functions. If you're handwriting CPS you will probably find that `try_catch` is worth defining. If you're generating it automatically, perhaps as transform your compiler runs on its input, you may or may not think so.

# Contexts

We've had exceptions, in which you jump out of a function to some handler code. They're good, but we can do better. How about jumping out of a function _and then jumping back in, resuming where you left off?_

The most common application of this technique in mainstream programming languages is found in generators and coroutines, but it turns out those are quite a bit more complex than exceptions. So, before we go there, let's introduce a weaker concept: _resumable_ exceptions. You can jump out of a function and go to a handler, and then you can jump back in, possibly receiving a value from the handler as you do so. A practical application of this technique is DI: You jump from wherever you are out to the DI provider asking it for a resource, and the provider jumps back into your code giving you a resource. For the sake of simplicity, let's hardcode the _a_: you can get _one_ resource (though you can get it as many times as you want).

Introducing the functions `get`, which does exactly this, and `provide`, which lets you introduce a scope in which you have access to a resource provider.
```rs
fn get<T>(provider: fn(fn(T) -> !) -> !, resume: fn(T) -> !) -> ! {
    provider(resume)
}
fn provide<T>(resource: T, scope: fn(fn(T) -> !, fn() -> !) -> !, ret: fn() -> !) -> ! {
    scope(|| resource, ret)
}
```
Have an example:
```rs
fn foo(ret: fn() -> !) -> ! {
    provide(clock, |provider| {
        bar(provider, ret)
    })
}
fn bar(provider: fn(fn(T) -> !) -> !, ret: fn() -> !) -> ! {
    get(provider, |clock| get_current_time(clock, |time| print(time, ret)))
}
```
This code prints the current time.

Calling `get(provider)` effectively jumps to where `provider` was _defined_ (which is to say inside `provide`), gets the provider from there, and then jumps _back_ to `bar`, specifically to the closure starting with `|clock|`.

Of course, you could, and rightfully should, point out that this is a remarkably convoluted way to pass one argument to a function. You're not wrong. However, it's also nearly infinitely expandable. Imagine the above functions with slightly more powerful signatures:
```rs
fn get<T>(provider: fn(Number, fn(T) -> !) -> !, resume: fn(T) -> !) -> !;
fn provide<T>(resource: fn(Number, fn(T) -> !) -> !, scope: fn(fn(T) -> !, fn() -> !) -> !, ret: fn() -> !) -> !;
```
the `Number` stands for a resource ID, letting you identify _which_ resource you want. Now imagine _these_ signatures:
```rs
fn get<T>(provider: fn(fn(AnyMap) -> !) -> !, resume: fn(T) -> !) -> !;
fn provide(resources: AnyMap, scope: fn(fn(AnyMap) -> !, fn() -> !) -> !, ret: fn() -> !) -> !;
```
Think of an `AnyMap` as a `HashMap` that lets you store values of different types, one per type. You can now make a single provider that stores a `Clock`, an `Rng`, a `DatabaseConnection`, or whatever else you want, and conveniently access these resources via `get`. Better yet, it's extremely mockable, even nestably so: at literally any time, you can make up a new `AnyMap`, call `provide`, and now you have a new provider! Furthermore, with some additional compiler support, you could even make the provider an implicit argument, which would result in excellent ergonomics for this and many other use cases. Think about how many problems you could solve more conveniently if you had an easy way to pass necessary-but-unwieldy arguments around. And you can make it as statically or as dynamically typed as you please!

You can go even further: I've written `provide` to take a resource (or a collection thereof), but you could make it take an function that computes a resource on the fly, in as complex a way as it wants.

With the easy case of "jump out of a function and back in" handled (in which the _caller_ provides and the _callee_ gets), lets deal with the hard case (in which the _callee_ provides and the _caller_ gets). Let's talk about generators.

# Generators

Generators, if you haven't used a language that has them, are a convenient way to define iterators. You can write normal-looking code and sprinkle in the `yield` keyword (typically called that in all languages with generators). Every time your code reaches a `yield`, the iterator you're defining returns whatever value you yielded. For example, this generator yields even numbers from 0 to 10 exclusive, and then yields 99:
```rs
fn example() -> Generator<T> {
    for i in 0..10 {
        if i % 2 == 0 {
            yield i;
        }
    }
    yield 99;
}
```
You can use it in a `for` loop, like this:
```rs
for elem in example() {
    print(elem)
}
```
this code will print `0`, `2`, `4`, `6`, `8`, and `99`, in that order.

Running a generator is interesting. Conventionally, you call a function and get a generator object, which you can use as an iterator. Let's implement something like that: our generator will be something with a `next` function. We also, of course, should have a `yield` function, so we can _create_ a generator.
```rs
struct Generator<T>(fn(fn(T, Generator<T>) -> !, fn() -> !) -> !);
fn yield<T>(val: T, handler: fn(T, Generator<T>) -> !, resume: fn(fn(T, Generator<T>) -> !, fn() -> !) -> !) -> ! {
    Generator(resume, |g| handler(val, g))
}
fn next<T>(gen: Generator<T>, on_elem: fn(T, Generator<T>) -> !, on_empty: fn() -> !) -> ! {
    gen.0(|g| g(on_elem, on_empty))
}
```
Note that I've CPSd the `Generator` constructor (so it takes its one argument and then the continuation to pass the wrapped argument to) and the notion of field access (which is why `.0` takes a function: it calls it with the field's value). You could argue field access doesn't _return_ anything, since it's not a function, but I'm being as much of a purist as possible.

Here's an example. In the following, assume `unreachable` is a function that crashes the program if it ever runs: it won't matter, since it won't run.
```rs
// This function yields `1`, then `2`, then `3`, then it "returns"
fn generator(handler1: fn(Number, Iterator<Number>) -> !, _ret: fn() -> !) -> ! {
    yield(1, handler1, |handler2, _| {
        yield(2, handler2, |handler3, _| {
            yield(3, handler3, |_, ret3| {
                ret3()
            })
        })
    })
}
// This one prints "one: 1", then "two: 2", then "three: 3", and finally "done".
fn consume(exit: fn() -> !) {
  Generator(generator, |gen| next(gen, 
        |one, rest| print("one: {one}", || next(rest,
            |two, rest| print("two: {two}", || next(rest,
                |three, rest| print("three: {three}", || next(rest
                    unreachable
                    || print("done", exit)
                )),
                unreachable
            )),
            unreachable
        )),
        unreachable
    )
  ) 
}
```
To explain the above, let's start from the `Generator` type definition:
```rs
struct Generator<T>(fn(fn(T, Generator<T>) -> !, fn() -> !) -> !);
```
A generator is a function which accepts two continuation:
- The first one is called whenever an element is yielded and receives two arguments: the element and the rest of the generator. Interestingly, the "rest of a generator" is _itself_ a generator (since it can continue to yield more elements), so the type of this continuation is `fn(T, Generator<T>) -> !`
- The second one is calld when the generator runs out of elements. When it does so, it simply wants to "return" in the same way as any other function would do, and has no particular value to return. The return continuation thus has type `fn() -> !`.

While you're at it, you might be wondering why I even _have_ a `Generator` struct: it would be most obvious if a generator was _a function_: the entire purpose of the struct appears to be to force you to wrap a generator function in it before you can use it. Unfortunately, this would require a really unfortunate type: `type Generator<T> = fn(fn(T, Generator<T>) -> !, fn() -> !) -> !`. This is a recursive type alias. For reasons beyond the scope of this blog post, typesystems almost always refuse to support recursive type aliases due to them introducing a lot of complexity for very little gain. To work around this limitation, we upgrade `Generator` into a full-fledged type by making it a struct, which introduces the need to wrap generator functions before they can be used as proper generators.[^7]

Once we understand what a `Generator` _is_, it should be much easier to understand the other two functions. We start from `yield`:
```rs
fn yield<T>(val: T, handler: fn(T, Generator<T>) -> !, resume: fn(fn(T, Generator<T>) -> !, fn() -> !) -> !) -> ! {
    Generator(resume, |g| handler(val, g))
}
```
`yield` takes three arguments:
- the value to be yielded.
- The handler that will process this value and probably continue by resuming the rest of the generator. 
- The rest of the generator: you may notice the type of `resume` is the same as the type of `Generator` minus the wrapper struct. This is purely for user convenience, `yield` wraps it so you don't have to wrap it at callsites.

Now let's see `next`:
```rs
fn next<T>(gen: Generator<T>, on_elem: fn(T, Generator<T>) -> !, on_empty: fn() -> !) -> ! {
    gen.0(|g| g(on_elem, on_empty))
}
```
`next` unwraps the `Generator` to access the actual function, then simply calls it with the two handlers for the two cases. Remember `next` from the chapter about `for`? It's exactly the same as this function. That chapter just used the word `Iterator` instead of `Generator` to name the type.

The end result of this structure is that if you try to follow the control flow of a generator, you end up with an alternating structure: the generator and the consumer keep "jumping" into each other. For example, we could use the `generator` and `consume` functions defined above for an example. Ignoring unimportant closures, control flow looks like this:
- you call `consume`
- `consume` calls `generator`, passing it the rest of `consume` and `unreachable` as the two continuations.
- `generator` calls the rest of `consume`, passing it `1` and the rest of itself
- `consume` calls the rest of `generator` _again_, passing it the rest of `consume` and `unreachable`
- `generator` calls the rest of `consume`, passing it `2` and the rest of itself
- `consume` calls the rest if `generator`, passing it the rest of `consume` and `unreachable`
- `generator` calls the rest of `consume`, passing it `3` and the rest of itself
- `consume` calls the rest of `generator`, passing it `unreachable` and the final closure
- `generator` calls the final closure, _without_ passing it the rest of itself. The generator is done.

Indeed, at a high level a generator is just a way to write two functions that keep jumping into each other, without having to hardcode each function in the other one.

While we're here, remember what I said at the start of this paragraph?
> Conventionally, you call a function and get a generator object

The code I defined above _doesn't_ construct a generator object by calling a function (remember that the `Generator` struct is just an uninteresting workaround). Rather, the function itself _is_ a generator object: `generator` isn't a function that "returns" a generator. It is the generator _itself_. Calling it doesn't create a fresh new generator, it runs the generator until its first yield.  
If you prefer a mental model in which a generator is constructed by calling a function, you can still easily obtain that. Just write a wrapper function that returns `generator` and call _that_.

# Async (Ok, actually coroutines)

Yup. CPS can do async too. It turns out it can do just about everything: CPS can implement algebraic effects, and algebraic effects can handle practically all forms of exciting control flow.
That said, async is an _extremely general_ concept which simultaneously admits a billion different implementations and makes people think of a single use case. Let's implement coroutines instead, and then I'll point out how you could use them to write async code.

Coroutines are like generators, except when you `yield` you can be resumed _with a value_. In some sense, they are bidirectional generators: you give a value to the caller, the caller gives a value to you.  
Additionally, since this is a generalization of generators, let's generalize some more: Coroutines can yield values, accept resume arguments, and also return a final value, not necessarily of the same type as the ones they were yielding.

Like async, we can build coroutines on top of `yield` and `next`, with slightly different signatures. Also, a type alias, because wow will we be writing a long type today:
```rs
type UnwrappedCoroutine<Y, R, N> = fn(
    N, 
    fn(Y, Coroutine<Y, R, N>) -> !, 
    fn(R) -> !,
) -> !;

struct Coroutine<Y, R, N>(UnwrappedCoroutine<Y, R, N>);

fn yield<Y, R, N>(
    val: Y, 
    handler: fn(Y, Coroutine<Y, R, N>) -> !, 
    resume: UnwrappedCoroutine<Y, R, N>,
) -> ! {
    Coroutine(resume, |coro| handler(val, coro))
}

fn next<Y, R, N>(
    coro: Coroutine<Y, R, N>, 
    arg: N, 
    on_yield: fn(Y, Coroutine<Y, R, N>) -> !, 
    on_return: fn(R) -> !,
) -> ! {
    coro.0(|c| c(arg, on_yield, on_return))
}
```
You know your functions are interesting when the signature is a lot longer than the body. Anyway, here's your customary example:
```rs
// Type alias inlined. I know the readability suffers, but I want the type to be explicit as possible while you look at what I do with it
fn coroutine(
    arg: Number, 
    handler1: fn(String, Coroutine<String, Bool, Number>) -> !, 
    resume: fn(Number, fn(String, Coroutine<String, Bool, Number>) -> !, fn(Bool) -> !) -> !,
) -> ! {
    yield("foo", handler1, |num1, handler2, _| print(num1, || {
        yield("bar", handler2, |num2, handler3, _| print(num2 || {
            yield("baz", handler3, |_, _, ret3| ret3(true))
        }))
    }))
}
fn consume(ret: fn() -> !) -> ! {
    Coroutine(coroutine, |coro| next(coro, 1
        |foo, rest| print(foo, || next(rest, 2, 
            |bar, rest| print(bar, || next(rest, 3,
                |baz, rest| print(baz, || next(rest, 99,
                    unreachable,
                    |t| print(t, ret),
                )),
                unreachable,
            )),
            unreachable,
        )),
        unreachable,
    ))
}
```
Calling `consume` prints, in this order, `foo`, `1`, `bar`, `2`, `baz`, `3`, `true`. Given how complex the code has gotten, you might have less trouble reading it in a non-CPS style. Just remember what I'm about to write can be desugared to CPS:
```rs
fn coroutine() -> Coroutine<String, Bool, Number> {
    let num1 = yield "foo";
    print(num1);
    let num2 = yield "bar";
    print(num2);
    let num3 = yield "baz";
    print(num3);
    return true;
}
fn consume() {
    let coro = coroutine();
    let CoroState::Yield(foo, coro) = coro.next(1) else { unreachable() };
    print(foo);
    let CoroState::Yield(bar, coro) = coro.next(2) else { unreachable() };
    print(bar);
    let CoroState::Yield(baz, coro) = coro.next(3) else { unreachable() };
    print(baz);
    let CoroState::Return(t) = coro.next(99) else { unreachable() };
    print(t);
    return;
}
```
With the example written, let's do what we did last chapter again, and start by looking at the struct (with the type alias inlined)
```rs
struct Coroutine<Y, R, N>(fn(
    N, 
    fn(Y, Coroutine<Y, R, N>) -> !, 
    fn(R) -> !,
) -> !);
```
A `Coroutine<Y, R, N>` is a coroutine that `yield`s values of type `Y`, eventually "returns" a value of type `R`, and requires a value of type `N` to call `next`. As with generators, a coroutine is a function, but this time it has _three_ arguments:
- `N` is the `next` argument: this is the value that `yield` returns
- `fn(Y, Coroutine<Y, R, N>) -> !` is the yield continuation: If the coroutine yields, it will pass the yielded element and the rest of itself to this function
- `fn(R) -> !` is the return continuation: Once the coroutine ends, it will return like any normal function. It just so happens that it returns _a value_, of type `R`.

Compared to generators, while `Coroutine` is a more complex type, `yield` and `return` are surprisingly almost unchanged: First, let's look at `yield`:
```rs
//once again, the type alias is inlined here
fn yield<Y, R, N>(
    val: Y, 
    handler: fn(Y, Coroutine<Y, R, N>) -> !, 
    resume: fn(N, fn(Y, Coroutine<Y, R, N>) -> !, fn(R) -> !) -> !,
) -> ! {
    Coroutine(resume, |coro| handler(val, coro))
}
```
We wrap `resume` into `Coroutine` for the usual "recursive type aliases are bad" reason, then we run our yield handler, passing it the value we yielded and the rest of the coroutine: just like the rest of a generator is a generator, the rest of a coroutine is a coroutine. `resume` does, however, have an interesting difference: It takes an argument of type `N`, as well as a continuation of type `fn(R) -> !`. The first is typically called the _resume argument_ and is the value supplied by `next`'s caller. The second is the new return continuation, which the rest of the coroutine can use to return.

`next` is a little bit more exciting, but only a little:
```rs
fn next<Y, R, N>(coro: Coroutine<Y, R, N>, arg: N, on_yield: fn(Y, Coroutine<Y, R, N>) -> !, on_return: fn(R) -> !) -> ! {
    coro.0(|c| c(arg, on_yield, on_return))
}
```
It hasn't changed a lot, but we can notice two interesting additions:
- `arg: N`. In generators, `yield` does not provide a value from the caller to the generator body. In coroutines it does, and this is that value.
- `on_return: fn(R) -> !`. Generators never return any value. Coroutines return `R`, so they need a continuation that accepts `R`.
Other than that, the logic is exactly the same. We unwrap `coro` and proceed to call it, passing it the resume argument and the two continuations to handle the two cases.

## Practical applications 3: Streaming parsers

In conventional parser design, one needs to have the entire input on hand before they can call `parse`. If I try to ask rustc to parse a source file that reads `fn main() { printl`, I'm going to get a whole bunch of error messages, even though this code isn't exactly _wrong_: it's _incomplete_. You can imagine a way to add more code that would make what I've written perfectly valid without any other change.

This design is perfectly suitable for most parsers, but not necessarily _all_ of them. In some cases, you may want to be able to receive data in chunks and parse it incrementally as the chunks arrive without having to wait for them all. For example, you might be interested in alternating between receiving data from a network connection and parsing the data you have received. The simplest way to design such a parser uses coroutines: The parser receives some input, advances its internal state consuming the input as it goes along, and eventually runs out. When it does, it `yield`s to the caller, remaining suspended. As soon as the caller can supply additional input, they can resume the parser from where it left off by calling `next`, passing the new block of input as a resume argument. Finally, when the parser is done (i.e. it has identified a complete message or found a definite error), it can return its result as normal.
This particular coroutine, interestingly, uses a `next` argument and a return value, but doesn't really `yield` any value. Given the type is `Coroutine<Y, R, N>`, a streaming parser might be a `Coroutine<(), &[u8], Result<Output, Error>>`. 

Alternatively or additionally, you may use the yield value for something: perhaps your parser simultaneously returns information about a given chunk of text and builds up a final return value: if you yield the information, your caller can get it immediately, which may, for example, help them decide whether they even care about the rest of the input.

## Practical applications 4: Async

When reading the previous section, you might have noticed that it's basically async code: you have a function that runs concurrently with a network request, and at any time it can choose to sit around and wait for the response to arrive before continuing.

More generally, the abstract idea of coroutines, the `Coroutine` type, is a natural model for async. The `yield` and `next` functions, however, are not and should be replaced.
- `yield` is the function that lets you suspend your coroutine for no particular reason, and doesn't arrange for it to be resumed automatically. This is not quite what you want. In async, you'd want to have a plethora of functions, each specific to a task. For example, you might have an async `get_data` function which starts a download. This function would want to take, on top of its normal arguments, a continuation to be invoked once the download is complete, which is basically the `resume` argument `yield` takes (if you're wondering where the `handler` argument went, async has very little need for it. You can still write it into your functions if it helps, but it rarely will).
- `next` lets you resume a coroutine. This is nice, but the download in progress won't immediately complete just because you called `next`. In async, you generally want `get_data` to be the one to "jump into" the rest of the coroutine, which it will do when it's time. This is still identifiably a coroutine, though: `get_data` just resumes it. It even passes a resume argument: the result of the download.

If you've used sufficiently old JavaScript, you might know this approach by the name "callback hell". Yes indeed, JS users did not particularly like doing async like this. Ironically, while they eventually solved their problem by doing _fewer_ callbacks (which worked great and was the correct solution for JS), the CPS approach is to solve the problem by doing _more_ callbacks. After all, once you live in a world where `return`, `break`, `continue`, exceptions, contexts, generators, and any other form of control flow you can come up with, _already work across functions_, you don't really lose any expressiveness by adding callbacks. You definitely do lose _readability_, but as I reiterated several times, this problem is inherent in CPS no matter what, and can be fixed with macros or a syntax transformation built into the language. You can even reinvent async/await, and desugar them in terms of CPS.

## Practical applications 5: Sans-IO

[Sans-IO](<https://sans-io.readthedocs.io/>) is a technique for writing implementations of protocols that would normally be used on files or across the network. The "traditional" way to write such code is by either hardcoding a single way to read/write files/messages (such as by only ever working with `File`s) or by somehow accepting an abstraction that does the reading/writing. (such as by accepting a `R: Read` or a `W: Write` in your functions).  
The problem with this technique is that it isn't as general as people will inevitably want it to be: even if you choose to be generic over `Read`/`Write`, someone will turn up and complain that your function doesn't work with _async_ readers and writers. You could add that, and then someone else will complain that your function doesn't work for their special use case that your new APIs _still_ don't cover. You could solve this problem by adding a new API every time someone complains, or you could do what sans-IO tells you to do and not do IO at all, instead accepting a sequence of bytes (or whatever your IO would have given you) as an argument and producing a sequence of bytes (or whatever you would have written) as a return value. Doing the IO to read/write those sequences of bytes is the caller's problem, which means they can do it however they want.

The catch is that Sans-IO works well in simple cases, but if you're working with a complicated protocol it can become _very annoying_. At its extreme, you might have to read a small amount of input just so you can tell the caller what you read so they can call another function with more input, which severely complicates what could have been two lines of code.  
Coroutines offer an elegant solution: Whenever you need to write some output, `yield` it to the caller and let them deal with it. Whenever you need to read some input, `yield` to the caller a value that tells them how much to read and wait for them to resume you with the result.

[^1]: Well, almost everything. If you're a pure functional language, you may be interested in having the freedom to evaluate expressions that do not depend on each other independently, possibly in parallel. CPS introduces artificial data dependencies all over the place, much like how imperative languages do, making this optimization more difficult. You _can_ add a primitive to explicitly unsequence multiple continuations and run them all in any order and/or in parallel, but that's additional work and will complicate your formal model if you have one

[^2]: I am a telepath. Fear me.

[^3]: Technically, even branching can be implemented without using a compiler builtin, if you're willing to get creative with how booleans are defined. For example, we could [Church-encode our booleans](https://en.wikipedia.org/wiki/Church_encoding#Church_Booleans). That said, actually doing that is besides the point of this blog post.

[^4]: Haskell's `where`, simplifying a little bit, is a keyword you can put after an expression to declare a variable used in that expression.

[^5]: A functional programmer would probably not write `for` at all, since using it to do anything useful requires side effects. What I'm actually doing is just replacing iteration with recursion.

[^6]: If it reminds you of pattern matching on a list, you have the right idea in mind.

[^7]: The astute reader may notice that it's possible to redefine `next` so that you don't need to wrap a function in a `Generator` to call it. This is true, but it forces you to unwrap a `Generator`, if you ever get an already-wrapped one, to call `next`. In practice, it just moves the inconvenience somewhere else.

[^8]: In fact, once you build proper coroutines, `yield` and `next` stop being different operations. They become the exact same thing. While you can still logically think of your code as having a driver that pulls elements from a generator, you can also think of it as having a generator that pushes elements into a sink. The two models are interchangeable.