# Don't fear the Monad

*I was hesitant to publishing this article right from the beginning and even more so after reading [Letter to a Young Haskell Enthusiast](http://comonad.com/reader/2014/letter-to-a-young-haskell-enthusiast/). I wrote this post mainly for myself, because explaining a concept to someone else is oftentimes the best way of understanding it yourself. I decided to still put it out there because I’ve personally benefitted a lot from similarly written articles, but please bear this in mind while reading and complaining about the article.*

People who’ve worked with me will undoubtedly know that I’m a big fan of dependency injection[1]. Ease of unit testing is one benefit, but the biggest win in my opinion comes from forcing yourself to be explicit about the dependencies every component in a system has and the ability to easily swap them out.

## Prior Art

But I’m not the first one arguing for DI. This is why we find different tools and libraries for nearly every imaginable language and framework combination out there leveraging different mechanisms. For Java you can choose between [AOT Code Generation](http://square.github.io/dagger/), [Annotations + Reflection](http://code.google.com/p/google-guice/) and [unholy amounts of XML](http://spring.io/). With Scala you also get [implicits](http://blog.originate.com/blog/2013/10/21/reader-monad-for-dependency-injection/), in addition to the previous options. For JavaScript [AngularJS](https://docs.angularjs.org/guide/di) was arguably the first project to bring DI to the masses, making use of named function parameters which are going to make way for ES7 annotations with AngularJS 2.0 and [DI.js](https://github.com/angular/di.js) through transpilation. There is, however, another way in which we approach this problem and that is the Reader Monad.

## Category Theory to the Rescue

The Reader Monad is a bit of an odd one. When I was [learning Haskell](https://github.com/krispo/awesome-haskell), I found it quite hard to get an intuition for it. Owing to its very abstract definition, it’s hard to get an idea for a concrete use case. However, when I heard that it could be used for DI, I was intrigued to see how that might look like. Whenever I’m learning a new concept and am having trouble wrapping my head around it, I try to apply it to something else that I already know. In this case, we are going to try this by implementing the Reader Monad pattern in JavaScript.

We’re going to start by looking at some simple but frankly quite contrived application examples in Haskell and translate them to JavaScript afterwards. I’m not assuming any advanced Haskell knowledge - as this would be a completely ludicrous requirement for a Haskell noob like me to make - but you should probably have seen the syntax before. If you don’t, just skim them and come back once we got to the JavaScript counterparts. Also, don’t confuse this for a Monad tutorial. I am not going to run into that trap.

## An Unmotivating Example

Most examples for the Reader Monad are going to show you something like this, sadly this one isn’t going to be an exception:


```haskell
example :: String
example = runReader computation "Hello"
    where
        computation :: Reader String String
        computation = do
            greeting <- ask
            return $ greeting ++ ", Haskell"

main = putStrLn example
```


> `Hello, Haskell`

So what’s going on here? We execute (or "run") a Reader `computation` by passing “Hello” in as context. By looking at the signature of `computation`, we can see that it is a `Reader` with two type parameter. The first one specifies the context, the latter is what it is ultimately going to evaluate to when we run it. By using `ask` we can get ahold of the context carried by the Reader. The reason why this example on its own is pretty useless is because we could do the same much easier by passing `greeting` as parameter to our `computation` and thus avoiding all the Monad-y jazz.

Let’s change things slightly to make the benefit of the separate context more apparent:


```haskell
example1 :: String -> String
example1 context = runReader (computation "Tom") context
    where
        computation :: String -> Reader String String
        computation name = do
            greeting <- ask
            return $ greeting ++ ", " ++ name

main :: IO ()
main = putStrLn example1 "Hello"
```


> `Hello, Tom`

Looking at the type signature of `computation` we can see that it now accepts `name` as a direct parameter and continues to return a `Reader String String`. As you can see, the context serves as configuration to the function here and is completely transparently passed through the function execution. Even for this simple example it’s possible to envision how an [I18N](https://en.wikipedia.org/wiki/Internationalization_and_localization) integration through the Reader could look like by passing different contexts based on the current locale.

## Compo >>= sition

Whenever you ask someone about the benefits of functional programming, you can be sure that they will mention composition at least 42 times. And not without reason! So let’s make a final modification before we dive into JavaScript land. We are going to add some punctuation to our greeting context. If the greeting is "Hello", we’re going to append an exclamation mark, otherwise a full stop:


```haskell
example2 :: String -> String
example2 context = runReader (greet "James" >>= end) context
    where
        greet :: String -> Reader String String
        greet name = do
            greeting <- ask
            return $ greeting ++ ", " ++ name

        end :: String -> Reader String String
        end input = do
            isHello <- asks (== "Hello")
            return $ input ++ if isHello then "!" else "."

main :: IO ()
main = putStrLn $ example2 "Hello"
```


> `Hello, James!`

As you can see, the Reader context is passed on completely transparently. All we need to do is bind our `greet` function to `end` where you can think of `>>=` as a pipe you’d use in your favorite shell.

## Plumbing

Now that we had enough fun putting the Reader to work, let’s have a look behind the scenes and see how we can create a minimal implementation[2] of it.


```haskell
newtype Reader r a = Reader { runReader :: r -> a }

instance Monad (Reader r) where
    return a = Reader $ \_ -> a
    m >>= k = Reader $ \r -> runReader (k $ runReader m r) r

asks :: (r -> a) -> Reader r a
asks f = Reader f

ask :: Reader a a
ask = Reader id
```


Let’s go through this line by line. The first step is to create a new data type for `Reader`. This takes two type parameters: `r` -- our context -- and `a` -- our eventual result. The right hand side of the equation defines the constructor for `Reader` taking a function of type `r -> a` -- from our context to the result type -- that we can later access through `runReader`.

We then make `Reader` a Monad by implementing the corresponding type class. The `return` function is really straight-forward: Given an arbitrary input, we create a new Reader whose context ignores any input and directly returns the result. Instead of `\_ -> a` we could also have used `const` from Prelude which is defined exactly as this.

The next line is probably where most confusion stems from, but there’s no magic at work. Again, we’re creating a new `Reader`, but this time while taking the parameter into account. This `r` -- our original context -- is simply passed on as context for our new instance, completely unmodified. Inside the parenthesis we’re passing the unwrapping the `Reader` and passing its result to the function `k`. In essence, there’s just a bunch of unwrapping and rewrapping going on, resulting in a Reader with a new input while keeping the same context.

## A Monad in JavaScript

So far, so confusing. Haskell’s beautiful conciseness is also its greatest weakness when it comes to learning. As we will see, achieving the same in JavaScript is going to be a lot more verbose, but maybe just because of that also easier to digest.

Let’s approach this from the opposite end and start by implementing the Reader first and then see how we can use it in our application code.


```js
"use strict";

var Reader = function (fn) {
    this.f = fn;
};

Reader.ask = function () {
    return new Reader(_.identity);
};

Reader.asks = function (fn) {
    return new Reader(fn);
};
```


We first mirror the type class, which in this case is a simple constructor function, and the two value extraction functions, which we scope to the `Reader` namespace. By using Underscore or Lo-Dash we can save a few keystrokes.


```js
Reader.prototype.run = function (ctx) {
    return this.f(ctx);
};
```


The `run` function is the first function on the prototype and is equivalent to our `runReader` function in the Haskell implementation.


```js
Reader.prototype.unit = function (fn) {
    return new Reader(_.constant(fn));
};

Reader.prototype.flatMap = function (k) {
    return new Reader(function (r) {
        return k.call(this, this.run(r)).run(r);
    }.bind(this));
};
```


These two functions are what turn our object into a monad. Instead of `return` and `>>=` (bind) we name them `unit` and `flatMap` because the former two have an entirely different meaning in JavaScript.[3]

`unit` is pretty straightforward. We create and return a new `Reader` instance while ignoring the previous context and instead provide a new function that always returns `fn` independent of the parameters its called with.

`flatMap` is a bit more involved, but if we remind ourselves of the Haskell implementation, we can see that it’s actually a 1:1 translation of `m >>= k = Reader $ \r -> runReader (k $ runReader m r) r`. `m` is the previous instance of the `Reader` which is simply `this` in the JS world. `k` is the new `Reader` we’re "binding" to or “flatMapping” over. The return value, once again, is a new `Reader` that takes a context `r` that it uses to unwrap the passed in as well as the previous instance by calling `run` on it. In JavaScript, we have to make sure that we maintain the correct context. This is why we use `bind` on the outer scope so we can use `this` inside the new `Reader` when we call `k`.

## The JS Reader in Action

Let’s put our JavaScript `Reader` to use and replicate the previous examples: 


```js
var greet = function (name) {
    return Reader.ask().flatMap(function (ctx) {
        return this.unit(ctx + ", " + name);
    });
};

var example0 = function () {
    console.log(greet("JavaScript").run("Hi"));
};

example0();
```


> `Hi, JavaScript`

This looks familiar, doesn’t it? We actually cheated a bit by passing `name` to the outer function and closing over it instead of properly currying the parameter, but I’ll leave this as an exercise to the purist readers.[4]

This is great! The `greet` behaves like any normal function from the outside but instead of directly executing, it returns us a `Reader` that defers its execution until we `run` it with an appropriate context. But Will It Compose?

## Compose like there’s no tomorrow

Let’s dive right into the third Haskell example that added punctuation based on our incredibly elaborate ruleset:


```js
var end = function (str) {
    var isHello = _.partial(_.isEqual, "Hello");
    return Reader.asks(isHello).flatMap(function (isH) {
        return this.unit(str + (isH ? "!" : "."));
    });
};

var example1 = function () {
    console.log(greet("James").flatMap(end).run("Hello"));
    console.log(greet("Tom").flatMap(end).run("Hi"));
};

example1();
```


> `Hello, James!`

> `Hi, Tom.`

For this example, we’re reusing our previous `greet` function which works seamlessly together with our new `end` function that makes use of the `asks` function to run a comparison over the context. In order to form a new `Reader` based on the two, we compose them with `flatMap` just as we did before with `(greet "James" >>= end)`. The resulting `Reader` still takes one context that is transparently passed through the evaluation with one central entry point. Now that we’ve arrived in JavaScript land it’s probably easier to imagine changing our primitive String context with a more complex data structure that could contain actual configuration data. 

## Following the evaluation

If you followed the JavaScript example closely, one question might have crossed your mind: Where do all those nested functions come from? If we imagined further accesses to our `Reader` context each of them would require a new closure. Why do we see this here, but not in Haskell? The answer lies in the `do` expression. The `do` notation in Haskell is actually just syntactical sugar for Monads[5]. If we desugar our `end` example, it becomes apparent where all the curly braces in JavaScript came from. While doing this, let’s add another rule: If the second letter is an "e", repeat the character twice.


```haskell
end :: String -> Reader String String
end input = do
    isHello <- asks (== "Hello")
    hasE <- asks((== "e") . (drop 1) . (take 2))
    let count = if hasE then 3 else 1
    return $ input ++ (concat $ replicate count $ if isHello then "!" else ".")

end' :: String -> Reader String String
end' input =
    asks (== "Hello") >>= (\isHello ->
        asks((== "e") . (drop 1) . (take 2)) >>= (\hasE ->
            let count = if hasE then 3 else 1 in
            return $ input ++ (concat $ replicate count $ if isHello then "!" else ".")
        )
    )
```


Do you see what happened? When we desugar the `do` notation, it unfolds into a bunch of nested lambdas, exactly like it did in JavaScript. Manually unpacking the `do` notation can really help trying to understand what actually happens to your Monads. Some even [consider the `do` notation harmful](http://syntaxfree.wordpress.com/2006/12/12/do-notation-considered-harmful/) because it gives code a deceptive imperative look.

## The value of implicit contexts

Imagine that instead of these rather short-lived values we’d use the Reader context to pass on application-wide configuration values. A typical example would be an API endpoint the application talks to. In JavaScript applications, the usual way to provide these is through a configuration file that assigns them to a global object. This is problematic if you want to test against a different environments and mutating global state means that you cannot execute multiple tests in parallel.

With a Reader Monad you can create the illusion for your application code to run with a global immutable configuration that is actually scoped to its particular evaluation context. On powerful feature I didn’t touch is the possibility to run a computation in a modified environment with [`local`](https://hackage.haskell.org/package/mtl-1.1.0.2/docs/Control-Monad-Reader-Class.html#v:local) which would be entirely impossible with a shared, mutable state.

## Conclusion

The Reader Monad is a very light-weight and flexible solution to the configuration problem that can be used in pretty much every programming environment. Does that mean that you should rewrite your AngularJS apps to use the implementation above instead? Most certainly not. In fact, if you break down common dependency injection libraries they are just different implementations of the Reader Monad pattern. Most likely in a way that makes better use of the underlying platform and language features, though.

I hope this was helpful to someone else, it certainly was for me. Breaking down scary looking problems into smaller ones and trying to apply them to different environments can be a very powerful way to develop an intuition. 

## Further Reading

[Three Useful Monads](http://adit.io/posts/2013-06-10-three-useful-monads.html)

[Brent Yorgey’s Typeclassopedia](http://www.haskell.org/haskellwiki/Typeclassopedia)

[monet.js - For when you have no choice but to use JavaScript](https://cwmyers.github.io/monet.js/)

## Footnotes

[1] Some people, especially those coming from a more functional programming oriented background, frown upon the term dependency injection which has its root in the OOP world. I actually prefer referring to this as the configuration problem, but DI has a way broader recognition these days.

[2] We’ll deliberately omit some features of the Reader Monad that we don’t use in this article for simplicity’s sake. The full implementation in the `mtl` library is still [very readable](https://hackage.haskell.org/package/mtl-1.1.0.2/docs/src/Control-Monad-Reader.html#Reader).

[3] Also `return` is obviously a keyword so even if we wanted, we couldn’t give our function that name.

[4] And with that last part of the sentence I can finally cross that item off my life’s TODO list.

[5] [And hopefully soon for `Applicatives`, too!](https://ghc.haskell.org/trac/ghc/wiki/ApplicativeDo)
