---
name: nightwatch-configure
description: Configure Laravel Nightwatch data collection, sampling rates, filtering rules, and redaction policies. Use this when setting up Nightwatch, managing data volume, protecting sensitive data (PII), or optimizing event collection for production workloads.
---

# Nightwatch Configuration Guide

This skill helps configure Laravel Nightwatch data collection to balance observability, performance, and privacy. Covers sampling strategies, filtering rules, and redaction methods across all event types.

## When to Use This Skill

Invoke when:

- Setting up Nightwatch for the first time
- Reducing data volume or event quota usage
- Protecting sensitive data (PII, credentials, tokens)
- Configuring production monitoring
- Optimizing for high-traffic applications
- Understanding filtering vs sampling vs redaction

## Documentation Reference

The [Nightwatch Documentation](https://nightwatch.laravel.com/docs) is the definitive and up-to-date source of information for all Nightwatch configuration options. This skill provides practical guidance and common patterns, but always consult the official documentation as the primary source of truth for specific details, environment variables, and API behavior. The documentation includes comprehensive coverage of:

- [Filtering and Configuration](https://nightwatch.laravel.com/docs/filtering) - Core concepts for sampling, filtering, and redaction
- Individual event type pages with specific configuration options:
  - [Requests](https://nightwatch.laravel.com/docs/requests) - Request sampling, header handling, payload capture
  - [Commands](https://nightwatch.laravel.com/docs/commands) - Command sampling and redaction
  - [Queries](https://nightwatch.laravel.com/docs/queries) - Query filtering and redaction
  - [Cache](https://nightwatch.laravel.com/docs/cache) - Cache event filtering by key or pattern
  - [Jobs](https://nightwatch.laravel.com/docs/jobs) - Job filtering and sampling decoupling
  - [Mail](https://nightwatch.laravel.com/docs/mail) - Mail event filtering
  - [Notifications](https://nightwatch.laravel.com/docs/notifications) - Notification filtering by channel
  - [Exceptions](https://nightwatch.laravel.com/docs/exceptions) - Exception sampling and throttling
  - [Outgoing Requests](https://nightwatch.laravel.com/docs/outgoing-requests) - HTTP request filtering

## Data Collection Flow

Nightwatch processes events through three stages:

1. **Sampling** - Controls which entry points are captured (requests, commands, scheduled tasks)
2. **Filtering** - Excludes specific events after sampling (queries, cache, mail, etc.)
3. **Redaction** - Modifies captured data to remove/obfuscate sensitive information

```
Request/Command/Scheduled Task
       |
       v
   [Sampling?] ----NO----> Drop entire trace
       | YES
       v
   Events generated
       |
       v
   [Filtering?] ----YES---> Drop specific event
       | NO
       v
   [Redaction] ----------> Store modified data
```

---

## Sampling Configuration

Sampling determines which entry points (requests, commands, scheduled tasks) trigger full trace collection. When an entry point is sampled, all related events are captured. For complete and authoritative information on all available sampling methods and environment variables, refer to the [Requests documentation](https://nightwatch.laravel.com/docs/requests) for HTTP request sampling and the [Commands documentation](https://nightwatch.laravel.com/docs/commands) for Artisan command sampling.

### Global Sample Rates

Configure via environment variables:

```bash
# Default: 100% sampling (all requests/commands captured)
NIGHTWATCH_REQUEST_SAMPLE_RATE=0.1      # Recommended: 10% of requests
NIGHTWATCH_COMMAND_SAMPLE_RATE=1.0      # Capture all commands
NIGHTWATCH_EXCEPTION_SAMPLE_RATE=1.0    # Always capture exceptions
```

**Recommendation**: Start with `0.1` (10%) for requests in production, adjust based on volume and needs.

The full list of environment variables for sampling configuration and their default values can be found in the official [Requests](https://nightwatch.laravel.com/docs/requests) and [Commands](https://nightwatch.laravel.com/docs/commands) documentation pages.

### Route-Based Sampling

Apply different rates to specific routes using the `Sample` middleware:

```php routes/web.php
use Illuminate\Support\Facades\Route;
use Laravel\Nightwatch\Http\Middleware\Sample;

// Sample admin routes at 100%
Route::middleware(Sample::rate(1.0))->prefix('admin')->group(function () {
    // All admin routes sampled fully
});

// Sample API routes at 5%
Route::middleware(Sample::rate(0.05))->prefix('api')->group(function () {
    // API routes sampled sparingly
});

// Always sample critical endpoints
Route::post('/checkout', [CheckoutController::class, 'process'])
    ->middleware(Sample::always());

// Never sample health checks
Route::get('/health', [HealthController::class, 'check'])
    ->middleware(Sample::never());
```

### Unmatched Route Sampling

Handle 404/bot traffic with reduced sampling:

```php routes/web.php
Route::fallback(fn () => abort(404))
    ->middleware(Sample::rate(0.01));  // 1% sampling for unmatched routes
```

### Dynamic Sampling

Sample based on runtime conditions (user role, request attributes):

```php app/Http/Middleware/SampleAdminRequests.php
use Closure;
use Illuminate\Http\Request;
use Laravel\Nightwatch\Facades\Nightwatch;

class SampleAdminRequests
{
    public function handle(Request $request, Closure $next)
    {
        if ($request->user()?->isAdmin()) {
            Nightwatch::sample();  // Always sample admin requests
        }
        return $next($request);
    }
}
```

### Command Sampling

Exclude specific commands from sampling:

```php AppServiceProvider.php
use Illuminate\Console\Events\CommandStarting;
use Illuminate\Support\Facades\Event;
use Laravel\Nightwatch\Facades\Nightwatch;

public function boot(): void
{
    Event::listen(function (CommandStarting $event) {
        if (in_array($event->command, ['schedule:finish', 'horizon:snapshot'])) {
            Nightwatch::dontSample();
        }
    });
}
```

### Vendor Commands

Nightwatch automatically ignores framework/internal commands. Opt-in to capture them:

```php
Nightwatch::captureDefaultVendorCommands();
```

---

## Filtering Configuration

Filtering excludes specific events from collection after sampling. Use filtering to reduce noise and quota usage. Each event type has specific filtering methods documented on its respective page. Refer to the [Queries](https://nightwatch.laravel.com/docs/queries), [Cache](https://nightwatch.laravel.com/docs/cache), [Jobs](https://nightwatch.laravel.com/docs/jobs), [Mail](https://nightwatch.laravel.com/docs/mail), [Notifications](https://nightwatch.laravel.com/docs/notifications), and [Outgoing Requests](https://nightwatch.laravel.com/docs/outgoing-requests) documentation for complete filtering options for each event type.

### Database Queries

**Filter all queries** (disable query collection):

```bash
NIGHTWATCH_IGNORE_QUERIES=true
```

**Filter specific queries** by SQL pattern:

```php AppServiceProvider.php
use Laravel\Nightwatch\Facades\Nightwatch;
use Laravel\Nightwatch\Records\Query;

public function boot(): void
{
    // Filter job table queries (PostgreSQL)
    Nightwatch::rejectQueries(function (Query $query) {
        return str_contains($query->sql, 'into "jobs"');
    });

    // Filter cache table queries (MySQL)
    Nightwatch::rejectQueries(function (Query $query) {
        return str_contains($query->sql, 'from `cache`')
            || str_contains($query->sql, 'into `cache`');
    });
}
```

### Cache Events

**Filter all cache events**:

```bash
NIGHTWATCH_IGNORE_CACHE_EVENTS=true
```

**Filter by cache key patterns**:

```php
Nightwatch::rejectCacheKeys([
    'my-app:users',                    // Exact match
    '/^my-app:posts:/',                // Regex: starts with my-app:posts:
    '/^[a-zA-Z0-9]{40}$/',             // Regex: session IDs
]);
```

**Filter with callback**:

```php
use Laravel\Nightwatch\Records\CacheEvent;

Nightwatch::rejectCacheEvents(function (CacheEvent $cacheEvent) {
    return str_starts_with($cacheEvent->key, 'temp:');
});
```

### Mail Events

**Filter all mail**:

```bash
NIGHTWATCH_IGNORE_MAIL=true
```

**Filter specific mail**:

```php
use Laravel\Nightwatch\Records\Mail;

Nightwatch::rejectMail(function (Mail $mail) {
    return str_contains($mail->subject, 'Newsletter');
});
```

### Notification Events

**Filter all notifications**:

```bash
NIGHTWATCH_IGNORE_NOTIFICATIONS=true
```

**Filter by channel**:

```php
use Laravel\Nightwatch\Records\Notification;

Nightwatch::rejectNotifications(function (Notification $notification) {
    return $notification->channel === 'database';
});
```

### Outgoing HTTP Requests

**Filter all outgoing requests**:

```bash
NIGHTWATCH_IGNORE_OUTGOING_REQUESTS=true
```

**Filter by URL**:

```php
use Laravel\Nightwatch\Records\OutgoingRequest;

Nightwatch::rejectOutgoingRequests(function (OutgoingRequest $request) {
    return str_contains($request->url, 'analytics.example.com');
});
```

### Queued Jobs

**Filter specific jobs**:

```php
use Laravel\Nightwatch\Records\QueuedJob;

Nightwatch::rejectQueuedJobs(function (QueuedJob $job) {
    return $job->name === 'App\Jobs\LowPriorityJob';
});
```

### Decoupling Job Sampling

Sample jobs independently from parent contexts:

```php
use Illuminate\Support\Facades\Queue;

public function boot(): void
{
    Queue::before(fn () => Nightwatch::sample(rate: 0.5));
}
```

---

## Redaction Configuration

Redaction modifies captured data to remove or obfuscate sensitive information. Unlike filtering, redaction keeps the event but sanitizes its content. For authoritative details on what data can be redacted from each event type and the exact callback signatures, consult the individual event type documentation pages referenced in the filtering section above.

### Request Redaction

**Redact sensitive headers** (automatically redacts: Authorization, Cookie, X-XSRF-TOKEN):

```bash
# Customize redacted headers
NIGHTWATCH_REDACT_HEADERS=Authorization,Cookie,Proxy-Authorization,X-API-Key

# Disable header collection entirely
NIGHTWATCH_IGNORE_REQUEST_HEADERS=true
```

**Redact request payloads** (disabled by default):

```bash
# Enable payload capture
NIGHTWATCH_CAPTURE_REQUEST_PAYLOAD=true

# Customize redacted fields
NIGHTWATCH_REDACT_PAYLOAD_FIELDS=password,password_confirmation,ssn,credit_card
```

**Programmatic redaction**:

```php
use Laravel\Nightwatch\Facades\Nightwatch;
use Laravel\Nightwatch\Records\Request;

Nightwatch::redactRequests(function (Request $request) {
    $request->url = str_replace('secret', '***', $request->url);
    $request->ip = preg_replace('/\d+$/', '***', $request->ip);
});
```

### Query Redaction

```php
use Laravel\Nightwatch\Records\Query;

Nightwatch::redactQueries(function (Query $query) {
    $query->sql = str_replace('secret_token', '***', $query->sql);
});
```

### Cache Redaction

```php
use Laravel\Nightwatch\Records\CacheEvent;

Nightwatch::redactCacheEvents(function (CacheEvent $cacheEvent) {
    $cacheEvent->key = str_replace('user:', 'user:***:', $cacheEvent->key);
});
```

### Command Redaction

```php
use Laravel\Nightwatch\Records\Command;

Nightwatch::redactCommands(function (Command $command) {
    $command->command = preg_replace('/--password=\S+/', '--password=***', $command->command);
});
```

### Exception Redaction

```php
use Laravel\Nightwatch\Records\Exception;

Nightwatch::redactExceptions(function (Exception $exception) {
    $exception->message = str_replace('secret', '***', $exception->message);
});
```

### Mail Redaction

```php
use Laravel\Nightwatch\Records\Mail;

Nightwatch::redactMail(function (Mail $mail) {
    $mail->subject = str_replace('Invoice #', 'Invoice ***', $mail->subject);
});
```

### Outgoing Request Redaction

```php
use Laravel\Nightwatch\Records\OutgoingRequest;

Nightwatch::redactOutgoingRequests(function (OutgoingRequest $outgoingRequest) {
    $outgoingRequest->url = preg_replace('/api_key=\w+/', 'api_key=***', $outgoingRequest->url);
});
```

---

## Configuration Summary by Event Type

| Event Type            | Sampling                                           | Filtering                                                                    | Redaction                 |
| --------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------- | ------------------------- |
| **Requests**          | `NIGHTWATCH_REQUEST_SAMPLE_RATE`, Route middleware | Not applicable                                                               | Headers, payload, URL, IP |
| **Commands**          | `NIGHTWATCH_COMMAND_SAMPLE_RATE`, Event listener   | Not applicable                                                               | Command arguments         |
| **Queries**           | Parent context                                     | `rejectQueries()`, `NIGHTWATCH_IGNORE_QUERIES`                               | SQL statement             |
| **Cache**             | Parent context                                     | `rejectCacheKeys()`, `rejectCacheEvents()`, `NIGHTWATCH_IGNORE_CACHE_EVENTS` | Cache key                 |
| **Jobs**              | Parent context, Queue::before                      | `rejectQueuedJobs()`                                                         | Not applicable            |
| **Mail**              | Parent context                                     | `rejectMail()`, `NIGHTWATCH_IGNORE_MAIL`                                     | Subject                   |
| **Notifications**     | Parent context                                     | `rejectNotifications()`, `NIGHTWATCH_IGNORE_NOTIFICATIONS`                   | Not applicable            |
| **Outgoing Requests** | Parent context                                     | `rejectOutgoingRequests()`, `NIGHTWATCH_IGNORE_OUTGOING_REQUESTS`            | URL                       |
| **Exceptions**        | `NIGHTWATCH_EXCEPTION_SAMPLE_RATE`                 | Not applicable                                                               | Exception message         |

---

## Production Recommendations

### High-Traffic Applications

```bash
# Conservative sampling
NIGHTWATCH_REQUEST_SAMPLE_RATE=0.01          # 1% of requests
NIGHTWATCH_COMMAND_SAMPLE_RATE=0.1           # 10% of commands
NIGHTWATCH_EXCEPTION_SAMPLE_RATE=1.0         # Always capture exceptions

# Filter noisy events
NIGHTWATCH_IGNORE_CACHE_EVENTS=true
NIGHTWATCH_IGNORE_QUERIES=true               # Or filter specific queries programmatically
```

### Privacy-Conscious Applications

```bash
# Disable sensitive data collection
NIGHTWATCH_CAPTURE_REQUEST_PAYLOAD=false
NIGHTWATCH_IGNORE_REQUEST_HEADERS=true

# Or use redaction in AppServiceProvider
```

### Balanced Configuration (Recommended Start)

```bash
# Sample rates
NIGHTWATCH_REQUEST_SAMPLE_RATE=0.1
NIGHTWATCH_COMMAND_SAMPLE_RATE=1.0
NIGHTWATCH_EXCEPTION_SAMPLE_RATE=1.0

# Filter obvious noise programmatically
# Redact PII as needed
```

---

## Verification Checklist

After configuration:

- [ ] Sampling rates appropriate for traffic volume
- [ ] Noisy events filtered (cache, certain queries)
- [ ] Sensitive data redacted (PII, tokens, credentials)
- [ ] Exceptions always captured for debugging
- [ ] Test in development with `NIGHTWATCH_REQUEST_SAMPLE_RATE=1.0`
- [ ] Monitor event quota usage in Nightwatch dashboard

---

## Common Patterns

### Filter Health Checks + Reduce Sampling

```php
Route::get('/health', fn() => ['status' => 'ok'])
    ->middleware(Sample::never());
```

### Exclude Internal/Vendor Queries

```php
Nightwatch::rejectQueries(fn($q) =>
    str_contains($q->sql, 'telescope') ||
    str_contains($q->sql, 'pulse')
);
```

### Protect User Data in Cache Keys

```php
Nightwatch::redactCacheEvents(fn($e) =>
    $e->key = preg_replace('/user:\d+/', 'user:***', $e->key)
);
```
