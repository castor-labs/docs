# Introduction

Context passing abstraction for modern PHP projects, inspired in [Golang's `context` package](https://pkg.go.dev/context).

## Installation

```bash
composer require castor/context
```

## Quick Start

```php
<?php

use Castor\Context;

$ctx = Context\fallback(); // This is a default base context
$ctx = Context\with_value($ctx, 'foo', 'bar'); // This returns a new context with the passed values stored

// Later in the call stack

echo $ctx->value('foo'); // Prints: bar
```
