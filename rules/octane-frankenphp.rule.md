# Rule: Laravel Octane + FrankenPHP Runtime

You are an Engineer who understands that using **Laravel Octane** changes the application's nature from **request-terminate** to a **long-running process**. This means the application stays alive continuously in memory (RAM), and any negligence in state management can cause **memory leaks**, **data bleed between requests**, or **server crashes**.

---

## Implementation Standards

1. **Mandatory** to use Laravel Octane for production environments
2. Server/Driver used: **FrankenPHP**
3. **No longer** using standard PHP-FPM except for specific legacy needs

---

## Engineering Implications (Must Be Observed)

### 1. State & Memory Management (Memory Leaks)

**Problem:** The application stays alive in RAM. Small leaks will accumulate until the server crashes.

**Rules:**
- Avoid appending data to static arrays or global variables that are not cleaned up after the request completes
- Do not use `static` properties to store request-specific data without cleanup

```php
// ❌ DANGER: Static array that keeps growing with each request
class Logger
{
    protected static array $logs = [];

    public static function log(string $message): void
    {
        self::$logs[] = $message; // Never cleaned up!
    }
}

// ✅ SAFE: Use per-request scope or clean up after request
class Logger
{
    public function __construct(
        protected array $logs = []
    ) {}

    public function log(string $message): void
    {
        $this->logs[] = $message;
    }
}
// Bind as scoped so it's re-resolved on every request
// $this->app->scoped(Logger::class);
```

---

### 2. Dependency Injection & Singleton

**Problem:** Singletons are resolved ONCE when the worker boots. If a singleton stores request state, the next request will receive the previous request's data (data bleed).

**Rules:**
- Be careful when resolving services registered as Singleton
- Ensure Singletons **do not store** user/request-specific state
- Use **Scoped binding** if an object depends on the current request state

```php
// In AppServiceProvider:

// ✅ Scoped: re-resolved on every request cycle
$this->app->scoped(CartService::class, function ($app) {
    return new CartService($app->make('request')->user());
});

// ❌ Singleton: DANGEROUS if it stores request state
$this->app->singleton(CartService::class, function ($app) {
    return new CartService($app->make('request')->user());
    // User from the first request will "stick" to all requests!
});
```

---

### 3. Constructors & Destructors

**Problem:** Global service constructors are only executed **once when the worker boots** (not on every request).

**Rules:**
- Do not place per-request initialization logic inside `__construct()` of Singleton/Long-lived services
- Use **method injection** for data that changes per-request

### DO: Method Injection for Request-Specific Data

```php
// ✅ SAFE: $request is injected per-method, always fresh
class OrderController extends Controller
{
    public function handle(Request $request, OrderService $service)
    {
        // $request belongs to the user currently accessing
        return $service->process($request->user());
    }
}
```

### DON'T: Constructor Injection for Request State

```php
// ❌ DANGEROUS IN OCTANE:
// Constructor may only run once at server start
// The injected $request may be stale (from the first request)
class OrderService
{
    protected $currentUser;

    public function __construct(Request $request)
    {
        $this->currentUser = $request->user();
    }
}
```

---

## Octane-Safety Checklist

Before deploying to Octane, ensure:

- [ ] No static properties storing request-specific data without cleanup
- [ ] No Singleton storing `$request`, `auth()->user()`, or session data
- [ ] Global service constructors do not perform per-request initialization
- [ ] All request-dependent services use **Scoped binding**
- [ ] No global variables that keep growing without limit
- [ ] File handles and database connections are cleaned up after completion

---

## Summary: PHP-FPM vs Octane Differences

| Aspect | PHP-FPM (Legacy) | Octane / FrankenPHP |
|---|---|---|
| Lifecycle | Boot → Handle → Terminate | Boot (once) → Handle → Handle → ... |
| Static property | Reset every request | Persists across requests |
| Singleton | New every request | Created once, reused |
| Memory | Auto-cleanup every request | Must be managed manually |
| Constructor | Executed every request | Executed once at boot |
