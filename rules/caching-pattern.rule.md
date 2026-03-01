# Rule: Caching Pattern & View Composer

You are an Engineer who builds a cache system that is **consistent**, **easy to invalidate**, and **does not cause stale data**. Cache is not the source of truth — the Model is the single source of truth. You also understand that View Composer is a **data glue, not a data processor**.

---

## Part 1: Cache Pattern Standardization

### Recommended Cache Architecture

```
Model (source of truth)
    ↓
Cache Class (stores ready-to-use data)
    ↓
Observer / Trait (auto-invalidation)
    ↓
View / Service (only READS cache, does not rebuild)
```

### DO: One Cache Class per Data Domain

```php
// ✅ Dedicated Cache Class with explicit key
class JenisOutputCache
{
    const KEY = 'jenis_output:options';

    public static function options(): array
    {
        return Cache::remember(
            self::KEY,
            now()->addDay(),
            fn () => JenisOutput::pluck('nama', 'id')->toArray()
        );
    }

    public static function clear(): void
    {
        Cache::forget(self::KEY);
    }
}
```

### DO: Auto-Invalidation via Model Observer

```php
// ✅ Cache automatically invalidated when data changes
class JenisOutputObserver
{
    public function created(JenisOutput $model): void
    {
        JenisOutputCache::clear();
    }

    public function updated(JenisOutput $model): void
    {
        JenisOutputCache::clear();
    }

    public function deleted(JenisOutput $model): void
    {
        JenisOutputCache::clear();
    }
}
```

### DO: Alternative Auto-Invalidation via Trait

```php
// ✅ Reusable trait for auto cache invalidation
namespace App\Models\Traits;

trait ClearCache
{
    protected static function bootClearCache(): void
    {
        static::updated(function ($model) {
            static::clearCache($model);
        });
        static::deleted(function ($model) {
            static::clearCache($model);
        });
    }

    abstract private static function clearCache($model): void;
}
```

### DON'T: Scattered Cache Without Standards

```php
// ❌ Cache in random controller — not centralized
public function index()
{
    return Cache::remember('jenis_output', 86400, function () {
        return JenisOutput::all(); // over-fetching too!
    });
}

// ❌ rememberForever without invalidation — stale data forever
Cache::rememberForever('jenis_output', function () {
    return JenisOutput::pluck('nama', 'id');
});
```

### Cache Principles (Mandatory)

| Principle | Explanation |
|---|---|
| Cache is not the source of truth | Model = single source of truth |
| Cache must be easy to clear | Use Cache Class with `clear()` |
| Cache must be centralized | One class per domain, not scattered |
| Cache must auto-invalidate | Use Observer or Trait |
| Cache key must be explicit & documented | Use constants, not random strings |

### Risks If Ignored

- Hard-to-reproduce bugs (only appear in production)
- Inconsistent behavior across modules
- Expensive & risky refactoring
- Undetected stale data

---

## Part 2: View Composer

### Danger of Misused View Composer

View Composer is called **every time a view/partial is rendered**. On a single page this can mean:
- 5–10x invocations
- Repeated permission loops
- Redundant queries

### DO: Memoize per-Request

```php
// ✅ Heavy computation ONLY ONCE per request
protected static $viewCache = null;

public function handle(Request $request, Closure $next)
{
    View::composer(['*::pages.*', '*::livewire.*'], function ($view) use ($request) {
        // Check if already computed in this request
        if (is_null(self::$viewCache)) {
            $viewData = $view->getData();
            self::$viewCache = Page::buildViewData(
                module: 'spmi',
                menu: Menu::navbar(),
                // ... other parameters
            );
        }
        // Reuse data from cache
        View::share(self::$viewCache);
    });

    return $next($request);
}
```

### DON'T: Heavy Computation on Every Render

```php
// ❌ Every time the view is rendered, heavy computation runs again
public function handle(Request $request, Closure $next)
{
    View::composer(['*::pages.*', '*::livewire.*'], function ($view) {
        $viewData = $view->getData();
        View::share(
            Page::buildViewData(
                module: 'spmi',
                menu: Menu::navbar(), // Query on every render!
                sidebar: $viewData['sidebar'] ?? null,
            )
        );
    });

    return $next($request);
}
```

### Rule of Thumb for View Composer

1. View Composer = **data glue**, not a data processor
2. If heavy → **cache** (at minimum per-request memoization)
3. If complex → **move to a Service**
4. Do not put heavy queries, permission loops, or recursive array builds in View Composer
