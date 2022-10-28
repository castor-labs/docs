Best Practices
==============

The `Context` api is definitely a new pattern and approach to solve the context passing problem in PHP.
Whenever a new pattern is introduced, is always important to define some best practices for its good use.

These are some of what we consider to be those best practices.

## Custom functions SHOULD be defined to work with `Context`

Context is all about composition, and you should also compose the base context functions to provide a nicer api
to store or retrieve context values, like we did in the logger example. Of course, you could also rely on 
static methods, but in my personal opinion, functions are nicer.

You can even go as far as to create your own `Context` implementation that composes a `Context`. We do that for some
of our libraries. But we even discourage that. The thinner you can keep `Context` api surface, the better.

## `Context` SHOULD be the first argument of the function or method

Whenever a function or method call accepts a `Context`, this should be the first argument of the function, and 
it should be named `$ctx`. We know some people hate abbreviations and prefer more explicit names. I agree with that
suggestion almost in every situation, but in this case the abbreviation does not yield any sort of confusion and
is pretty universal and easily understood.

It also has the benefit that it is short, and since `Context` is there to be added to your public api, we want
to minimize as much as possible the space impact that has in your method's argument list.

## Enums SHOULD be used as keys rather than strings

When calling `Context\withValue()` prefer enums as keys. They are lightweight, offer autocompletion, and they cannot 
collide like strings could. In the case that your application still does not support PHP 8.1 and above, you MUST use
string keys with a vendor namespace.

## `Context` SHOULD NOT be stored inside other data structures

Always explicitly pass `Context`. Do not store it inside objects or arrays, unless is explicitly necessary. For
instance, if you are using the `Context` api in PSR-7 applications, it is very likely your `Context` instance
will be stored in the Request attributes (which is implemented by an array). This is acceptable, but for a better
HTTP layer supporting context natively, we recommend `castor/http`.

## `Context\withValue` SHOULD NOT be overused

Because of its particular implementation, every time you add a value to a `Context`, you increase the potential call
stack size to reach the value by 1. Although the performance impact of this is negligent, is still slower than fetching
a value directly from a map, for instance.

So, bottom line, don't overuse `Context\withValue`. This means that if you have to store related values in
`Context`, store a data structure instead and not each value individually.

Again, the performance impact of not doing this is negligible, so measure and make decisions based on that. 

> In a potential new major version, we are considering swapping the `Context\withValue()` implementation by using a
[`DS\Map` if the extension is available](https://www.php.net/manual/en/class.ds-map.php) to avoid the performance
penalty.

## `Context` SHOULD NOT contain immutable values

A `Context` implementation is already immutable, and it's lifetime does not go outside the request. This means it is
safe for manipulation and free of unexpected side effects.

As long as `Context` holds values derived from the request, whose lifetime will also die with it, then it is safe to
store mutable values in it. If you store immutable values and decide that a new reference of that value needs to be
passed down the call stack it means the value should have never been immutable in the first place. You'll have to call
`Context\withValue` again and "override" that value.

## Services SHOULD NOT be stored inside `Context`

We greatly discourage storing services inside `Context`. It is not a good idea to mix request-scoped values with 
application scoped-values like services. Always prefer dependency injection instead of passing services down the
context.

There could be some use cases when this could make sense. For instance, if you need to pass an event loop instance
down the call stack to reach some method and make it work asynchronously. I would say that is part of the execution
context, although an event loop can be classified as a service.