# Rule: Query Performance & Data Fetching

You are a performance-conscious Engineer. Every query you write must consider **query count**, **memory usage**, and **scalability**. You never rely on implicit lazy loading, never over-fetch data, and never use JSON as an internal data transformation tool.

**Golden Rule: "Fetch as little data as possible, as close to the database as possible."**

---

## 1. Eager Loading — Prevent N+1 Query

### DO: Use `with()` for Eager Loading

```php
// ✅ One query (or minimal) for the entire relation hierarchy
$organizations = Organization::with([
    'branches.departments.teams.projects'
])->get();

// ✅ Define relations in the Model, not in controller/service
class Organization extends Model
{
    public function branches()
    {
        return $this->hasMany(Branch::class);
    }
}
```

### DON'T: Query Inside a Loop

```php
// ❌ N+1: each iteration triggers a new query
$organizations = Organization::all();
foreach ($organizations as $org) {
    $branches = $org->branches; // Lazy load per iteration!
    foreach ($branches as $branch) {
        $departments = $branch->departments; // Another query!
    }
}
```

**Impact without eager loading:**
- Queries executed repeatedly (can be hundreds/thousands)
- Latency increases dramatically
- The problem isn't Laravel, it's the **usage pattern**

**Principles:**
1. Avoid N+1 Query — always
2. Relations are written once in the Model, used forever
3. Explicit query > implicit loading

---

## 2. Using `pluck()` for Efficiency

### DO: Use `Model::pluck()` Directly

```php
// ✅ Query directly for needed columns, no model hydration
$emails = Employee::pluck('email', 'id')->toArray();
// SQL: SELECT `email`, `id` FROM `employees`
```

### DON'T: `all()` Then `pluck()`

```php
// ❌ Over-fetching: SELECT * then filter in PHP
$emails = Employee::all()->pluck('email', 'id')->toArray();
// SQL: SELECT * FROM `employees` → create full Collection → then pluck
```

**Comparison:**

| Aspect | `Model::pluck()` (✅) | `Model::all()->pluck()` (❌) |
|---|---|---|
| SQL | `SELECT email, id` | `SELECT *` |
| Memory | Low (only 2 columns) | High (all columns + model objects) |
| Hydration | None | All records hydrated |
| Speed | Fast | Slower as data grows |

**Anti-pattern:** "Since I eventually need a Collection, I'll use `all()` first." This is wrong for large datasets.

### When Multiple Columns Are Needed

If you need more than a key-value pair but not all columns, use `select()`:

```php
// When you need specific columns (not just key-value):
$users = User::select('id', 'name', 'email')->where('active', true)->get();
```

This fetches only the required columns without full model hydration overhead.

---

## 3. Prohibition of `json_encode` / `json_decode` for Data Transformation

### DO: Use Laravel Collection API / DTO / Resource

```php
// ✅ If using Query Builder:
$results = DB::table('table')->get()->toArray();

// ✅ If using Eloquent:
$results = Model::query()->get()->toArray();

// ✅ If still using DB::select():
$results = collect($resultQuery)
    ->map(fn ($row) => (array) $row)
    ->toArray();
```

### DON'T: JSON Encode-Decode for Casting

```php
// ❌ Code Smell: JSON as an internal conversion tool
$resultQuery = DB::select($sql, ['id' => $id]);
return json_decode(json_encode($resultQuery), true);
```

**Why this is a Code Smell:**
1. `json_encode()` → expensive on CPU, creates a copy of data
2. Not type-safe — loses type information
3. Hides poor data design
4. Usually a sign of: unclear data structure, vague layer boundaries, no DTO/Resource layer

**Clean Code Principle:** "Do not use a data exchange format (JSON) as an internal manipulation tool."

---

## Checklist Before Writing Queries

- [ ] Are all required relations eager loaded with `with()`?
- [ ] Are only necessary columns being fetched? (use `select()` or `pluck()`)
- [ ] Are there queries inside loops? (move to eager loading)
- [ ] Is there any `json_encode`/`json_decode` for conversion? (replace with Collection/DTO)
- [ ] Is `all()` being used when only a few columns are needed?
