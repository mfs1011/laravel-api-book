# 06 — Service Provider

[← Oldingi: Service Container](05-service-container.md) · [Mundarija](README.md) · [Keyingi: Fasadlar va helperlar →](07-fasadlar-va-helperlar.md)

---

## Service Provider nima

Provider — bu **ilovani yig'ish joyi**. Har bir provider ikkita ish qiladi:

1. `register()` — konteynerga xizmatlarni bog'laydi.
2. `boot()` — ilova tayyor bo'lgach bajariladigan sozlamalar (route model binding, validatsiya qoidalari, view composer'lar, policy'lar, rate limiter'lar...).

Laravel'ning o'zi ham shunday qurilgan: `DatabaseServiceProvider`, `QueueServiceProvider`, `MailServiceProvider` va boshqalar. Ya'ni freymvork xizmatlari ham, sizniki ham bir xil mexanizm bilan ulanadi.

> **Symfony bilan solishtirish:** Provider ≈ Bundle + `Extension` klassi + `boot()`. Symfony'da bundle konfiguratsiyani YAML orqali yuklaydi; Laravel'da provider PHP kodida bog'laydi.

---

## `register()` va `boot()` — farqi va nega bu muhim

Laravel avval **barcha** provider'larning `register()` metodini chaqiradi, keyin **barcha** `boot()` metodlarini.

```php
namespace App\Providers;

use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function register(): void
    {
        // FAQAT bog'lash. Boshqa xizmatlarga murojaat qilmang!
        $this->app->singleton(PaymentGateway::class, fn () => new StripeGateway(
            config('services.stripe.secret')
        ));
    }

    public function boot(): void
    {
        // Bu yerda hamma narsa tayyor: config, DB, route'lar, boshqa provider'lar
        Model::shouldBeStrict(! app()->isProduction());
    }
}
```

**Nega `register()` da boshqa xizmatdan foydalanmaslik kerak?** Chunki o'sha paytda boshqa provider'lar hali `register()` qilinmagan bo'lishi mumkin — ular hali konteynerda yo'q. `boot()` chaqirilganda esa hamma bog'lanishlar allaqachon mavjud. Bu — "provider tartibi"dan kelib chiqadigan klassik xato manbai.

---

## Provider yaratish va ro'yxatga qo'shish

```shell
php artisan make:provider ReportServiceProvider
```

Laravel 13 da bu buyruq provider'ni **avtomatik** `bootstrap/providers.php` ga qo'shadi:

```php
return [
    App\Providers\AppServiceProvider::class,
    App\Providers\ReportServiceProvider::class,
];
```

---

## `boot()` ichida odatda nima yoziladi

Bu ro'yxat — keyingi boblarning xaritasi ham:

```php
public function boot(): void
{
    // 1) Eloquent qat'iy rejimi (N+1 va typo'larni erta ushlaydi) — 17-bob
    Model::shouldBeStrict(! $this->app->isProduction());

    // 2) Rate limiter — 09-bob
    RateLimiter::for('api', fn (Request $request) =>
        Limit::perMinute(60)->by($request->user()?->id ?: $request->ip())
    );

    // 3) Route model binding uchun maxsus qoida — 09-bob
    Route::bind('slug', fn ($value) => Post::where('slug', $value)->firstOrFail());

    // 4) Gate — 22-bob
    Gate::define('admin-panel', fn (User $user) => $user->is_admin);

    // 5) Maxsus validatsiya qoidasi — 13-bob
    Validator::extend('uzbek_phone', fn ($attr, $value) => (bool) preg_match('/^\+998\d{9}$/', $value));

    // 6) Blade direktivasi — 30-bob
    Blade::directive('sum', fn ($expr) => "<?php echo number_format($expr, 0, '.', ' ').' so\'m'; ?>");
}
```

---

## Paketlarning provider'lari: auto-discovery

Symfony'da yangi bundle o'rnatganda uni `config/bundles.php` ga qo'shish kerak (Flex buni avtomatlashtiradi). Laravel'da esa paket o'z `composer.json` ida shunday yozadi:

```json
{
  "extra": {
    "laravel": {
      "providers": ["Laravel\\Sanctum\\SanctumServiceProvider"],
      "aliases": {"Sanctum": "Laravel\\Sanctum\\Sanctum"}
    }
  }
}
```

Composer o'rnatganda `php artisan package:discover` ishlaydi va provider `bootstrap/cache/packages.php` ga yoziladi. Shuning uchun `bootstrap/providers.php` da faqat **sizning** provider'laringiz turadi.

Biror paketning auto-discovery'sini o'chirish kerak bo'lsa, loyiha `composer.json` ida:

```json
"extra": { "laravel": { "dont-discover": ["barryvdh/laravel-debugbar"] } }
```

---

## Kechiktirilgan (deferred) provider'lar

Agar provider faqat konteynerga bog'lasa va uning xizmati har so'rovda kerak bo'lmasa, uni **kechiktirish** mumkin — shunda u faqat xizmat so'ralganda yuklanadi:

```php
use Illuminate\Contracts\Support\DeferrableProvider;

class ReportServiceProvider extends ServiceProvider implements DeferrableProvider
{
    public function register(): void
    {
        $this->app->singleton(ReportGenerator::class, fn () => new ReportGenerator);
    }

    /** @return array<int, string> */
    public function provides(): array
    {
        return [ReportGenerator::class];
    }
}
```

**Nega?** Har bir so'rovda kerak bo'lmagan ob'ektni yaratish — bekorga sarflangan vaqt. Deferred provider `boot()` ga ega bo'lmasligi kerak (chunki u yuklanmasligi mumkin).

---

## Paket resurslari (o'z paketingizni yozganda)

```php
public function boot(): void
{
    $this->loadRoutesFrom(__DIR__.'/../routes/web.php');
    $this->loadMigrationsFrom(__DIR__.'/../database/migrations');
    $this->loadViewsFrom(__DIR__.'/../resources/views', 'billing');
    $this->loadTranslationsFrom(__DIR__.'/../lang', 'billing');

    $this->publishes([
        __DIR__.'/../config/billing.php' => config_path('billing.php'),
    ], 'billing-config');
}

public function register(): void
{
    $this->mergeConfigFrom(__DIR__.'/../config/billing.php', 'billing');
}
```

Foydalanuvchi keyin `php artisan vendor:publish --tag=billing-config` bilan nusxa oladi.

---

## Provider'larni ko'rish

```shell
php artisan about --only=drivers
php artisan optimize        # config + route + event + provider keshlari
php artisan optimize:clear
```

---

## Amaliyot

1. `php artisan make:provider ReportServiceProvider` — `bootstrap/providers.php` ga avtomatik qo'shilganini tekshiring.
2. `register()` ichida `logger()->info('register')`, `boot()` ichida `logger()->info('boot')` yozing. Sahifani oching va `storage/logs/laravel.log` da tartibni ko'ring.
3. `register()` ichidan `config('app.name')` ni o'qib ko'ring — ishlaydi, chunki config bootstrap bosqichida yuklanadi. Keyin `register()` ichidan `Route::getRoutes()` ni chaqiring va nega bo'sh ekanini o'ylab ko'ring.
4. Provider'ni `DeferrableProvider` qiling va `provides()` ni to'ldiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/providers>
- <https://laravel.com/docs/13.x/packages>

---

[← Oldingi: Service Container](05-service-container.md) · [Mundarija](README.md) · [Keyingi: Fasadlar va helperlar →](07-fasadlar-va-helperlar.md)
