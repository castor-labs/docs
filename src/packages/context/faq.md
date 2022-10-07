Frequently Asked Questions
==========================

## 1. Why no array context?

For mainly two reasons. (1) I wanted an api that used composition to extend the context, and (2) I wanted an API that
let me use any type as key. Arrays only support integers and strings.

## 2. Should I use it in my domain?

Yes! It is designed for this very purpose. Once you add the `Context` interface as a type hint of a repository
method or some other function, a world of possibilities are opened in terms of evolvavility and extensibility.