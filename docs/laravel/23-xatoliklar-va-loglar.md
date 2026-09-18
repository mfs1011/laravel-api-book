# 23 — Xatoliklar va loglar

[← Oldingi: Avtorizatsiya](22-avtorizatsiya.md) · [Mundarija](README.md) · [Keyingi: Artisan va konsol →](24-artisan-va-konsol.md)

---

## Xatoliklar qayerda sozlanadi

Laravel 11+ da `App\Exceptions\Handler` klassi yo'q. Hammasi `bootstrap/app.php` da:

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->shouldRenderJsonWhen(
        fn (Request $request) => $request->is('api/*') || $request->expectsJson(),
    );
})
```

Bu loyihada shu qator allaqachon bor — ya'ni `/api/*` yo'llaridagi xatolar HTML emas, **JSON** bo'lib qaytadi. API uchun bu aynan kerak.

---

## Xatolikni qayta ishlashning ikki tomoni

| Tushuncha | Ma'nosi |
| --- | --- |
| **report** | Xatoni qayerga yozish (log, Sentry, Flare) |
| **render** | Foydalanuvchiga qanday javob qaytarish |

```php
->withExceptions(function (Exceptions $exceptions): void {
    // 1) Qayta ishlash — loglash
    $exceptions->report(function (PaymentFailedException $e) {
        logger()->channel('payments')->error($e->getMessage());
    });

    // 2) Ba'zi xatolarni umuman loglamaslik
    $exceptions->dontReport([
        \App\Exceptions\ClientAbortedException::class,
    ]);

    // 3) Javobni o'zgartirish
    $exceptions->render(function (PaymentFailedException $e, Request $request) {
        return response()->json([
            'message' => 'To\'lov amalga oshmadi',
            'code' => $e->getCode(),
        ], 402);
    });

    // 4) Har bir logga qo'shimcha kontekst
    $exceptions->context(fn () => [
        'user_id' => auth()->id(),
        'url' => request()->fullUrl(),
    ]);
})
```

---

## Laravel avtomatik qaytaradigan javoblar

| Istisno | Status | Qachon |
| --- | --- | --- |
| `ValidationException` | 422 | Validatsiya o'tmadi |
| `AuthenticationException` | 401 | `auth` middleware, foydalanuvchi yo'q |
| `AuthorizationException` / `AccessDeniedHttpException` | 403 | Policy/Gate rad etdi |
| `ModelNotFoundException` | 404 | `findOrFail`, route model binding |
| `NotFoundHttpException` | 404 | Route topilmadi |
| `MethodNotAllowedHttpException` | 405 | Noto'g'ri HTTP metod |
| `ThrottleRequestsException` | 429 | Rate limit |
| Boshqa har qanday `Throwable` | 500 | Kutilmagan xato |

JSON javob formati:

```json
{ "message": "No query results for model [App\\Models\\Post] 42." }
```

Productionda (`APP_DEBUG=false`) 500 xatosi faqat `{"message": "Server Error"}` qaytaradi — ichki tafsilotlar sizib chiqmaydi.

---

## O'z istisnolaringiz

```shell
php artisan make:exception InsufficientBalanceException
```

```php
namespace App\Exceptions;

use Exception;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class InsufficientBalanceException extends Exception
{
    public function __construct(public readonly int $required, public readonly int $available)
    {
        parent::__construct("Balans yetarli emas: {$required} kerak, {$available} bor.");
    }

    /** Javobni istisnoning o'zi qaytarishi mumkin */
    public function render(Request $request): JsonResponse
    {
        return response()->json([
            'message' => $this->getMessage(),
            'required' => $this->required,
            'available' => $this->available,
        ], 402);
    }

    /** Loglashni ham o'zi boshqarishi mumkin */
    public function report(): void
    {
        logger()->channel('payments')->warning($this->getMessage());
    }
}
```

```php
throw new InsufficientBalanceException(required: 50_000, available: 12_000);
```

**Nega bunday qilgan ma'qul?** Chunki "qanday javob qaytarish" mantiqi istisno bilan bir joyda turadi — kontrollerda `try/catch` yozish shart emas.

Tez yordamchilar:

```php
abort(404, 'Topilmadi');
abort_if($post->isLocked(), 409, 'Post yopilgan');
abort_unless($user->is_admin, 403);
throw_if($balance < $sum, InsufficientBalanceException::class, $sum, $balance);
report($e);                  // loglash, lekin davom etish
rescue(fn () => risky(), fallback: null, report: false);
```

---

## API uchun bir xil xato formati

Barcha xatolar bir xil ko'rinishda bo'lishi frontend uchun qulay:

```php
->withExceptions(function (Exceptions $exceptions): void {
    $exceptions->render(function (Throwable $e, Request $request) {
        if (! $request->is('api/*')) {
            return null;                    // web uchun standart xatti-harakat
        }

        $status = match (true) {
            $e instanceof ValidationException => 422,
            $e instanceof AuthenticationException => 401,
            $e instanceof AuthorizationException => 403,
            $e instanceof ModelNotFoundException, $e instanceof NotFoundHttpException => 404,
            $e instanceof HttpExceptionInterface => $e->getStatusCode(),
            default => 500,
        };

        return response()->json([
            'message' => $status === 500 && ! config('app.debug')
                ? 'Server xatosi'
                : $e->getMessage(),
            'errors' => $e instanceof ValidationException ? $e->errors() : null,
        ], $status);
    });
})
```

> **Maslahat:** standart format allaqachon yaxshi (`message` + `errors`). Uni faqat haqiqiy ehtiyoj bo'lganda o'zgartiring — mijozlar standart formatni kutadi.

---

## Loglash

```php
use Illuminate\Support\Facades\Log;

Log::debug('Batafsil ma\'lumot');
Log::info('Foydalanuvchi kirdi', ['user_id' => $user->id]);
Log::warning('Sekin so\'rov', ['ms' => 1200]);
Log::error('To\'lov xatosi', ['order' => $order->id]);
Log::critical('Baza ishlamayapti');

logger('tez yozuv');
logger()->info('...');
```

Darajalar (RFC 5424): `debug` < `info` < `notice` < `warning` < `error` < `critical` < `alert` < `emergency`. `.env` dagi `LOG_LEVEL=debug` shu darajadan yuqorilarini yozadi.

### Kanallar — `config/logging.php`

```php
'channels' => [
    'stack' => ['driver' => 'stack', 'channels' => ['single'], 'ignore_exceptions' => false],
    'single' => ['driver' => 'single', 'path' => storage_path('logs/laravel.log'), 'level' => 'debug'],
    'daily' => ['driver' => 'daily', 'days' => 14],
    'slack' => ['driver' => 'slack', 'url' => env('LOG_SLACK_WEBHOOK_URL'), 'level' => 'critical'],
    'stderr' => ['driver' => 'monolog', 'handler' => StreamHandler::class],
],
```

```php
Log::channel('slack')->critical('Server yiqildi');
Log::stack(['single', 'slack'])->error('Muhim xato');
```

Yangi kanal qo'shish (masalan to'lovlar uchun alohida fayl):

```php
'payments' => [
    'driver' => 'daily',
    'path' => storage_path('logs/payments.log'),
    'level' => 'debug',
    'days' => 30,
],
```

> **Productionda qaysi kanal?** Docker/Laravel Cloud kabi muhitlarda `stderr` — loglar konteyner chiqishiga yoziladi va tashqi tizim ularni yig'adi. An'anaviy serverda `daily` yaxshi (fayl cheksiz o'smaydi).

### Kontekst

```php
use Illuminate\Support\Facades\Context;

Context::add('trace_id', (string) Str::uuid());
Context::add('user_id', auth()->id());

Log::info('Buyurtma yaratildi');
// log yozuvida trace_id va user_id avtomatik qo'shiladi
```

Kontekst navbatdagi job'larga ham ko'chadi — ya'ni bitta so'rovni HTTP va fon ishlari bo'ylab kuzatish mumkin.

---

## Loglarni o'qish

```shell
php artisan pail                      # jonli oqim
php artisan pail --filter=payment
php artisan pail --level=error
tail -f storage/logs/laravel.log
```

---

## Debug vositalari

```php
dd($var);                 // chiqarib to'xtatish
dump($var);
ray($var);                // Ray ilovasi bo'lsa
report($exception);
```

Foydali paketlar (dev-only): `barryvdh/laravel-debugbar`, `laravel/telescope`.

> **Diqqat:** `dd()`/`dump()` ni koddan olib tashlashni unutmang. `laravel/pint` va code review buni ushlashga yordam beradi.

---

## Amaliyot

1. `routes/api.php` da ataylab istisno tashlang va `Accept: application/json` bilan chaqiring — JSON xato qaytishini tekshiring.
2. `APP_DEBUG` ni `false` qilib, xato xabari yashirilganini ko'ring.
3. `InsufficientBalanceException` yozing, `render()` metodini qo'shing va 402 javob qaytarilishini tekshiring.
4. `config/logging.php` ga `payments` kanalini qo'shing va unga yozing.
5. `Context::add('trace_id', ...)` ni middleware'da qo'shing va ikki xil log yozuvida bir xil `trace_id` chiqishini ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/errors>
- <https://laravel.com/docs/13.x/logging>
- <https://laravel.com/docs/13.x/context>

---

[← Oldingi: Avtorizatsiya](22-avtorizatsiya.md) · [Mundarija](README.md) · [Keyingi: Artisan va konsol →](24-artisan-va-konsol.md)
