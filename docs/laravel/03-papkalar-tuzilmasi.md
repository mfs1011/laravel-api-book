# 03 — Papkalar tuzilmasi

[← Oldingi: O'rnatish](02-ornatish-va-ishga-tushirish.md) · [Mundarija](README.md) · [Keyingi: So'rovning hayot sikli →](04-sorov-hayot-sikli.md)

---

## Ildiz papkalar

```
app/          → sizning ilova kodingiz (modellar, kontrollerlar, xizmatlar)
bootstrap/    → ilovani yig'ish (app.php) va kesh fayllari
config/       → konfiguratsiya fayllari
database/     → migratsiyalar, factory'lar, seeder'lar, sqlite fayli
public/       → veb-server root'i: index.php, rasm/css/js
resources/    → Blade shablonlar, css/js manbalari, til fayllari
routes/       → route ta'riflari (web.php, console.php, api.php)
storage/      → loglar, kesh, yuklangan fayllar, kompilyatsiya qilingan Blade
tests/        → testlar (Pest)
vendor/       → Composer paketlari (tegmaysiz)
artisan       → CLI kirish nuqtasi
```

> **Symfony bilan solishtirish:** `app/` ≈ `src/`, `resources/views/` ≈ `templates/`, `storage/` ≈ `var/`, `public/` ≈ `public/`. Symfony'da `config/` YAML bo'lsa, Laravel'da `config/` — massiv qaytaruvchi PHP fayllar.

---

## `app/` ichida nima bor

Yangi loyihada `app/` juda kichkina:

```
app/
├── Http/
│   └── Controllers/
├── Models/
│   └── User.php
└── Providers/
    └── AppServiceProvider.php
```

Qolgan papkalar **kerak bo'lganda avtomatik paydo bo'ladi** — siz ularni qo'lda yaratmaysiz, tegishli `make:` buyrug'i yaratadi:

| Papka | Qaysi buyruq yaratadi | Nima uchun |
| --- | --- | --- |
| `app/Http/Middleware` | `make:middleware` | So'rov filtrlari |
| `app/Http/Requests` | `make:request` | Form Request (validatsiya) |
| `app/Http/Resources` | `make:resource` | JSON transformatsiya |
| `app/Console/Commands` | `make:command` | Artisan buyruqlari |
| `app/Events`, `app/Listeners` | `make:event`, `make:listener` | Hodisalar |
| `app/Jobs` | `make:job` | Navbat vazifalari |
| `app/Mail`, `app/Notifications` | `make:mail`, `make:notification` | Xabarlar |
| `app/Policies` | `make:policy` | Avtorizatsiya qoidalari |
| `app/Rules` | `make:rule` | Maxsus validatsiya qoidalari |
| `app/Observers` | `make:observer` | Model hodisalari |
| `app/Exceptions` | `make:exception` | Maxsus istisnolar |

**Nega bo'sh papkalar oldindan yaratilmaydi?** Chunki Laravel 11 dan beri "slim skeleton" siyosati bor: siz ishlatmagan narsa ko'rinmasin. Kam fayl — kam chalg'ish.

`app/` dagi hamma narsa `App\` namespace'iga PSR-4 orqali bog'langan (`composer.json` → `autoload.psr-4`).

---

## `bootstrap/app.php` — ilovaning markazi

Bu **eng muhim fayl**. Laravel 11+ da `Kernel.php` fayllari yo'q qilindi va hamma narsa shu yerga yig'ildi. Loyihadagi hozirgi holat:

```php
<?php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Exceptions;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web: __DIR__.'/../routes/web.php',
        commands: __DIR__.'/../routes/console.php',
        health: '/up',
    )
    ->withMiddleware(function (Middleware $middleware): void {
        //
    })
    ->withExceptions(function (Exceptions $exceptions): void {
        $exceptions->shouldRenderJsonWhen(
            fn (Request $request) => $request->is('api/*') || $request->expectsJson(),
        );
    })->create();
```

Bu yerda nima sozlanadi:

| Metod | Vazifasi | Batafsil |
| --- | --- | --- |
| `withRouting()` | Qaysi route fayllari yuklanadi, health-check manzili | [09-bob](09-marshrutlash.md) |
| `withMiddleware()` | Global middleware, guruhlar, aliaslar, tartib | [10-bob](10-middleware.md) |
| `withExceptions()` | Xatoliklarni qanday ko'rsatish/xabar berish | [23-bob](23-xatoliklar-va-loglar.md) |
| `withCommands()` | Qo'shimcha Artisan buyruqlari papkalari | [24-bob](24-artisan-va-konsol.md) |
| `withEvents()` | Listener'lar qidiriladigan papkalar | [26-bob](26-hodisalar-va-observerlar.md) |
| `withSchedule()` | Rejalashtirilgan vazifalar | [24-bob](24-artisan-va-konsol.md) |

`health: '/up'` — tayyor endpoint. `GET /up` ilova tirik bo'lsa 200 qaytaradi. Load balancer va Docker healthcheck uchun.

> **Symfony bilan solishtirish:** `bootstrap/app.php` ≈ `src/Kernel.php` + `config/bundles.php` + `config/packages/framework.yaml` ning birlashmasi.

---

## `bootstrap/providers.php`

```php
return [
    App\Providers\AppServiceProvider::class,
];
```

Bu — ilovaning **o'z** service provider'lari ro'yxati (Symfony'dagi `config/bundles.php` ga o'xshaydi). Freymvork va paketlarning provider'lari bu yerda emas: paketlar **auto-discovery** orqali o'zini ro'yxatdan o'tkazadi. Batafsil [06-bob](06-service-provider.md).

`bootstrap/cache/` — bu yerda `config.php`, `routes-*.php`, `packages.php` keshlari yotadi. Ularni qo'lda tahrirlamang.

---

## `routes/`

| Fayl | Nima | Prefiks | Middleware guruhi |
| --- | --- | --- | --- |
| `web.php` | Brauzer uchun sahifalar | yo'q | `web` (sessiya, cookie, CSRF) |
| `api.php` | Stateless API | `/api` | `api` |
| `console.php` | Closure-buyruqlar va scheduler | — | — |
| `channels.php` | Broadcast kanallari | — | — |

`api.php` **yangi loyihada yo'q**. Uni yaratish uchun:

```shell
php artisan install:api
```

Bu buyruq Sanctum'ni o'rnatadi, `routes/api.php` yaratadi va `bootstrap/app.php` ga `api:` qatorini qo'shadi. Batafsil [09](09-marshrutlash.md) va [21-bob](21-autentifikatsiya.md).

---

## `config/`

Har bir fayl — oddiy PHP massivi:

```php
// config/app.php (qisqartirilgan)
return [
    'name' => env('APP_NAME', 'Laravel'),
    'env' => env('APP_ENV', 'production'),
    'debug' => (bool) env('APP_DEBUG', false),
    'timezone' => 'UTC',
];
```

Kodda o'qish — **nuqtali sintaksis** bilan:

```php
config('app.name');
config('database.default');
config(['app.timezone' => 'Asia/Tashkent']); // faqat joriy so'rov uchun
```

Bu loyihada `config/` da 10 ta fayl bor (`app`, `auth`, `cache`, `database`, `filesystems`, `logging`, `mail`, `queue`, `services`, `session`). Laravel 11+ da **ko'p config fayllar skeletonda yo'q** — chunki standart qiymatlar freymvork ichida. Kerak bo'lsa nusxasini olasiz:

```shell
php artisan config:publish          # ro'yxatdan tanlash
php artisan config:publish cors     # aniq bittasini
```

---

## `database/`, `storage/`, `public/`

```
database/
├── factories/     → test ma'lumot generatorlari (18-bob)
├── migrations/    → jadval sxemalari (15-bob)
├── seeders/       → boshlang'ich ma'lumotlar (18-bob)
└── database.sqlite

storage/
├── app/           → ilova yuklagan fayllar (private/public)
├── framework/     → sessiya, view kesh, cache fayllari
└── logs/          → laravel.log

public/
├── index.php      → YAGONA kirish nuqtasi
├── build/         → Vite yiqqan css/js
└── storage        → `php artisan storage:link` yaratadigan symlink
```

`storage/` yozish huquqiga ega bo'lishi kerak (serverda `chmod -R ug+rwx storage bootstrap/cache`).

Yuklangan faylni internetdan ko'rsatish uchun:

```shell
php artisan storage:link
```

Bu `public/storage` → `storage/app/public` symlink'ini yaratadi. **Nega?** Chunki `storage/` public emas; faqat siz ochiq deb belgilagan qism ko'rinishi kerak.

---

## Amaliyot

1. `php artisan make:controller PostController` ni ishlating va `app/Http/Controllers/` ga qarang.
2. `php artisan make:middleware CheckToken` ni ishlating — `app/Http/Middleware/` papkasi o'zi paydo bo'lganini ko'ring.
3. `bootstrap/app.php` ni oching va `health: '/up'` ni o'chirib ko'ring, keyin `/up` manziliga kiring (404 bo'ladi), so'ng qaytaring.
4. `php artisan config:show app` bilan `config/app.php` ning to'liq hisoblangan qiymatlarini ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/structure>
- <https://laravel.com/docs/13.x/configuration>

---

[← Oldingi: O'rnatish](02-ornatish-va-ishga-tushirish.md) · [Mundarija](README.md) · [Keyingi: So'rovning hayot sikli →](04-sorov-hayot-sikli.md)
