# Rule: Code Quality Principles

You are an Engineer who writes high-quality code based on team-agreed principles. Every line of code must be **readable**, **consistent**, **type-safe**, and **framework-aligned**. You use Laravel as it was designed, not against it.

---

## 1. Readable Before Clever

Code must be easy to read **before** being considered concise or "smart."

### DON'T: Too Clever

```php
// ❌ Technically correct, but hard to debug & unclear intent
$total = collect($orders)->filter(fn($o) => $o->status === 'paid')
    ->map(fn($o) => $o->items->sum('price'))
    ->sum();
```

### DO: Readable & Intentional

```php
// ✅ Longer, but intent is clear, easy to debug & breakpoint
$paidOrders = $orders->where('status', 'paid');

$totalRevenue = $paidOrders->sum(function ($order) {
    return $order->items->sum('price');
});
```

**Rule of thumb:** "If a reviewer has to pause for more than 3 seconds to understand a single line of code, that line needs to be simplified."

---

## 2. Expressive and Concise Code

Laravel provides helpers and shortcuts to improve code clarity. Use them when they make the code **more readable**, not just shorter.

### Preferred Patterns

| Instead Of | Use | Reason |
|---|---|---|
| `Session::get('key')` | `session('key')` | Shorter, idiomatic Laravel |
| `->orderBy('created_at', 'desc')` | `->latest()` | More expressive |
| `$user ? $user->name : null` | `optional($user)->name` | Null-safe, no conditional |
| `Request::input('key')` | `request('key')` | Shorter, idiomatic Laravel |
| `Config::get('app.name')` | `config('app.name')` | Shorter, idiomatic Laravel |

### Boundary

Do not force shortcuts when they reduce clarity. The rule from Section 1 still applies: **readable before clever**. If a helper obscures intent or makes debugging harder, use the explicit form.

**Rule of thumb:** "If a reviewer has to pause for more than 3 seconds to understand a single line of code, that line needs to be simplified."

---

## 3. Consistency Over Preference

Team consistency is more important than individual developer style.
- If the team agrees to use Service Class → everyone uses Service Class
- If the team agrees on a certain naming convention → everyone follows it

---

## 4. Single Source of Truth (SSOT)

There must be no duplication of business logic. The same rule MUST NOT be written in more than one place.

### DON'T: Duplicated Logic

```php
// ❌ In Controller:
if ($order->status !== 'paid') {
    abort(403);
}

// ❌ In another Service (SAME rule, written again):
if ($order->status !== 'paid') {
    throw new Exception('Invalid order');
}
```

### DO: Centralize in Service/Model

```php
// ✅ Rule written ONCE in Model/Service
class Order extends Model
{
    public function isPaid(): bool
    {
        return $this->status === 'paid';
    }
}

// Used anywhere:
if (! $order->isPaid()) { ... }
```

---

## 5. Single Responsibility Principle (SRP)

Every class and method should have only **one responsibility**.

**Signs of SRP violation:**
- Method is too long
- Too many if/else blocks
- Difficult to explain in one sentence

**Recommended practices:**
1. Separate validation, formatting, and business rules
2. Use small methods with clear names
3. Move complex logic to Service or Domain Layer

**"SRP is not about line count, but about reasons for change."**

---

## 6. Don't Repeat Yourself (DRY)

Every business rule must have a single source of truth.

**Mandatory DRY application:**

| Context | Solution |
|---|---|
| Repetitive Eloquent queries | Use **local scope** |
| Repetitive Blade templates | Use `@extends`, `@include`, `@component` |
| Repetitive business logic | Use **Service Class** |
| Shared behavior across models | Use **Trait** |

**DRY anti-patterns:**
- Copy-pasting queries across models
- Duplicating validation in controller and service
- Global helpers without domain context

---

## 7. Type Safety & Code Contracts

### Type Declaration (Mandatory)

```php
// ✅ Mandatory type hints on parameters and return values
declare(strict_types=1);

public function calculateDiscount(Order $order, float $percentage): float
{
    return $order->total * ($percentage / 100);
}

// ✅ Nullable type if it can be null
public function findUser(int $id): ?User
{
    return User::find($id);
}

// ✅ Union type (PHP 8+) if relevant
public function process(int|string $identifier): Result
{
    // ...
}
```

**Engineering rule:** "A method without a return type is considered an incomplete contract."

### DocBlock

Used for:
1. Documentation
2. Static analysis (PHPStan, Psalm)
3. Additional context beyond native types (e.g., `@param array<int, OrderItem> $items`)

---

## 8. Code Style & Laravel Pint

**Laravel Pint** is mandatory as the standard formatter.

**Rules:**
1. Pint must be run **before every commit**
2. No subjective formatting discussions in code reviews
3. Use the default Laravel `pint` preset

**Benefits:**
- Consistency across the team
- Reviews focus on logic, not spaces/indentation

---

## 9. Prohibition of Overriding Laravel Core Features

### Prohibited

- Modifying internal Laravel behavior
- Overriding core classes without strong architectural justification

### Official Principle: **"Extend, don't modify"**

### Risks of Overriding

| Risk | Impact |
|---|---|
| Issues during Laravel upgrades | Unexpected breaking changes |
| Hidden bugs | Difficult to trace due to non-standard behavior |
| Dependency on internal implementation | Fragile code |

### Use Alternatives

- **Service Provider** — for binding and bootstrapping
- **Event / Listener** — for hooking into Laravel lifecycle
- **Macro** — for extending classes without modifying
- **Custom abstraction** — for additional layers on top of the framework
