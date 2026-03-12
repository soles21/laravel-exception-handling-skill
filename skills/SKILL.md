---
name: Laravel Exception Handling Expert
description: Senior-level exception handling patterns for Laravel. Use this skill whenever writing controllers, services, form requests, or exception handling in Laravel. Triggers on any task involving try-catch, exceptions, error handling, validation, API responses, or Handler.php. Avoid junior patterns like wrapping everything in try-catch — use these 4 patterns instead.
compatible_agents:
  - Claude Code
  - Cursor
tags:
  - laravel
  - php
  - exceptions
  - error-handling
  - validation
  - api
---

# Laravel Exception Handling Expert

## Context

You are writing exception handling for a Laravel application (PHP 8.3+, Laravel 13). The goal is to handle failures at the right layer, with the right tool — not to wrap everything in try-catch. Use these four patterns to determine which approach fits each situation.

## The Core Mental Model

Categorise every failure before writing a single line of handling code:

- **Exceptional** — Should not happen under normal operation: DB connection drops, third-party API timeout, out of memory. These deserve try-catch, logging, and alerts.
- **Expected** — Normal business outcomes: user not found, order already processed, payment declined. These deserve Form Requests, custom exceptions, or Result objects — never try-catch.

---

## Pattern 1: Validate First, Catch Never

If you are catching an exception caused by invalid input, you have a **validation problem**, not an exception problem. Use Laravel Form Requests to enforce rules before data reaches the service layer.

```php
// ❌ WRONG — catching what validation should have prevented
public function store(Request $request): JsonResponse
{
    try {
        $user = $this->userService->create($request->all());
        return response()->json($user, 201);
    } catch (\InvalidArgumentException $e) {
        return response()->json(['error' => $e->getMessage()], 422);
    }
}

// ✅ CORRECT — validate at the door, nothing bad gets inside
public function store(CreateUserRequest $request): JsonResponse
{
    $user = $this->userService->create($request->validated());
    return response()->json($user, 201);
}
```

```php
// app/Http/Requests/CreateUserRequest.php
class CreateUserRequest extends FormRequest
{
    public function rules(): array
    {
        return [
            'name'  => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'unique:users,email'],
            'age'   => ['required', 'integer', 'min:18'],
        ];
    }

    public function messages(): array
    {
        return [
            'age.min' => 'Must be at least 18 years old.',
        ];
    }
}
```

Laravel automatically returns a 422 JSON response when validation fails on API routes — no try-catch needed.

---

## Pattern 2: A Custom Exception Hierarchy

Catching `\Exception` or `\Throwable` and then trying to figure out what went wrong means your exceptions are not specific enough. Build a hierarchy where every thrown exception is self-documenting.

```php
// ❌ WRONG — generic exceptions tell you nothing
try {
    $this->orderService->process($order);
} catch (\Exception $e) {
    Log::error('Something failed: ' . $e->getMessage());
    return response()->json(['error' => 'Server error'], 500);
}

// ✅ CORRECT — a structured exception hierarchy
```

```php
// app/Exceptions/AppException.php
abstract class AppException extends \RuntimeException
{
    public function __construct(
        string $message,
        public readonly int $statusCode = 500,
        public readonly string $errorCode = 'INTERNAL_ERROR',
        ?\Throwable $previous = null,
    ) {
        parent::__construct($message, 0, $previous);
    }
}

// app/Exceptions/NotFoundException.php
class NotFoundException extends AppException
{
    public function __construct(string $resource, int|string $id)
    {
        parent::__construct(
            message: "{$resource} not found with id: {$id}",
            statusCode: 404,
            errorCode: 'NOT_FOUND',
        );
    }
}

// app/Exceptions/BusinessRuleException.php
class BusinessRuleException extends AppException
{
    public function __construct(string $rule)
    {
        parent::__construct(
            message: "Business rule violated: {$rule}",
            statusCode: 409,
            errorCode: 'BUSINESS_RULE_VIOLATED',
        );
    }
}
```

```php
// Service layer — no try-catch needed
public function processOrder(int $orderId): Order
{
    $order = Order::find($orderId)
        ?? throw new NotFoundException('Order', $orderId);

    if ($order->isAlreadyProcessed()) {
        throw new BusinessRuleException('Order already processed');
    }

    return tap($order)->update(['status' => 'processed']);
}
```

---

## Pattern 3: Centralised Exception Rendering in Handler

If you are writing the same catch block in more than one controller, move all exception-to-response mapping into `bootstrap/app.php` (Laravel 11+) or `app/Exceptions/Handler.php`.

```php
// ❌ WRONG — duplicated catch logic across controllers
public function show(int $id): JsonResponse
{
    try {
        return response()->json($this->orderService->findById($id));
    } catch (NotFoundException $e) {
        return response()->json(['error' => $e->getMessage()], 404);
    } catch (\Exception $e) {
        return response()->json(['error' => 'Server error'], 500);
    }
}
```

```php
// ✅ CORRECT — bootstrap/app.php (Laravel 11+)
->withExceptions(function (Exceptions $exceptions) {

    // Map all AppException subclasses to JSON automatically
    $exceptions->render(function (AppException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'error' => $e->errorCode,
                'message' => $e->getMessage(),
            ], $e->statusCode);
        }
    });

    // Validation failures
    $exceptions->render(function (ValidationException $e, Request $request) {
        if ($request->expectsJson()) {
            return response()->json([
                'error' => 'VALIDATION_FAILED',
                'message' => 'The given data was invalid.',
                'errors' => $e->errors(),
            ], 422);
        }
    });

    // Report unexpected exceptions, silence known ones
    $exceptions->report(function (AppException $e) {
        // Known domain exceptions — no Sentry/Bugsnag noise
        return false;
    });
})
```

```php
// Now controllers are clean — exceptions handle themselves
public function show(int $id): JsonResponse
{
    return response()->json($this->orderService->findById($id));
}
```

---

## Pattern 4: Result Objects for Expected Failures

Not every failure is exceptional. "User not found" is not an error — it is a normal outcome. Using exceptions for normal control flow makes call sites harder to reason about. Use a Result object when the caller needs to act differently on success vs failure **without it being a bug**.

```php
// ❌ WRONG — exception thrown for an expected scenario
public function findUser(int $id): User
{
    return User::find($id)
        ?? throw new NotFoundException('User', $id);
    // Every caller must now try-catch or let it bubble
}

// ✅ CORRECT — Result communicates success and failure explicitly
```

```php
// app/Support/Result.php
/**
 * @template T
 */
final class Result
{
    private function __construct(
        private readonly bool $success,
        private readonly mixed $value,
        private readonly ?string $error,
    ) {}

    /** @param T $value */
    public static function ok(mixed $value): self
    {
        return new self(true, $value, null);
    }

    public static function failure(string $error): self
    {
        return new self(false, null, $error);
    }

    public function isSuccess(): bool { return $this->success; }

    /** @return T */
    public function value(): mixed { return $this->value; }

    public function error(): ?string { return $this->error; }
}
```

```php
// Service
public function findUser(int $id): Result
{
    $user = User::find($id);

    return $user
        ? Result::ok($user)
        : Result::failure("User not found with id: {$id}");
}

// Controller — caller knows exactly what happened
public function show(int $id): JsonResponse
{
    $result = $this->userService->findUser($id);

    if (! $result->isSuccess()) {
        return response()->json(['message' => $result->error()], 404);
    }

    return response()->json($result->value());
}
```

Use Result when the "not found" or "failed" case is **expected business logic**. Use `NotFoundException` (Pattern 2 + 3) when the absence truly indicates a bug or a bad request that should be surfaced uniformly.

---

## Rules

- Never catch `\Exception` or `\Throwable` in service or repository classes — let exceptions bubble to the handler.
- Never use try-catch to handle input that a Form Request could have rejected.
- Never duplicate `catch` blocks across controllers — register renderers in `bootstrap/app.php`.
- Always extend a base `AppException` so the central handler can map domain exceptions to HTTP responses without `instanceof` chains.
- Use `throw new X() ?? throw` (null coalesce throw, PHP 8+) instead of `if (!$model) throw`.
- Silence known domain exceptions in `report()` to keep Sentry/Bugsnag signal clean.
- Return `Result` objects from service methods where the failure is expected and the caller must branch on it.
- Reserve try-catch for genuine I/O boundaries: external HTTP calls, filesystem writes, queue publishing.

## Anti-Patterns

- `catch (\Exception $e) { return response()->json(['error' => $e->getMessage()], 500); }` — leaks stack traces, hides root cause.
- Silent catch: `catch (\Exception $e) {}` — errors vanish, incidents follow.
- Try-catch inside a loop — masks which iteration failed.
- Throwing `NotFoundException` for a search/filter that returns zero results — that is not an error.
- Logging inside the service and also inside the handler — produces duplicate log entries.

## References

- [Laravel Error Handling](https://laravel.com/docs/errors)
- [Laravel Form Requests](https://laravel.com/docs/validation#form-request-validation)
- [PHP 8 throw expression](https://www.php.net/manual/en/language.exceptions.php)
