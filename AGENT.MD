# Laravel Exception Handling — Extended Reference

This file is referenced by `SKILL.md` for deeper context on advanced scenarios. Read the relevant section when the task requires it.

---

## Table of Contents

1. [Full Bootstrap/App Registration](#1-full-bootstrapapp-registration)
2. [Exception Hierarchy — Full Boilerplate](#2-exception-hierarchy--full-boilerplate)
3. [Result Object — Advanced Usage](#3-result-object--advanced-usage)
4. [Handling Third-Party / I/O Boundaries](#4-handling-third-party--io-boundaries)
5. [Queue & Job Exception Handling](#5-queue--job-exception-handling)
6. [Logging Strategy](#6-logging-strategy)
7. [API vs Web Response Splitting](#7-api-vs-web-response-splitting)
8. [Testing Exception Behaviour](#8-testing-exception-behaviour)
9. [Common Mistakes Checklist](#9-common-mistakes-checklist)

---

## 1. Full Bootstrap/App Registration

Complete `bootstrap/app.php` exception block for a JSON API application:

```php
->withExceptions(function (Exceptions $exceptions) {

    // ── Domain exceptions ─────────────────────────────────────────
    $exceptions->render(function (AppException $e, Request $request) {
        if (! $request->expectsJson()) {
            return null; // fall through to web handler
        }

        return response()->json([
            'error'   => $e->errorCode,
            'message' => $e->getMessage(),
        ], $e->statusCode);
    });

    // ── Validation ────────────────────────────────────────────────
    $exceptions->render(function (ValidationException $e, Request $request) {
        if (! $request->expectsJson()) {
            return null;
        }

        return response()->json([
            'error'   => 'VALIDATION_FAILED',
            'message' => 'The given data was invalid.',
            'errors'  => $e->errors(),
        ], 422);
    });

    // ── Model not found (auto-thrown by ->findOrFail()) ───────────
    $exceptions->render(function (ModelNotFoundException $e, Request $request) {
        if (! $request->expectsJson()) {
            return null;
        }

        $model = class_basename($e->getModel());

        return response()->json([
            'error'   => 'NOT_FOUND',
            'message' => "{$model} not found.",
        ], 404);
    });

    // ── Auth ──────────────────────────────────────────────────────
    $exceptions->render(function (AuthenticationException $e, Request $request) {
        if (! $request->expectsJson()) {
            return null;
        }

        return response()->json(['error' => 'UNAUTHENTICATED', 'message' => 'Unauthenticated.'], 401);
    });

    $exceptions->render(function (AuthorizationException $e, Request $request) {
        if (! $request->expectsJson()) {
            return null;
        }

        return response()->json(['error' => 'FORBIDDEN', 'message' => 'This action is unauthorized.'], 403);
    });

    // ── Silence known domain exceptions from Sentry/Bugsnag ───────
    $exceptions->report(function (AppException $e) {
        return false; // do not report — these are expected
    });

    // ── Report unexpected exceptions ──────────────────────────────
    $exceptions->report(function (\Throwable $e) {
        // Sentry, Bugsnag, etc. auto-integrated via provider
        // Add any extra context here if needed
    });
})
```

---

## 2. Exception Hierarchy — Full Boilerplate

Copy this structure into `app/Exceptions/`:

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
        parent::__construct("{$resource} not found with id: {$id}", 404, 'NOT_FOUND');
    }
}

// app/Exceptions/BusinessRuleException.php
class BusinessRuleException extends AppException
{
    public function __construct(string $rule)
    {
        parent::__construct("Business rule violated: {$rule}", 409, 'BUSINESS_RULE_VIOLATED');
    }
}

// app/Exceptions/ConflictException.php
class ConflictException extends AppException
{
    public function __construct(string $message)
    {
        parent::__construct($message, 409, 'CONFLICT');
    }
}

// app/Exceptions/UnprocessableException.php
class UnprocessableException extends AppException
{
    public function __construct(string $message)
    {
        parent::__construct($message, 422, 'UNPROCESSABLE');
    }
}

// app/Exceptions/ExternalServiceException.php
// Use for third-party API failures that bubble up to the domain
class ExternalServiceException extends AppException
{
    public function __construct(string $service, string $reason, ?\Throwable $previous = null)
    {
        parent::__construct(
            "{$service} is currently unavailable: {$reason}",
            503,
            'EXTERNAL_SERVICE_ERROR',
            $previous,
        );
    }
}
```

---

## 3. Result Object — Advanced Usage

### Chaining Results

```php
public function activateUser(int $id): Result
{
    $findResult = $this->findUser($id);

    if (! $findResult->isSuccess()) {
        return $findResult; // propagate failure without re-throwing
    }

    $user = $findResult->value();

    if ($user->isAlreadyActive()) {
        return Result::failure('User is already active.');
    }

    $user->update(['active' => true]);

    return Result::ok($user);
}
```

### Result with Error Codes

When callers need to distinguish between failure reasons:

```php
final class Result
{
    private function __construct(
        private readonly bool $success,
        private readonly mixed $value,
        private readonly ?string $error,
        private readonly ?string $code,
    ) {}

    public static function ok(mixed $value): self
    {
        return new self(true, $value, null, null);
    }

    public static function failure(string $error, ?string $code = null): self
    {
        return new self(false, null, $error, $code);
    }

    public function isSuccess(): bool   { return $this->success; }
    public function value(): mixed      { return $this->value; }
    public function error(): ?string    { return $this->error; }
    public function code(): ?string     { return $this->code; }
}

// Usage
$result = $this->paymentService->charge($order);

return match ($result->code()) {
    'INSUFFICIENT_FUNDS' => response()->json(['error' => $result->error()], 402),
    'CARD_DECLINED'      => response()->json(['error' => $result->error()], 402),
    default              => response()->json(['error' => 'Payment failed'], 500),
};
```

---

## 4. Handling Third-Party / I/O Boundaries

The **only** place try-catch belongs in a Laravel app is at true I/O boundaries: HTTP clients, filesystem writes, queue publishing. Wrap the call, convert to a domain exception, and let the central handler take over.

```php
// app/Services/StripeService.php
public function charge(Order $order): PaymentResult
{
    try {
        $response = Http::stripe()->post('/charges', [
            'amount'   => $order->total_cents,
            'currency' => $order->currency,
        ]);

        if ($response->failed()) {
            return PaymentResult::declined($response->json('error.message'));
        }

        return PaymentResult::ok($response->json('id'));

    } catch (\Illuminate\Http\Client\ConnectionException $e) {
        // Convert to domain exception — controller never sees the raw HTTP exception
        throw new ExternalServiceException('Stripe', 'connection timeout', $e);
    }
}
```

Rules:
- Catch only the **specific** exception the library throws, not `\Exception`.
- Either convert to a domain exception **or** return a Result — never both.
- Never log inside the service. Let the handler log.

---

## 5. Queue & Job Exception Handling

Jobs run outside the HTTP lifecycle. The central handler does not apply. Use job-specific hooks.

```php
class ProcessPaymentJob implements ShouldQueue
{
    public int $tries = 3;
    public int $backoff = 60;

    public function handle(PaymentService $service): void
    {
        // No try-catch — let Laravel retry on failure
        $service->charge($this->order);
    }

    public function failed(\Throwable $e): void
    {
        // Called after all retries exhausted
        Log::error('Payment job permanently failed', [
            'order_id' => $this->order->id,
            'error'    => $e->getMessage(),
        ]);

        $this->order->update(['status' => 'payment_failed']);

        // Notify ops, fire event, etc.
    }
}
```

For expected failures inside jobs (e.g. external API declining a charge), use Result objects inside the service and inspect the result in `handle()`. Only **unexpected** exceptions should surface as job failures.

---

## 6. Logging Strategy

Never log the same exception at multiple layers — it creates duplicate noise.

| Layer | Responsibility |
|---|---|
| Service | Throw or return Result. **Never log.** |
| Controller | No catch blocks. **Never log.** |
| Central handler | Log **unexpected** exceptions once. |
| `report()` callback | Route to Sentry/Bugsnag/etc. |

```php
// In bootstrap/app.php — log unexpected exceptions with full context
$exceptions->report(function (\Throwable $e) {
    if ($e instanceof AppException) {
        return false; // silence domain exceptions
    }

    Log::error($e->getMessage(), [
        'exception' => get_class($e),
        'file'      => $e->getFile(),
        'line'      => $e->getLine(),
        'trace'     => $e->getTraceAsString(),
    ]);
});
```

---

## 7. API vs Web Response Splitting

For apps that serve both web and API routes, guard every renderer with `$request->expectsJson()` so Blade error pages still work for browser traffic.

```php
$exceptions->render(function (NotFoundException $e, Request $request) {
    if ($request->expectsJson()) {
        return response()->json([
            'error'   => $e->errorCode,
            'message' => $e->getMessage(),
        ], 404);
    }

    // Fall through to resources/views/errors/404.blade.php
    return null;
});
```

Alternatively, use route prefix matching:

```php
$exceptions->render(function (NotFoundException $e, Request $request) {
    if ($request->is('api/*')) {
        return response()->json(['error' => 'NOT_FOUND', 'message' => $e->getMessage()], 404);
    }

    return redirect()->route('home')->with('error', $e->getMessage());
});
```

---

## 8. Testing Exception Behaviour

Test that exceptions render correctly and that valid input never throws.

```php
// Feature test — exception renders as JSON
it('returns 404 when order is not found', function () {
    $response = $this->getJson('/api/orders/99999');

    $response->assertStatus(404)
             ->assertJson([
                 'error'   => 'NOT_FOUND',
                 'message' => 'Order not found with id: 99999',
             ]);
});

// Feature test — validation blocks bad input before service is called
it('returns 422 when email is missing', function () {
    $response = $this->postJson('/api/users', ['name' => 'Solomon']);

    $response->assertStatus(422)
             ->assertJsonValidationErrors(['email']);
});

// Unit test — Result propagates correctly
it('returns failure result when user does not exist', function () {
    $result = (new UserService)->findUser(99999);

    expect($result->isSuccess())->toBeFalse()
        ->and($result->error())->toContain('not found');
});
```

---

## 9. Common Mistakes Checklist

Before submitting a PR, verify:

- [ ] No `catch (\Exception $e)` or `catch (\Throwable $e)` outside `bootstrap/app.php` or I/O boundary wrappers
- [ ] No try-catch inside a controller action
- [ ] No silent catch blocks (`catch (\Exception $e) {}`)
- [ ] No `Log::error()` inside service or repository classes
- [ ] All Form Requests have a `rules()` method — no raw `$request->validate()` in controllers for complex input
- [ ] All custom exceptions extend `AppException`
- [ ] `ExternalServiceException` wraps the original `\Throwable` as `$previous` so stack trace is preserved
- [ ] Result objects used where failure is **expected**; exceptions used where failure is **exceptional**
