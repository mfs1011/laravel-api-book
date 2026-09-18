# 10 — Middleware

[← Oldingi: Marshrutlash](09-marshrutlash.md) · [Mundarija](README.md) · [Keyingi: Kontrollerlar →](11-kontrollerlar.md)

---

## Middleware nima

Middleware — so'rov kontrollerga yetib borishidan oldin (va javob qaytishida) ishlaydigan qatlam. Autentifikatsiya, CSRF, til tanlash, loglash, CORS — hammasi middleware.

```php
public function handle(Request $request, Closure $next): Response
{
    // 1) so'rovdan OLDIN
    $response = $next($request);   // 2) keyingi qatlam
    // 3) javobdan KEYIN
    return $response;
}
```

> **Symfony bilan solishtirish:** Symfony'da bu `kernel.request` / `kernel.response` event listener'lari yoki HttpKernel dekoratorlari. Laravel'da esa "pipeline" — piyoz qatlamlari kabi ichkariga kirib, keyin tashqariga chiqadi.

---

## Middleware yaratish

```shell
php artisan make:middleware EnsureTokenIsValid
```

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class EnsureTokenIsValid
{
    public function handle(Request $request, Closure $next): Response
    {
        if ($request->header('X-Api-Token') !== config('services.internal.token')) {
            abort(403, 'Token noto\'g\'ri');
        }

        return $next($request);
    }
}
```

So'rovni to'xtatish uchun `$next()` ni chaqirmay javob qaytarsangiz bo'ldi:

```php
if (! $request->user()->is_active) {
    return redirect('/blocked');          // web uchun
    // return response()->json(['message' => 'Bloklangan'], 403);   // api uchun
}
```

---

## Ro'yxatdan o'tkazishning 4 usuli

Hammasi `bootstrap/app.php` → `withMiddleware()` ichida.

### 1) Global — har bir so'rovda

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->append(\App\Http\Middleware\LogRequests::class);   // oxiriga
    $middleware->prepend(\App\Http\Middleware\TrustProxies::class); // boshiga
})
```

### 2) Guruhga qo'shish

```php
$middleware->web(append: [EnsureUserIsSubscribed::class]);
$middleware->api(prepend: [EnsureTokenIsValid::class]);
```

Standart guruhlar (Laravel 13):

| `web` | `api` |
| --- | --- |
| `EncryptCookies` | `SubstituteBindings` |
| `AddQueuedCookiesToResponse` | |
| `StartSession` | |
| `ShareErrorsFromSession` | |
| `PreventRequestForgery` (CSRF) | |
| `SubstituteBindings` | |

> Laravel 13 da CSRF middleware'i `VerifyCsrfToken` emas, **`PreventRequestForgery`** deb ataladi. Eski darsliklarda eski nom uchraydi.

Guruh elementini almashtirish yoki olib tashlash:

```php
$middleware->web(replace: [StartSession::class => StartCustomSession::class]);
$middleware->web(remove: [StartSession::class]);
```

### 3) O'z guruhingizni yaratish

```php
$middleware->appendToGroup('internal', [
    EnsureTokenIsValid::class,
    LogInternalCall::class,
]);
```

```php
Route::middleware('internal')->group(/* ... */);
```

### 4) Alias (qisqa nom)

```php
$middleware->alias([
    'subscribed' => EnsureUserIsSubscribed::class,
]);
```

```php
Route::get('/premium', ...)->middleware('subscribed');
```

Laravel'ning tayyor aliaslari: `auth`, `auth:sanctum`, `guest`, `can`, `throttle`, `signed`, `verified`, `password.confirm`, `cache.headers`.

---

## Parametrli middleware

```php
public function handle(Request $request, Closure $next, string ...$roles): Response
{
    if (! in_array($request->user()->role, $roles, true)) {
        abort(403);
    }

    return $next($request);
}
```

```php
Route::put('/post/{post}', ...)->middleware('role:admin,editor');
```

Klass nomi bilan ham yozish mumkin (Laravel 11+):

```php
Route::put('/post/{post}', ...)->middleware(EnsureUserHasRole::using('admin', 'editor'));
```

---

## Route'dan middleware'ni olib tashlash

```php
Route::middleware([EnsureTokenIsValid::class])->group(function () {
    Route::get('/a', ...);
    Route::get('/b', ...)->withoutMiddleware([EnsureTokenIsValid::class]);
});
```

`withoutMiddleware` faqat **route** middleware'iga ta'sir qiladi, globalga emas.

---

## Tartib (priority) muhim bo'lganda

Middleware'lar odatda ro'yxat tartibida ishlaydi. Lekin ba'zilari **doim** boshqasidan oldin kelishi kerak (masalan til yoki URL default'ini o'rnatuvchi middleware `SubstituteBindings` dan oldin):

```php
$middleware->prependToPriorityList(
    before: \Illuminate\Routing\Middleware\SubstituteBindings::class,
    prepend: \App\Http\Middleware\SetDefaultLocaleForUrls::class,
);
```

---

## Terminable middleware — javobdan keyin ishlash

```php
class LogRequests
{
    public function handle(Request $request, Closure $next): Response
    {
        return $next($request);
    }

    public function terminate(Request $request, Response $response): void
    {
        Log::info('request', [
            'url' => $request->fullUrl(),
            'status' => $response->getStatusCode(),
        ]);
    }
}
```

`terminate()` javob **brauzerga yuborilgandan keyin** chaqiriladi (FastCGI da). Ya'ni foydalanuvchi kutmaydi. Agar `handle()` va `terminate()` da bir xil ob'ekt kerak bo'lsa, middleware'ni singleton qiling:

```php
// AppServiceProvider::register()
$this->app->singleton(LogRequests::class);
```

---

## Tez-tez kerak bo'ladigan tayyor middleware'lar

```php
Route::get('/hisobot', ...)->middleware('auth');                    // kirgan bo'lishi kerak
Route::get('/login', ...)->middleware('guest');                     // kirmagan bo'lishi kerak
Route::get('/panel', ...)->middleware('can:admin-panel');           // Gate/Policy
Route::get('/api/data', ...)->middleware('auth:sanctum');           // API token
Route::get('/unsubscribe', ...)->middleware('signed');              // imzolangan URL
Route::get('/sozlama', ...)->middleware('password.confirm');        // parolni qayta so'rash
Route::get('/terms', ...)->middleware('cache.headers:public;max_age=3600;etag');
```

---

## Amaliyot

1. `php artisan make:middleware ForceJsonResponse` yarating: u `Accept` sarlavhasini `application/json` ga o'rnatsin.

```php
public function handle(Request $request, Closure $next): Response
{
    $request->headers->set('Accept', 'application/json');

    return $next($request);
}
```

2. Uni `api` guruhiga qo'shing: `$middleware->api(prepend: [ForceJsonResponse::class]);`
3. Endi API'dagi xatolik HTML emas, JSON bo'lib qaytishini tekshiring (mavjud bo'lmagan manzilga so'rov yuboring).
4. `terminate()` metodi bilan so'rov davomiyligini loglaydigan middleware yozing va `php artisan pail` orqali ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/middleware>
- <https://laravel.com/docs/13.x/csrf>

---

[← Oldingi: Marshrutlash](09-marshrutlash.md) · [Mundarija](README.md) · [Keyingi: Kontrollerlar →](11-kontrollerlar.md)
