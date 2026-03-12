# Laravel Exception Handling Skill

> Senior-level exception handling patterns for Laravel. Stop wrapping everything in try-catch — handle failures at the right layer, with the right tool.

## Install

```bash
php artisan boost:add-skill soles21/laravel-exception-handling-skill
```

## What This Skill Teaches

Most codebases default to try-catch everywhere. This skill encodes four patterns that senior Laravel developers use instead — keeping controllers thin, errors self-documenting, and production incidents visible.

### Pattern 1 — Validate First, Catch Never

Use [Form Requests](https://laravel.com/docs/validation#form-request-validation) to reject invalid input before it reaches the service layer. If you are catching an exception caused by bad input, you have a validation problem, not an exception problem.

### Pattern 2 — Custom Exception Hierarchy

Build a typed exception hierarchy rooted at a base `AppException`. Every thrown exception becomes self-documenting: what failed, why it failed, and what HTTP status to return — with zero guesswork.

```
AppException
├── NotFoundException       → 404 NOT_FOUND
├── BusinessRuleException   → 409 BUSINESS_RULE_VIOLATED
└── ... your domain exceptions
```

### Pattern 3 — Centralised Exception Rendering

Register all exception-to-response mappings once in `bootstrap/app.php` via `withExceptions()`. Controllers become one-liners. No duplicated catch blocks across 15 controllers.

### Pattern 4 — Result Objects for Expected Failures

"User not found" is not a bug — it is a normal outcome. Return a `Result<T>` object when the caller needs to branch on success vs failure without it being exceptional. Reserve exceptions for things that should not happen.

## The Mental Model

| Failure type | Tool |
|---|---|
| Invalid input | Form Request |
| Domain rule violation | Custom Exception + central handler |
| Expected business outcome | Result object |
| True I/O failure (DB down, API timeout) | try-catch + logging |

## Compatibility

- PHP 8.2+
- Laravel 11 / 12
- Compatible with Claude Code and Cursor

## License

MIT
