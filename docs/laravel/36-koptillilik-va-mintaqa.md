# 36 — Ko'p tillilik va mintaqaviylik

[← Oldingi: Real-time](35-realtime-broadcasting.md) · [Mundarija](README.md) · [Keyingi: Ilg'or Eloquent →](37-ilgor-eloquent-va-unumdorlik.md)

---

## Ikki xil "tarjima"

Buni aralashtirib yubormaslik muhim:

| Nima | Qayerda saqlanadi | Misol |
| --- | --- | --- |
| **Interfeys matnlari** (statik) | `lang/` fayllari | "Saqlash", "Parol noto'g'ri" |
| **Kontent** (dinamik, foydalanuvchi kiritadi) | **Bazada** | Mahsulot nomi uz/ru/en da |

Laravel'ning lokalizatsiya tizimi — birinchisi uchun. Ikkinchisini o'zingiz loyihalashtirasiz (quyida).

---

## 1. Interfeys matnlari

### Til fayllari

```shell
php artisan lang:publish        # freymvork matnlarini lang/en ga chiqaradi
```

Ikki uslub bor:

**a) Qisqa kalitlar** — `lang/uz/messages.php`:

```php
return [
    'welcome' => 'Xush kelibsiz!',
    'posts' => [
        'created' => 'Post yaratildi.',
        'deleted' => ':title o\'chirildi.',
    ],
];
```

```php
__('messages.welcome');
__('messages.posts.deleted', ['title' => $post->title]);
trans('messages.welcome');
```

**b) Matnning o'zi kalit sifatida** — `lang/uz.json` (ko'p matnli loyihalar uchun qulayroq):

```json
{
    "Save": "Saqlash",
    "Welcome, :name": "Xush kelibsiz, :name"
}
```

```php
__('Save');
__('Welcome, :name', ['name' => $user->name]);
```

Blade'da:

```blade
{{ __('messages.welcome') }}
@lang('messages.welcome')
```

### Til tanlash

```php
app()->getLocale();                 // joriy til
app()->setLocale('uz');
app()->isLocale('uz');

App::setFallbackLocale('en');       // tarjima topilmasa
```

`.env`:

```ini
APP_LOCALE=uz
APP_FALLBACK_LOCALE=en
APP_FAKER_LOCALE=uz_UZ
```

Vaqtinchalik boshqa tilda bajarish:

```php
$text = Lang::get('messages.welcome', [], 'ru');

App::setLocale('ru');
// ...
App::setLocale('uz');
```

### API'da tilni aniqlash — middleware

```shell
php artisan make:middleware SetLocale
```

```php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SetLocale
{
    private const SUPPORTED = ['uz', 'ru', 'en'];

    public function handle(Request $request, Closure $next): Response
    {
        $locale = $request->header('X-Locale')
            ?? $request->user()?->locale
            ?? $request->getPreferredLanguage(self::SUPPORTED);

        if (in_array($locale, self::SUPPORTED, true)) {
            app()->setLocale($locale);
        }

        return $next($request);
    }
}
```

```php
// bootstrap/app.php
$middleware->api(prepend: [\App\Http\Middleware\SetLocale::class]);
```

> **Tartib:** tilni o'rnatuvchi middleware `SubstituteBindings` dan oldin ishlashi kerak bo'lsa, `prependToPriorityList()` ishlating ([10-bob](10-middleware.md)).

### Ko'plik shakllari

```php
// lang/uz/messages.php
'apples' => '{0} Olma yo\'q|[1,*] :count ta olma',
```

```php
trans_choice('messages.apples', $count, ['count' => $count]);
```

O'zbek tilida ko'plik shakli o'zgarmaydi, lekin rus tilida uch shakl bor (`1 яблоко | 2 яблока | 5 яблок`) — `trans_choice` shuning uchun kerak.

### Validatsiya xabarlari

`lang/uz/validation.php` da barcha qoidalar tarjima qilinadi:

```php
return [
    'required' => ':attribute maydonini to\'ldirish shart.',
    'email' => ':attribute to\'g\'ri elektron pochta bo\'lishi kerak.',
    'min' => [
        'string' => ':attribute kamida :min belgidan iborat bo\'lsin.',
    ],
    'attributes' => [
        'title' => 'sarlavha',
        'body' => 'matn',
    ],
];
```

Endi 422 javoblari foydalanuvchi tilida keladi ([13-bob](13-validatsiya.md)).

### Xat va bildirishnomalarda til

Navbatga qo'yilgan xat foydalanuvchi so'rovidan keyin bajariladi — shuning uchun til saqlanishi kerak:

```php
$user->notify((new InvoicePaid($invoice))->locale('ru'));
Notification::locale('ru')->send($users, new InvoicePaid($invoice));
```

Yoki modelga afzal tilni "o'rgating":

```php
use Illuminate\Contracts\Translation\HasLocalePreference;

class User extends Authenticatable implements HasLocalePreference
{
    public function preferredLocale(): string
    {
        return $this->locale ?? config('app.locale');
    }
}
```

Shundan keyin `$user->notify(...)` avtomatik to'g'ri tilda ketadi.

---

## 2. Kontentni tarjima qilish (baza)

Uchta keng tarqalgan yondashuv:

### a) JSON ustun (eng oddiy, kichik loyihalar uchun)

```php
// migratsiya
$table->json('title');       // {"uz": "Salom", "ru": "Привет"}
```

```php
protected function casts(): array
{
    return ['title' => 'array'];
}

protected function localizedTitle(): Attribute
{
    return Attribute::get(fn () => $this->title[app()->getLocale()] ?? $this->title[config('app.fallback_locale')] ?? '');
}
```

So'rov:

```php
Post::where('title->uz', 'like', "%{$q}%")->get();
```

➕ oddiy, migratsiya kam. ➖ bo'yicha indekslash va murakkab qidiruv qiyin.

### b) Alohida tarjima jadvali

```
posts            → id, slug, status
post_translations → id, post_id, locale, title, body   (unique: post_id + locale)
```

➕ to'liq indeks, har til uchun to'liq nazorat. ➖ ko'proq `join` va kod.

### c) Tayyor paket

`spatie/laravel-translatable` (JSON yondashuvi ustida qulay qatlam) yoki `astrotomic/laravel-translatable` (alohida jadval).

> **Tavsiya:** 2-3 til va oddiy maydonlar bo'lsa — JSON ustun. Katalog, SEO va to'liq matnli qidiruv kerak bo'lsa — alohida jadval.

API javobida tarjima:

```php
// PostResource
'title' => $this->title[app()->getLocale()] ?? $this->title['en'] ?? null,
```

---

## 3. Sana, vaqt va mintaqa

```php
// config/app.php
'timezone' => 'UTC',       // bazada HAR DOIM UTC saqlang
```

**Qoida:** bazada UTC, ko'rsatishda foydalanuvchi mintaqasi.

```php
$post->published_at->timezone('Asia/Tashkent')->format('d.m.Y H:i');
$post->published_at->setTimezone($user->timezone)->toDayDateTimeString();

// API javobida — ISO 8601, mintaqa bilan
'published_at' => $post->published_at?->toIso8601String(),   // 2026-09-18T09:30:00+00:00
```

**Nega UTC?** Yozgi vaqt, mintaqa o'zgarishi va turli davlatlardagi foydalanuvchilar — bularning barchasida UTC yagona ishonchli nuqta. Formatlashni faqat chegarada (javob yoki shablon) qiling.

Carbon va til:

```php
use Carbon\Carbon;

Carbon::setLocale('uz');        // KIRILL: "3 соат аввал"
Carbon::setLocale('uz_Latn');   // LOTIN:  "3 soat avval"

now()->subHours(3)->diffForHumans();   // "3 soat avval"
now()->translatedFormat('d F Y');      // "18 Sentabr 2026"
```

> **Shu muhitda tekshirilgan:** `uz` locale'i Carbon'da **kirill** alifbosini beradi. Lotin yozuvi kerak bo'lsa `uz_Latn` ni ishlating. `APP_LOCALE=uz` va Carbon locale'i alohida sozlanadi — `AppServiceProvider::boot()` da `Carbon::setLocale('uz_Latn')` deb qo'ying.

Sonlar va pul:

```php
use Illuminate\Support\Number;

Number::format(1234567.891, precision: 2, locale: 'uz');      // 1 234 567,89
Number::currency(1500000, in: 'UZS', locale: 'uz');
Number::fileSize(2048);                                       // 2 KB
Number::percentage(12.345, precision: 1);
Number::spell(15, locale: 'en');                              // fifteen
```

> **Diqqat:** `Number::spell()` ICU kutubxonasiga tayanadi va o'zbek tili uchun "spellout" qoidasi yo'q — `locale: 'uz'` bersangiz ham inglizcha qaytadi (tekshirilgan). Sonni so'z bilan yozish kerak bo'lsa, o'z helperingizni yozing.

> **Pulni `float` da saqlamang** ([15-bob](15-migratsiyalar.md)). Eng ishonchli yo'l — butun songa aylantirib (tiyinda) saqlash yoki `decimal` ustun.

---

## 4. Amaliy maslahatlar

1. Kodda **qattiq yozilgan matn qoldirmang** — hattoki bitta tilli loyihada ham `__()` ishlating; keyin til qo'shish osonlashadi.
2. Kalit nomlarini mazmun bo'yicha guruhlang: `messages.posts.created`, `errors.payment.declined`.
3. Tarjima yo'qligini erta bilish uchun test yozing:

```php
it('barcha uz tarjimalari en bilan mos', function () {
    $en = array_keys(Arr::dot(require lang_path('en/messages.php')));
    $uz = array_keys(Arr::dot(require lang_path('uz/messages.php')));

    expect(array_diff($en, $uz))->toBeEmpty();
});
```

4. Foydalanuvchining tilini profilda saqlang (`users.locale`) — mobil ilova va xatlar uchun kerak bo'ladi.
5. API mijoziga tilni `X-Locale` sarlavhasi orqali tanlash imkonini bering va javobda `Content-Language` qaytaring.

---

## Amaliyot

1. `php artisan lang:publish` qiling, `lang/uz/validation.php` yarating va bir nechta xabarni tarjima qiling.
2. `SetLocale` middleware'ini yozing va `X-Locale: ru` sarlavhasi bilan 422 javobi tili o'zgarishini tekshiring.
3. `lang/uz.json` yarating va `__('Save')` ni ishlating.
4. `Number::currency(1500000, in: 'UZS', locale: 'uz')` natijasini tinker'da ko'ring.
5. `published_at` ni `Asia/Tashkent` mintaqasida chiqaring, bazada esa UTC qolganini tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/localization>
- <https://laravel.com/docs/13.x/helpers#numbers>
- <https://carbon.nesbot.com/docs/>

---

[← Oldingi: Real-time](35-realtime-broadcasting.md) · [Mundarija](README.md) · [Keyingi: Ilg'or Eloquent →](37-ilgor-eloquent-va-unumdorlik.md)
