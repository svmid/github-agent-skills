# Rule: Foundational Engineering Principles

You are an Engineer working on a Laravel 12+ project. Every technical decision you make must be grounded in these six foundational principles. These principles are not optional — they form the baseline for all code written, reviewed, and maintained by the team.

---

## 1. Performance-Aware Engineering

Every data operation must consider its impact on:

| Dimension | Question to Ask |
|---|---|
| Query count | How many queries does this operation generate? |
| Memory usage | How much memory does this consume? |
| CPU cost | Is there unnecessary computation? |
| Payload size | How large is the data sent to the client? |
| Scalability | Will this hold up when data or traffic increases 10x? |

Optimization must be done **proactively and consciously**, not reactively after the system degrades.

**Anti-pattern:** "It works fine now with small data." This is not a valid justification for ignoring performance characteristics.

---

## 2. Single Source of Truth

All business rules belong in the **Service Layer**. There must be exactly one authoritative location for each domain rule.

- Model is **not** a business orchestrator — it represents data and relations.
- Controller is **not** a place for domain logic — it handles request/response flow.
- Blade is **not** a place for business rules — it handles presentation only.

```php
// DO: Business rule written once in Service
class OrderService
{
    public function canBeRefunded(Order $order): bool
    {
        return $order->isPaid() && $order->created_at->diffInDays(now()) <= 30;
    }
}

// DON'T: Same rule duplicated in Controller and Job
// Controller: if ($order->status === 'paid' && ...) { ... }
// Job: if ($order->status === 'paid' && ...) { ... }
```

---

## 3. Explicit Over Implicit

Relations, caching, and data transformations must be **explicitly declared**, never assumed or left to defaults.

| Context | Explicit (DO) | Implicit (DON'T) |
|---|---|---|
| Relations | `with('user')` eager loading | Relying on lazy loading default |
| Cache | Dedicated Cache Class with clear key | `Cache::remember()` inline in controller |
| Data shape | `pluck('email', 'id')` | `all()` then filter in PHP |

Relying on implicit behavior (e.g., lazy loading) is a source of **hidden performance regressions** that only surface in production.

---

## 4. Framework-Aligned Development

Laravel must be used **as designed**. Do not override, bypass, or fight the framework's architecture.

**Official principle: "Extend, don't modify."**

| Approach | Use |
|---|---|
| Service Provider | For binding and bootstrapping |
| Event / Listener | For hooking into Laravel lifecycle |
| Macro | For extending classes without modifying source |
| Custom abstraction | For additional layers on top of the framework |

**Prohibited:**
- Overriding internal Laravel behavior
- Overriding core classes without strong architectural justification
- Using manual `DB::table()->join()` when Eloquent relations exist

---

## 5. Readable Before Clever

Code must be easy to understand **before** being considered concise or elegant. Laravel provides many shortcuts, but readability remains the priority.

**Rule of thumb:** "If a reviewer has to pause for more than 3 seconds to understand a single line of code, that line needs to be simplified."

```php
// DON'T: Too clever — unclear intent, hard to debug
$total = collect($orders)->filter(fn($o) => $o->status === 'paid')
    ->map(fn($o) => $o->items->sum('price'))
    ->sum();

// DO: Readable — clear intent, easy to breakpoint
$paidOrders = $orders->where('status', 'paid');
$totalRevenue = $paidOrders->sum(function ($order) {
    return $order->items->sum('price');
});
```

---

## 6. Deterministic and Scalable Architecture

Every request flow in Laravel must be:

1. **Clear in its boundaries** — each layer has a defined responsibility
2. **Easy to trace** — the path from request to response is predictable
3. **Free of mixed responsibilities** — no layer does another layer's job
4. **Stable under load** — behavior does not degrade as data and traffic increase

```
Request → FormRequest (validation) → Controller (routing) → Service (business logic) → Model (persistence) → Response
```

This flow must remain **deterministic**: given the same input, the same path is always followed, making the system predictable and debuggable.
