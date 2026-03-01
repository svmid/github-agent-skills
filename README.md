# 🧠 Laravel Engineering Playbook

A markdown-based knowledge base for **GitHub Copilot Chat**. Contains engineering rules and prompt templates extracted from the internal team's **Laravel 12+ Guidelines**.

No build step, no extensions — just clone into your project and start using it.

---

## What's Inside?

```
rules/        → Engineering rules that MUST be followed
prompts/      → AI task templates with personas for various scenarios
```

### Rules

| File | Topic |
|------|-------|
| `livewire-state-management` | Computed vs Public Property, minimal state |
| `query-performance` | N+1, eager loading, pluck, json_encode anti-pattern |
| `file-upload-transaction` | File upload outside DB transaction |
| `caching-pattern` | Cache Class, Observer invalidation, View Composer |
| `naming-architecture` | PSR, MVC + Service Layer, naming convention, language usage |
| `code-quality-principles` | SRP, DRY, type safety, expressive code, Laravel Pint |
| `octane-frankenphp` | Memory leaks, singleton, scoped binding |
| `engineering-principles` | Foundational engineering principles (performance-aware, SSOT, explicit over implicit) |

### Prompts (Task Templates)

| Prompt | Purpose |
|--------|---------|
| `/new-feature` | Build a new feature following standards |
| `/code-review` | Review code / merge request |
| `/performance-audit` | Audit performance bottlenecks |
| `/refactor` | Clean up technical debt |
| `/livewire-component` | Build a Livewire component |
| `/caching-strategy` | Design a caching strategy |
| `/database-optimization` | Optimize queries & data access |

---

## Installation

Clone this repository as a `.github` folder in the root of your Laravel project:

```bash
git clone https://github.com/BayuFirmansyaah/github-agents-skilss.git .github
```

If you don't need git history:

```bash
rm -rf .github/.git
```

Or as a git submodule (to keep receiving updates):

```bash
git submodule add https://github.com/BayuFirmansyaah/github-agents-skilss.git .github
```

**Done!** Open VS Code, open Copilot Chat, and the knowledge base is ready to use.

---

## Usage

### Using Prompts

In GitHub Copilot Chat, simply type `/` followed by the prompt name:

```
/new-feature Create an Inventory module with full CRUD
```

```
/code-review Review file OrderController.php
```

```
/performance-audit Check the dashboard page performance
```

```
/refactor Fix OrderService to comply with SRP
```

```
/livewire-component Build an assessment form component with rubric
```

```
/caching-strategy Design caching for province master data
```

```
/database-optimization Optimize queries in ReportService
```

### Referencing Rules

You can also reference rules directly:

```
@workspace follow rules query-performance, optimize the queries in this file
```

```
@workspace based on rules naming-architecture, review this folder structure
```

### Combining Prompt + Context

```
@workspace /code-review review changes in branch feature/payment
```

```
@workspace /new-feature create a PDF report export feature, follow rules caching-pattern
```

---

## Customization

### Adding New Rules

Create a file in `rules/` with the following format:

```markdown
# Rule: [Rule Name]

[Context paragraph — explain the core principle]

## DO: [Correct Practice]

​```php
// Example of correct code
​```

## DON'T: [Incorrect Practice]

​```php
// Example of incorrect code
​```
```

### Adding New Prompts

Create a file in `prompts/` with the format `prompt-name.prompt.md`:

```markdown
# Prompt: [Prompt Name]

> **Persona:** [AI role when executing this prompt]
> **Use when:** [When to use this prompt]

## Who You Are
[Persona description]

## Mandatory Rules
- [Rule 1](../rules/rule-name.rule.md)
- [Rule 2](../rules/rule-name.rule.md)

## Workflow
### Step 1: ...
### Step 2: ...

## Expected Output
[Output format]
```

### Adapting for Other Stacks

Fork this repository and replace the contents of `rules/` and `prompts/` with your own stack's standards. The folder structure and file format can remain the same.

---

## Contributing

Contributions are open to everyone!

### Contribution Steps

1. **Fork** this repository
2. **Clone** your fork locally
   ```bash
   git clone https://github.com/<your-username>/github-agents-skilss.git
   ```
3. **Create a branch** from `master`
   ```bash
   git checkout -b feat/contribution-name
   ```
4. **Make changes** — add prompts, rules, or improve existing ones
5. **Test** — clone into a Laravel project as `.github` and try it in Copilot Chat
6. **Commit** with [conventional commits](https://www.conventionalcommits.org/)
   ```bash
   git commit -m "feat(prompts): add deployment-checklist prompt"
   git commit -m "fix(rules): update caching-pattern examples"
   ```
7. **Push** and open a **Pull Request**
   ```bash
   git push origin feat/contribution-name
   ```

### What Can Be Contributed?

| Type | Location | File Name Format |
|------|----------|-----------------|
| New rule | `rules/` | `topic-name.rule.md` |
| New prompt | `prompts/` | `task-name.prompt.md` |
| Improvements | Existing files | — |

### Review Guidelines

- Ensure there are no duplicates with existing files
- Every prompt must reference at least 1 rule
- Use English for consistency
- Include clear code examples (DO/DON'T)

---

## License

MIT
