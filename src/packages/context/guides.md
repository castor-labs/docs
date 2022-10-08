Guides & Examples
=================

## Implementing Multi-Tenancy

A common use case for many applications is to support multi-tenancy. Multi tenancy is the ability of an application
to operate for multiple tenants or customers using the same instance and codebase.

Now we are going to see an example of how we can implement multi tenancy using the context api.

Let's suppose our imaginary application determines the tenant based on the subdomain, in the HTTP layer, in some
middleware:

```php
#<?php
#
#namespace MyMultiTenantApp;
#
#use Castor\Context;
#use Castor\Http\Handler;
#use Castor\Http\Request;
#use Castor\Http\ResponseWriter;
#
#class TenancyMiddleware implements Handler
#{
#    private Handler $next;
#    private TenantRepository $tenants;
    
#    public function __construct(Handler $next, TenantRepository $tenants)
#    {
#        $this->next = $next;
#        $this->tenants = $tenants;
#    }
#    
    public function handle(Context $ctx, ResponseWriter $wrt, Request $req): void
    {
        $hostname = $req->getUri()->getHostname();
        // This should separate subdomain.domain.tld 
        $parts = explode('.', $hostname, 3);
        
        // No subdomain
        if (count($parts) <= 2) {
            $this->next->handle($ctx, $wrt, $req);
            return;
        }
        
        $tenantName = $parts[0] ?? '';
        
        try {
            $tenant = $this->tenants->ofId($ctx, $id);
        } catch (NotFound) {
            $this->next->handle($ctx, $wrt, $req);   
            return;
        }
        
        // Once we have the tenant, we store it in the context
        $ctx = Context\with_value($ctx, 'tenant', $tenant);
        
        // And we pass it to the next handler
        $this->next->handle($ctx, $wrt, $req);
    }
#}
```

So, you have "captured" some important information about the context in which the request is being made, which
is the current tenant in use (or no tenant at all if no subdomain is present).

Now, further layers of the application that are interested in this piece of information, can modify their behaviour
based on the absence or presence of such value.

For instance, we want to list all the users, but only those of the current tenant:

```php
#<?php
#
#namespace MyMultiTenantApp;
#
#use Castor\Context;
#use MyMultiTenantApp\QueryBuilder;
#use MyMultiTenantApp\Tenant;
#
#class UserRepository
#{
#    private QueryBuilder $query;
#    
#    public function __construct(QueryBuilder $query)
#    {
#        $this->query = $query;
#    }
#    
    public function all(Context $ctx): array
    {
        $query = $this->query->select()->from('users');
        
        $tenant = $ctx->value('tenant');
        if ($tenant instanceof Tenant) {
            $query = $query->where('tenant_id = ?', $tenant->getId());
        }
        
        return $query->execute();
    }
#}
```

The context interface provides several benefits in this case. It's transparency means you could deploy
this for a single tenant and the application would work exactly the same. You could even implement lazy
database connection based on the tenant to point them to different databases.

Possibilities are endless.

## Passing Logger Context

Another common problem in PHP applications is the passing of context for log calls. Say, for example, you want all your
logs to contain the `request_id` property, so you can group them together and explore all the logs emitted by a single
request, but you don't want to pass the request id to every single call that could reach a logger. That would be a
terrible thing to do. What you want here is the power of context.

In this short guide, we'll explore more advanced things you can do with context. First, let's create a simple object
to hold the state we want to accumulate to later send to the logger.

```php
<?php

namespace MyApp\Logger;

class LogContext
{
    protected array $data;
    
    public function __construct()
    {
        $this->data = [];
    }
    
    public function add(string $key, mixed $value): LogContext
    {
        $this->data[$key] = $value;
        return $this;
    }
    
    public function merge(array $data): LogContext
    {
        $this->data = array_merge($this->data, $data);
        return $this;
    }
    
    public function toArray(): array
    {
        return $this->data;
    }
}
```

This is the class that we are going to store in the context. It does not need to be immutable, because Context is 
immutable and every context instance will have its own instance of this class. There is no possibility that that instance
will be shared outside the scope of the request, so it is safe. Immutability is good in certain contexts, but
in this case it is not needed. We want this class to be mutable *by design*.

Now, we will provide a simple set of functions on top of the context functions, to ease the manipulation of this
class. Note these are pure functions: passed the same input, they will yield the same output.

```php
<?php

namespace MyApp\Logger;

use Castor\Context;

#class LogContext
#{
#    protected array $data;
#    
#    public function __construct()
#    {
#        $this->data = [];
#    }
#    
#    public function add(string $key, mixed $value): LogContext
#    {
#        $this->data[$key] = $value;
#        return $this;
#    }
#    
#    public function merge(array $data): LogContext
#    {
#        $this->data = array_merge($this->data, $data);
#        return $this;
#    }
#    
#    public function toArray(): array
#    {
#        return $this->data;
#    }
#}

// First we create an enum for the key
enum Key
{
    case LOG_CONTEXT
}

// This function add an entry to the stored LogContext
function with_log_context(Context $ctx, string $key, mixed $value): Context
{
    $logCtx = $ctx->value(Key::LOG_CONTEXT);
    
    // If we have an instance already, we mutate that, and we don't touch the context
    if ($logCtx instanceof LogContext) {
        $logCtx->add($key, $value);
        return $ctx;   
    }
    
    // We mutate the context only if we don't have a LogContext object in it
    $logCtx = new LogContext();
    $logCtx->add($key, $value);
    
    return Context\with_value($ctx, Key::LOG_CONTEXT, $logCtx);
}

function get_log_context(Context $ctx): LogContext
{
    // We try to not throw exceptions but provide defaults instead.
    return $ctx->value(Key::LOG_CONTEXT) ?? new LogContext();
}
```

Note how these functions make it easy to add this particular bit of information to the context. They also
hide the need for client code to know the context key, and they take sensible decisions for client code. The functions
are pure, and with no side effects.

Now, let's work in the middleware that will capture our request:

```php
<?php

#namespace MyApp\Logger;
#
#use Castor\Context;
#
#class LogContext
#{
#    protected array $data;
#    
#    public function __construct()
#    {
#        $this->data = [];
#    }
#    
#    public function add(string $key, mixed $value): LogContext
#    {
#        $this->data[$key] = $value;
#        return $this;
#    }
#    
#    public function merge(array $data): LogContext
#    {
#        $this->data = array_merge($this->data, $data);
#        return $this;
#    }
#    
#    public function toArray(): array
#    {
#        return $this->data;
#    }
#}
#
#// First we create an enum for the key
#enum Key
#{
#    case LOG_CONTEXT
#}
#
#// This function add an entry to the stored LogContext
#function with_log_context(Context $ctx, string $key, mixed $value): Context
#{
#    $logCtx = $ctx->value(Key::LOG_CONTEXT);
#    
#    // If we have an instance already, we mutate that, and we don't touch the context
#    if ($logCtx instanceof LogContext) {
#        $logCtx->add($key, $value);
#        return $ctx;   
#    }
#    
#    // We mutate the context only if we don't have a LogContext object in it
#    $logCtx = new LogContext();
#    $logCtx->add($key, $value);
#    
#    return Context\with_value($ctx, Key::LOG_CONTEXT, $logCtx);
#}
#
#function get_log_context(Context $ctx): LogContext
#{
#    // We try to not throw exceptions but provide defaults instead.
#    return $ctx->value(Key::LOG_CONTEXT) ?? new LogContext();
#}
#
namespace MyApp\Http;

use MyApp\Logger;
use Castor\Context;
use Castor\Http\Handler;
use Castor\Http\Request;
use Castor\Http\ResponseWriter;
use Ramsey\Uuid;
use function MyApp\Logger\with_log_context;

class RequestIdMiddleware implements Handler
{
    private Handler $next;
    
    public function __construct(Handler $next)
    {
        $this->next = $next;
    }
    
    public function handle(Context $ctx, ResponseWriter $wrt, Request $req): void
    {
        $requestId = req->getHeaders()->get('X-Request-Id');
        
        // If no request id comes, we assign one
        if ($requestId === '') {
            $requestId = Uuid::v4();
        }
        
        $ctx = with_log_context($ctx, 'request_id', $requestId);
        $this->next->handle($ctx, $wrt, $req);
    }
}
```

Then, the only bit left to do is to consume this from the code that calls the logger.

```php
<?php

#namespace MyApp\Logger;
#
#use Castor\Context;
#
#class LogContext
#{
#    protected array $data;
#    
#    public function __construct()
#    {
#        $this->data = [];
#    }
#    
#    public function add(string $key, mixed $value): LogContext
#    {
#        $this->data[$key] = $value;
#        return $this;
#    }
#    
#    public function merge(array $data): LogContext
#    {
#        $this->data = array_merge($this->data, $data);
#        return $this;
#    }
#    
#    public function toArray(): array
#    {
#        return $this->data;
#    }
#}
#
#// First we create an enum for the key
#enum Key
#{
#    case LOG_CONTEXT
#}
#
#// This function add an entry to the stored LogContext
#function with_log_context(Context $ctx, string $key, mixed $value): Context
#{
#    $logCtx = $ctx->value(Key::LOG_CONTEXT);
#    
#    // If we have an instance already, we mutate that, and we don't touch the context
#    if ($logCtx instanceof LogContext) {
#        $logCtx->add($key, $value);
#        return $ctx;   
#    }
#    
#    // We mutate the context only if we don't have a LogContext object in it
#    $logCtx = new LogContext();
#    $logCtx->add($key, $value);
#    
#    return Context\with_value($ctx, Key::LOG_CONTEXT, $logCtx);
#}
#
#function get_log_context(Context $ctx): LogContext
#{
#    // We try to not throw exceptions but provide defaults instead.
#    return $ctx->value(Key::LOG_CONTEXT) ?? new LogContext();
#}
#
#namespace MyApp\Http;
#
#use MyApp\Logger;
#use Castor\Context;
#use Castor\Http\Handler;
#use Castor\Http\Request;
#use Castor\Http\ResponseWriter;
#use Ramsey\Uuid;
#use function MyApp\Logger\with_log_context
#
#class RequestIdMiddleware implements Handler
#{
#    private Handler $next;
#    
#    public function __construct(Handler $next)
#    {
#        $this->next = $next;
#    }
#    
#    public function handle(Context $ctx, ResponseWriter $wrt, Request $req): void
#    {
#        $requestId = req->getHeaders()->get('X-Request-Id');
#        
#        // If no request id comes, we assign one
#        if ($requestId === '') {
#            $requestId = Uuid::v4();
#        }
#        
#        $ctx = with_log_context($ctx, 'request_id', $requestId);
#        $this->next->handle($ctx, $wrt, $req);
#    }
#}

namespace MyApp\Services;

use Castor\Context;
use Psr\Log\LoggerInterface;
use function MyApp\Logger\get_log_context;

class SomeServiceThatLogsStuff
{
    private LoggerInterface $logger;
    
    public function __construct(LoggerInterface $logger)
    {
        $this->logger = $logger;   
    }
    
    public function doSomething(Context $ctx): void
    {
        // Do an action and then log it.
        // The context holds request_id and other values passed by other layers.
        $this->logger->debug('Action completed', get_log_context($ctx)->toArray());
    }
}
```

With this simple approach, your log calls will be richer and easier to filter. And you didn't need to
pollute your call stack with a massive class or resort to use globals. Everything is explicit, and by 
using the custom functions you make easy for your consumers to extract or insert things from and to the context.

This shows how evolvable the `Context` abstraction is, and how you can compose functionality on top of it by
keeping the interface thin.