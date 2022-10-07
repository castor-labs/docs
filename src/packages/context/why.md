Why this Library?
=================

This library attempts to solve the so called context-passing problem. For a good overview of the problem you can
read [Matthias Noback's blog post about it](https://matthiasnoback.nl/2018/04/context-passing/), or continue
reading here.

## Context Passing

In traditional PHP applications, the *context* in which an action is being executed is a fundamental
piece of information to determine program logic. By context, we could mean, for example, the user that is
executing the action, or the current tenant.

If you use the popular PHP frameworks, you can get that information pretty easily. It usually involves
calling a global (like the [Auth facade in Laravel][laravel-auth]) or injecting a service that stores this
information inside some shared class state (like [Symfony's Security service][symfony-auth]).

The problem with this solution is that it relies on global shared state. Although their access mechanisms vary (one
uses global access while the other uses dependency injection), they still rely on storing state in a shared instance.
They are fundamentally the same thing.

A better approach is to create a context object and pass it down explicitly down the call stack, so interested
parties can extract from it the information they need to process the request. But this object needs to be carefully
designed, so it does not expose a wide api and pollute client code making the interface too big (a violation of
the Dependency Inversion principle).

The context package provides such object. It allows you to give every request its own immutable context, where you
can store request specific values. It is not a service, so it can't be shared globally, and it must be passed
explicitly down the call stack, so no magic mutations happen during its lifecycle.

## The `Context` interface

This package defines one very small and simple interface called `Castor\Context`. Your code should typehint to this
interface always. This library is intentionally simple and has a small api surface and footprint because it is 
intended to be used even in your domain layer.

As we said, the `Context` interface has an intentionally simple api, with just one method: `value(mixed $key): mixed`.
This method returns the value "stored" in the context for the given key. Pay special attention to the fact that
you can store values of any type as keys, and comparison of the keys is done by strict equality.

If you are a good observer, you'll quickly realize the `Context` interface does not define any methods that mutate
its internal state like `set(mixed $key, mixed $value): void`. Although we considered that method for a second, we
copied from Go the idea to make this interface unavoidably immutable. You can only "store" new values by composing 
a new `Context`, and we have provided some functions to ease that process.

## The `Context\fallback()` and the `Context\with_value()` functions

First, in order to "store" a value in the context, you need to get the fallback context. This is some sort of "empty"
context that always returns `null`. You call `Context\fallback()` to do this.

```php
<?php

use Castor\Context;

// This gives you a context that always returns null for any key
$ctx = Context\fallback();

var_dump($ctx->value('foo')); // Prints: NULL
```

Once you have a `Context` instance, you can "store" a value in it by calling `Context\with_value()`:

```php
<?php

use Castor\Context;

// This gives you a context that always returns null for any key
$ctx = Context\fallback();

// This returns a new context with the stored key value pair
$ctx = Context\with_value($ctx, 'foo', 'bar');

var_dump($ctx->value('foo')); // Prints: string(3) "bar"
```

Basically, those two functions and a single interface is the whole api surface of the context
package. In the next chapter we'll go to some guides on how to leverage the power of the context api
to build powerful functionality in your applications.

[laravel-auth]: https://laravel.com/docs/9.x/authentication#retrieving-the-authenticated-user
[symfony-auth]: https://symfony.com/doc/current/security.html#fetching-the-user-from-a-service