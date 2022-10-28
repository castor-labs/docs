Introduction
============

Context is an abstraction for passing request-scoped values down the call stack of an application.

The public api is inspired in [Golang's `context` package](https://pkg.go.dev/context).

## Installation

```bash
composer require castor/context
```

## Quick Start

```php
<?php

use Castor\Context;

$ctx = Context\nil(); // This is a default base context
$ctx = Context\withValue($ctx, 'foo', 'bar'); // This returns a new context with the passed values stored

// Later in the call stack

echo $ctx->value('foo'); // Prints: bar
```
