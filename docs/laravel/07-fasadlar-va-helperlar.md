# 07 — Fasadlar va helperlar

[← Oldingi: Service Provider](06-service-provider.md) · [Mundarija](README.md) · [Keyingi: Konfiguratsiya va muhit →](08-konfiguratsiya-va-muhit.md)

---

## Muammo: `Cache::get()` statik metodga o'xshaydi, lekin statik emas

Laravel kodida doim shunday yoziladi:

```php
use Illuminate\Support\Facades\Cache;

Cache::put('key', 'value', 60);
$value = Cache::get('key');
```

Bu **statik chaqiruvga o'xshaydi**, lekin aslida shunday bo'ladi:

1. `Cache` fasadi `__callStatic()` ni ushlaydi.
2. Fasad konteynerdan `cache` xizmatini oladi.
3. Metodni **o'sha ob'ektda** chaqiradi.

Ya'ni fasad — konteynerga qisqa yo'l. Isbot: fasad klassi juda kichkina:

```php
class Cache extends Facade
{
    protected static function getFacadeAccessor(): string
    {
        return 'cache';   // konteynerdagi kalit
    }
}
```

> **Nega umuman shunday qilingan?** Chunki `app(CacheRepository::class)->get('key')` har safar yozish uzun. Fasad — o'qish qulayligi uchun, lekin ichida **haqiqiy DI** turadi. Symfony'da bunga o'xshash narsa yo'q — u yerda siz doim konstruktor injection yozasiz.

---

## Fasad va helper — bir xil narsa

Ko'p fasadlarning qisqa helper varianti bor:

```php
Cache::get('key');        // === cache('key')
View::make('profile');    // === view('profile')
Response::json([...]);    // === response()->json([...])
Auth::user();             // === auth()->user()
Config::get('app.name');  // === config('app.name')
URL::route('posts.show'); // === route('posts.show')
```

Farqi yo'q — helper ham oxir-oqibat o'sha konteyner xizmatini chaqiradi.

---

## Eng ko'p ishlatiladigan fasadlar

| Fasad | Vazifasi | Bob |
| --- | --- | --- |
| `Route` | Route ta'riflash | [09](09-marshrutlash.md) |
| `DB` | Query builder, tranzaksiyalar | [14](14-malumotlar-bazasi.md) |
| `Schema` | Jadval sxemasi | [15](15-migratsiyalar.md) |
| `Auth` | Joriy foydalanuvchi | [21](21-autentifikatsiya.md) |
| `Gate` | Ruxsat tekshirish | [22](22-avtorizatsiya.md) |
| `Cache` | Kesh | [27](27-kesh-va-fayllar.md) |
| `Storage` | Fayllar | [27](27-kesh-va-fayllar.md) |
| `Http` | Tashqi HTTP so'rovlar | [27](27-kesh-va-fayllar.md) |
| `Mail` / `Notification` | Xabarlar | [28](28-mail-va-bildirishnoma.md) |
| `Queue` / `Bus` | Navbatlar | [25](25-navbatlar.md) |
| `Event` | Hodisalar | [26](26-hodisalar-va-observerlar.md) |
| `Log` | Loglash | [23](23-xatoliklar-va-loglar.md) |
| `Validator` | Validatsiya | [13](13-validatsiya.md) |

---

## Fasadlarni testda "soxtalashtirish" (fake)

Bu fasadlarning eng katta amaliy foydasi:

```php
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Queue;
use Illuminate\Support\Facades\Event;
use Illuminate\Support\Facades\Storage;

Mail::fake();
Queue::fake();
Event::fake();
Storage::fake('photos');

// ... kodni ishga tushiramiz ...

Mail::assertSent(OrderShipped::class);
Queue::assertPushed(ProcessPodcast::class);
```

`Mail::fake()` konteynerdagi haqiqiy mailer'ni soxtasi bilan almashtiradi. Ya'ni fasad DI ustida qurilgani uchun uni almashtirish mumkin. Batafsil [29-bob](29-testlash.md).

Mockery bilan ham ishlaydi:

```php
Cache::shouldReceive('get')->with('key')->andReturn('value');
```

---

## Qachon fasad, qachon injection

| Holat | Tavsiya |
| --- | --- |
| Route/closure, kichik kontroller | Fasad yoki helper — qulay |
| Murakkab servis klassi | **Konstruktor injection** — bog'liqliklar ko'rinib turadi |
| Kutubxona/paket kodi | Injection (fasadga bog'lanmaslik uchun) |
| Testni yozish qiyinlashsa | Injection |

Amaliy qoida: fasadlarni erkin ishlating, lekin bitta klassda 5-6 ta turli fasad paydo bo'lsa — bu klass juda ko'p ish qilyapti degani.

---

## Real-time fasadlar

Har qanday klassni fasad sifatida ishlatish mumkin — namespace oldiga `Facades\` qo'shib:

```php
use Facades\App\Services\Publisher;

Publisher::publish($podcast);
```

Bu `app(Publisher::class)->publish($podcast)` bilan bir xil, lekin testda `Publisher::shouldReceive('publish')` yozish mumkin bo'ladi.

---

## Eng foydali helperlar

```php
// Yo'llar
base_path('composer.json');
config_path('app.php');
database_path('database.sqlite');
storage_path('logs/laravel.log');
public_path('build');

// URL'lar
url('/posts');
route('posts.show', ['post' => 1]);
asset('build/app.js');

// Ma'lumot
data_get($array, 'user.profile.name', 'default');
data_set($array, 'user.active', true);
collect([1, 2, 3])->sum();
str('Salom Dunyo')->slug();          // salom-dunyo
Str::uuid();

// Sana (Carbon 3)
now();
now()->addDays(3);
now()->plus(days: 3);                // Laravel 13 hujjatlarida shu uslub
today()->startOfMonth();

// Boshqa
optional($user)->name;
throw_if($count === 0, NotFoundException::class);
retry(3, fn () => $api->call(), 100);
rescue(fn () => risky(), fallback: null);
tap($user, fn ($u) => $u->save());
abort(404);
abort_if(! $user->is_admin, 403);
report($exception);
```

`str()` va `collect()` — eng ko'p ishlatiladiganlari:

```php
str('  Salom, Dunyo!  ')->trim()->lower()->slug();   // salom-dunyo
collect($users)->filter(fn ($u) => $u->active)->pluck('email')->all();
```

---

## Amaliyot

1. `php artisan tinker` da quyidagilarni bajaring va natijani solishtiring:

```php
app('cache')->put('x', 1, 60);
cache('x');
Illuminate\Support\Facades\Cache::get('x');
```

2. `Illuminate\Support\Facades\Cache` klassini `vendor/` ichida oching va `getFacadeAccessor()` ni ko'ring — fasad qanchalik "yupqa" ekaniga ishonch hosil qiling.
3. `str('Toshkent shahri')->slug()` va `Str::of('Toshkent shahri')->slug()` bir xil natija berishini tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/facades>
- <https://laravel.com/docs/13.x/helpers>
- <https://laravel.com/docs/13.x/strings>

---

[← Oldingi: Service Provider](06-service-provider.md) · [Mundarija](README.md) · [Keyingi: Konfiguratsiya va muhit →](08-konfiguratsiya-va-muhit.md)
