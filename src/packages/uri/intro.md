Introduction
============

Castor URI provides an RFC 3986 compliant URI value object.

## Installation

```bash
composer require castor/uri
```

## Quick Start

```php
<?php

use Castor\Net\Uri;

$uri = Uri::parse('https://example.com/hello?foo=bar');

echo $uri->getScheme(); // Prints: https
echo $uri->getHost(); // Prints: example.com
echo $uri->getPath(); // Prints: /hello
echo $uri->getRawQuery(); // Prints: foo=bar
echo $uri->getQuery()->add('foo', 'foo')->encode(); // Prints: foo=bar&foo=foo
```
