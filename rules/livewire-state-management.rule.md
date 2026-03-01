# Rule: Livewire State Management

You are an Engineer who builds interactive components using Laravel Livewire. You understand that Livewire performs **hydrate & dehydrate** on all public properties during every interaction, so you must be highly selective in determining what becomes a public property and what should remain a Computed Property.

## Core Principle

**State ≠ Reference Data. Public Property ≠ Static Data. Easy to write ≠ Safe for performance.**

---

## Separating State and Reference Data

Every Livewire component MUST separate:

| Data Type | Store In | Reason |
|---|---|---|
| User input, values that change (search, scores) | `public property` | Needs to sync with browser |
| Static data, read-only, large data for UI rendering | `#[Computed]` property / Service / Cache | No sync needed, saves payload |

---

## DO: Use Computed Property for Large Data

```php
use Livewire\Attributes\Computed;

class AssessmentForm extends Component
{
    // Only interactive state as public property
    public array $scores = [];
    public string $search = '';

    // Large & read-only data → Computed Property
    #[Computed]
    public function rubric()
    {
        return Rubric::with('indicators.options')->get();
    }

    public function render()
    {
        return view('livewire.assessment-form');
    }
}
```

**Why this is correct:**
- Livewire only sends the change delta for `$scores` and `$search`
- Rubric data is calculated on-demand, not included in hydrate/dehydrate
- Update payload remains small even if rubric data contains hundreds of rows

---

## DON'T: Store Large Data in Public Property

```php
// ❌ DO NOT DO THIS
class AssessmentForm extends Component
{
    public array $rubric = []; // Large data + mixed with state

    public function mount()
    {
        // 50 elements × 5 indicators × 5 options = thousands of items
        $this->rubric = Rubric::with('indicators.options')
            ->get()
            ->toArray();
    }
}
```

**Why this is wrong:**
- Livewire must hydrate & dehydrate `$rubric` on EVERY interaction
- Update payload can reach **megabytes**
- High UI latency, increased server load
- Mixes interactive state (`selected_score`) with reference data

---

## Performance Impact

| Aspect | Public Property (❌) | Computed Property (✅) |
|---|---|---|
| Payload per interaction | Megabytes (all data) | Kilobytes (only state delta) |
| Hydrate/Dehydrate | All data every request | Only minimal state |
| Scalability | Degrades as data grows | Stable |
| UI Responsiveness | Slow | Fast |

---

## When Data May Be a Public Property

Data should ONLY be a public property if it meets **all** of the following criteria:

1. The data **changes** based on user interaction
2. The change **needs to be known by the server** (not just a UI toggle)
3. The data size is **small** (not a list of hundreds of items)

**Correct examples as public property:**
- `$search` (search input string)
- `$selectedId` (selected item ID)
- `$scores` (array of user-entered values)
- `$perPage` (number of items per page)

**INCORRECT examples as public property:**
- List of all provinces/cities → use Computed / Cache
- Assessment rubric data → use Computed
- Large dropdown option lists → use Computed / Cache
