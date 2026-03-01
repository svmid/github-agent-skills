# Rule: Naming Convention & MVC + Service Architecture

You are an Engineer who works in a team with an **MVC + Service Layer** architecture (without a Repository layer). Every piece of code you write must follow consistent naming standards and place logic in the appropriate layer. No business rules in Blade, no complex queries in Controller.

---

## Naming Standards

### General Conventions

Must follow **PSR-1**, **PSR-4**, and **PSR-12**.

| Element | Rule | Example |
|---|---|---|
| Controller | Singular | `ArticleController` |
| Model | Singular | `User` |
| Route | Plural | `/articles` |
| Table | Plural snake_case | `article_comments` |
| Variable | camelCase | `$activeUsers` |
| Collection | Plural & descriptive | `$activeUsers`, `$paidOrders` |
| Method | camelCase & verb-based | `calculateTotal()`, `sendNotification()` |
| View | snake_case | `show_filtered.blade.php` |

**Engineering rule:** "If a name cannot be explained without a comment, then the name is wrong."

---

## Language Usage Rules

Consistency in language usage across the codebase is mandatory. The following table defines which elements must use English and which must use Indonesian.

| Must Use English | Must Use Indonesian |
|---|---|
| Class names | Table and migration names (table names, field names) |
| Method names | Button text and messages |
| Variable names | Alert text and messages |
| API endpoints | Notification text and messages |
| Enum values | Email content sent to users |
| Code comments (Indonesian is also permitted) | All view labels and UI text |

**Rationale:** Code-level elements (classes, methods, variables) use English for universal readability and framework alignment. User-facing text and database schema use Indonesian to match the business domain and end-user language.

---

## MVC + Service Layer Architecture

### Folder Structure

```
app/
├── Models/
├── Http/
│   ├── Controllers/
│   ├── Requests/
│   └── Middleware/
├── Services/
└── Providers/
```

**Not used:**
```
app/Repositories/    ← NOT USED
```

**Reasons for not using Repository:**
1. Eloquent already functions as a data access layer
2. Repository often becomes a passive wrapper without added value
3. Adds complexity without a real need

Repository is only considered if: multiple data sources, highly complex cross-domain queries, or a need to swap the persistence layer.

---

## Responsibilities of Each Layer

### Model

**Focus on:**
- Table representation
- Relations between models
- Query scopes
- Simple domain helpers (small pure logic)

```php
// ✅ Correct Model
class Order extends Model
{
    protected $fillable = ['user_id', 'reference', 'status'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function scopePaid($query)
    {
        return $query->where('status', 'paid');
    }

    public function isPaid(): bool
    {
        return $this->status === 'paid';
    }
}
```

**What MUST NOT be in Model:**
- Accessing Request
- Sending emails
- Running complex transactions
- Cross-domain queries

---

### Controller

**Focus on:**
- Receiving request
- Validation (via Form Request)
- Calling Service
- Returning response

```php
// ✅ Correct Controller — thin & deterministic
class OrderController extends Controller
{
    public function store(StoreOrderRequest $request, OrderService $service)
    {
        $order = $service->create($request->validated());
        return redirect()->route('orders.show', $order);
    }
}
```

**What MUST NOT be in Controller:**
- Complex queries
- Domain logic loops
- Business calculations
- Heavy transaction orchestration

**Problem indicator:** If a controller has more than ~15–20 lines of active logic, it's a sign that business logic has not been moved to a Service.

---

### Service (Business Logic Center)

**Responsible for:**
- Domain validation (beyond request validation)
- Transaction orchestration
- Integration with external services
- Enforcing business invariants

```php
// ✅ Correct Service
class OrderService
{
    public function create(array $data): Order
    {
        return DB::transaction(function () use ($data) {
            $order = Order::create($data);

            if (! $order->isPaid()) {
                throw new DomainException('Order must be paid.');
            }

            $this->applyReward($order);
            return $order;
        });
    }

    protected function applyReward(Order $order): void
    {
        // domain logic
    }
}
```

---

## Complete Request Flow

```
Request comes in
    ↓
FormRequest → format validation
    ↓
Controller → calls Service
    ↓
Service → runs business logic + transaction
    ↓
Model → persists data
    ↓
Response returned
```

Deterministic & easy to trace flow.

---

## Prohibited Anti-Patterns

| Anti-Pattern | Why It's Wrong |
|---|---|
| Business rules in Blade | Hidden logic, cannot be tested |
| Complex queries in Controller | Controller becomes fat, hard to maintain |
| Request logic in Model | Model should not know about HTTP |
| Scattered cache logic | Hard to invalidate, stale data |
| Service as thin wrapper only | Adds no value, adds complexity |
| Manual join without Eloquent relations | Not framework-aligned, hard to maintain |

### DO: Use Eloquent Relations

```php
// ✅ Framework-aligned
class Order extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
$orders = Order::with('user')->get();
```

### DON'T: Manual Join

```php
// ❌ Against Laravel's design
DB::table('orders')
    ->join('users', 'orders.user_id', '=', 'users.id')
    ->get();
```
