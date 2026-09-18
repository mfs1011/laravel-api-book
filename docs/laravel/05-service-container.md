# 05 — Service Container (DI konteyner)

[← Oldingi: So'rovning hayot sikli](04-sorov-hayot-sikli.md) · [Mundarija](README.md) · [Keyingi: Service Provider →](06-service-provider.md)

---

## Bir jumlada

Service container — bu ob'ektlarni **yaratib beruvchi** va ularning bog'liqliklarini avtomatik to'ldiruvchi mexanizm. Symfony'dagi konteyner bilan g'oyasi bir xil, farqi: Laravel'da konfiguratsiya YAML emas, **PHP kodi**, va ro'yxatdan o'tkazish (`services.yaml`) shart emas.

---

## Zero-configuration resolution (eng ko'p ishlatiladigan holat)

Agar klassning bog'liqliklari **konkret klasslar** bo'lsa, hech narsa sozlamaysiz:

```php
namespace App\Services;

class OrderService
{
    public function __construct(
        private PaymentGateway $gateway,   // konteyner o'zi yaratadi
    ) {}
}
```

Kontrollerda shunchaki tip ko'rsatasiz:

```php
class OrderController extends Controller
{
    public function __construct(private OrderService $orders) {}

    public function store(Request $request)   // Request ham konteynerdan keladi
    {
        // ...
    }
}
```

Laravel konstruktor argumentlarining tiplarini o'qiydi (reflection) va ularni rekursiv yaratadi.

> **Symfony bilan solishtirish:** bu — autowiring. Farqi: Symfony'da klass `services.yaml` da (odatda `App\:` resursi orqali) ro'yxatdan o'tgan bo'lishi kerak. Laravel'da esa **hech qanday ro'yxat yo'q** — konteyner talab bo'lganda reflection bilan yaratadi.

---

## Qachon qo'lda bog'lash kerak

Ikki holatda:

1. **Interfeysni** tip sifatida ko'rsatmoqchisiz — konteyner qaysi implementatsiyani berishni bilishi kerak.
2. Ob'ektni yaratish maxsus mantiq talab qiladi (masalan API kalitini uzatish).

Bog'lash `AppServiceProvider::register()` ichida yoziladi:

```php
use App\Contracts\SmsSender;
use App\Services\EskizSmsSender;

public function register(): void
{
    // 1) Interfeys → implementatsiya
    $this->app->bind(SmsSender::class, EskizSmsSender::class);

    // 2) Maxsus yaratish mantiqi
    $this->app->bind(SmsSender::class, function ($app) {
        return new EskizSmsSender(
            token: config('services.eskiz.token'),
            http: $app->make(\Illuminate\Http\Client\Factory::class),
        );
    });
}
```

Endi istalgan joyda faqat interfeysni so'raysiz:

```php
public function __construct(private SmsSender $sms) {}
```

**Nega interfeysga bog'lash yaxshi?** Testda `EskizSmsSender` o'rniga soxta (fake) implementatsiyani bog'lab qo'yasiz — kodingizning qolgan qismi umuman o'zgarmaydi.

---

## `bind` va `singleton` farqi

```php
$this->app->bind(Foo::class, fn () => new Foo);       // HAR safar yangi ob'ekt
$this->app->singleton(Foo::class, fn () => new Foo);  // bir marta yaratilib, qayta ishlatiladi
$this->app->instance(Foo::class, $allaqachonBor);     // tayyor ob'ektni ro'yxatga qo'yish
```

| Metod | Qachon |
| --- | --- |
| `bind` | Holati (state) bo'lgan, arzon ob'ektlar |
| `singleton` | Ulanish, mijoz, konfiguratsiya kabi qimmat yoki umumiy ob'ektlar |
| `scoped` | Bitta so'rov / bitta job davomida bitta nusxa (Octane kabi doimiy serverda muhim) |

```php
$this->app->scoped(RequestContext::class, fn () => new RequestContext);
```

Laravel 13 da buni atribut bilan ham belgilash mumkin:

```php
use Illuminate\Container\Attributes\Scoped;

#[Scoped]
class Transistor {}
```

> **Nega `singleton` ni Octane'da ehtiyot bo'lib ishlatish kerak?** Oddiy PHP-FPM da har so'rov yangi jarayon — singleton so'rov oxirida yo'qoladi. Octane'da jarayon uzoq yashaydi, shuning uchun singleton ichida so'rovga bog'liq ma'lumot saqlash ma'lumot "oqib ketishi"ga olib keladi. Shunday holatda `scoped` ishlating.

---

## Ob'ektni konteynerdan olish

```php
$service = app(OrderService::class);          // helper
$service = app()->make(OrderService::class);  // bir xil narsa
$service = resolve(OrderService::class);      // bir xil narsa
```

Parametr bilan:

```php
$service = app()->makeWith(Report::class, ['year' => 2026]);
```

> **Muhim qoida:** ilova kodida `app()` ni imkon qadar kam ishlating. To'g'ri yo'l — konstruktorda tip ko'rsatish (dependency injection). `app()` — bu "service locator" naqshi va u bog'liqliklarni yashiradi. Uni asosan provider'lar, helper funksiyalar va test kodida ishlating.

---

## Kontekstga bog'liq bog'lash (contextual binding)

Bitta interfeys, ikki xil implementatsiya — kim so'rayotganiga qarab:

```php
use App\Http\Controllers\ReportController;
use App\Http\Controllers\InvoiceController;
use Illuminate\Contracts\Filesystem\Filesystem;
use Illuminate\Support\Facades\Storage;

$this->app->when(ReportController::class)
    ->needs(Filesystem::class)
    ->give(fn () => Storage::disk('s3'));

$this->app->when(InvoiceController::class)
    ->needs(Filesystem::class)
    ->give(fn () => Storage::disk('local'));
```

Oddiy qiymatlarni bog'lash:

```php
$this->app->when(SmsSender::class)
    ->needs('$retries')
    ->give(3);
```

---

## Atributlar bilan bog'lash (Laravel 13)

Konstruktor argumentiga to'g'ridan-to'g'ri "qayerdan olish"ni yozish mumkin:

```php
use Illuminate\Container\Attributes\Config;
use Illuminate\Container\Attributes\DB;
use Illuminate\Container\Attributes\Storage;
use Illuminate\Container\Attributes\Auth;

class PhotoService
{
    public function __construct(
        #[Storage('s3')] protected Filesystem $disk,
        #[Config('app.timezone')] protected string $timezone,
    ) {}
}
```

Bu — `when()->needs()->give()` ning qisqa ko'rinishi va ko'p hollarda o'qish osonroq.

Interfeysni implementatsiyaga bog'lashni ham atribut bilan e'lon qilish mumkin — bunda provider'ga qator yozish shart emas:

```php
use Illuminate\Container\Attributes\Bind;
use Illuminate\Container\Attributes\Singleton;

#[Bind(EskizSmsSender::class)]
interface SmsSender {}

// faqat ma'lum muhitda boshqa implementatsiya:
#[Bind(FakeSmsSender::class, environments: ['local', 'testing'])]
interface SmsSender {}

#[Singleton]
class PaymentGateway {}
```

Mavjud atributlar: `#[Bind]`, `#[Singleton]`, `#[Scoped]`, `#[Give]`, `#[Config]`, `#[DB]`, `#[Storage]`, `#[Auth]`, `#[CurrentUser]`, `#[RouteParameter]`, `#[Log]`, `#[Cache]`, `#[Tag]`.

---

## Tagging (bir turdagi xizmatlar guruhi)

```php
$this->app->bind(SmsReport::class, fn () => new SmsReport);
$this->app->bind(EmailReport::class, fn () => new EmailReport);

$this->app->tag([SmsReport::class, EmailReport::class], 'reports');

// keyin:
$reports = $this->app->tagged('reports');   // iterable
```

> **Symfony bilan solishtirish:** bu `tags:` + "tagged iterator" ning aynan o'zi.

---

## Konteyner hodisalari

```php
$this->app->resolving(OrderService::class, function ($service, $app) {
    // har safar OrderService yaratilganda ishlaydi
});

$this->app->extend(SmsSender::class, function ($service, $app) {
    return new LoggingSmsSender($service);   // dekorator
});
```

`extend()` — Symfony'dagi decorator naqshining analogi.

---

## Amaliyot

1. `app/Contracts/Greeter.php` interfeysini va `app/Services/UzbekGreeter.php` implementatsiyasini yarating (`php artisan make:interface Contracts/Greeter`, `php artisan make:class Services/UzbekGreeter`).
2. `AppServiceProvider::register()` da ularni bog'lang.
3. `routes/web.php` da:

```php
Route::get('/salom', function (App\Contracts\Greeter $greeter) {
    return $greeter->greet('Maruf');
});
```

Route closure'ida ham tip ko'rsatish ishlashini ko'ring.

4. `bind` ni `singleton` ga o'zgartiring va ob'ekt `spl_object_id()` si ikki so'rovda bir xilmi-yo'qmi — tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/container>
- <https://laravel.com/docs/13.x/contracts> — freymvork interfeyslari

---

[← Oldingi: So'rovning hayot sikli](04-sorov-hayot-sikli.md) · [Mundarija](README.md) · [Keyingi: Service Provider →](06-service-provider.md)
