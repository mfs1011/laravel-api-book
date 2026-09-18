# 08 — Konfiguratsiya va muhit

[← Oldingi: Fasadlar va helperlar](07-fasadlar-va-helperlar.md) · [Mundarija](README.md) · [Keyingi: Marshrutlash →](09-marshrutlash.md)

---

## Ikki qatlam: `.env` va `config/`

```
.env  →  config/*.php  →  config('...') bilan kodda o'qiladi
```

- **`.env`** — muhitga bog'liq, maxfiy qiymatlar. Git'ga tushmaydi.
- **`config/`** — ilova sozlamalari. Git'da bo'ladi, `.env` dan qiymat oladi.
- **Kod** — faqat `config()` orqali o'qiydi.

Misol:

```ini
# .env
MAIL_MAILER=log
```

```php
// config/mail.php
'default' => env('MAIL_MAILER', 'smtp'),
```

```php
// kod
config('mail.default');
```

---

## ⚠️ Eng muhim qoida: `env()` ni faqat `config/` ichida ishlating

```php
// ❌ NOTO'G'RI — kontroller, model, servis ichida
$key = env('STRIPE_SECRET');

// ✅ TO'G'RI
$key = config('services.stripe.secret');
```

**Nega?** Productionda `php artisan config:cache` ishlatiladi. U barcha config fayllarini bitta keshlangan PHP fayliga yig'adi va shundan keyin **`.env` umuman o'qilmaydi**. Natijada `env()` `null` qaytaradi va ilova "tushunarsiz" buziladi. `config()` esa keshdan o'qiydi va ishlayveradi.

---

## `.env` sintaksisi

```ini
APP_NAME="Mening Ilovam"        # bo'shliq bo'lsa qo'shtirnoq
APP_DEBUG=true                  # true/false → bool
CACHE_PREFIX=                   # bo'sh qator → "" (bo'sh satr)
MAIL_FROM_NAME="${APP_NAME}"    # boshqa o'zgaruvchiga havola
# BU IZOH
```

Muhitni tekshirish:

```php
app()->environment();                 // 'local'
app()->environment('local');          // true/false
app()->isProduction();
app()->isLocal();
```

```shell
php artisan env
```

Muhitga qarab boshqa `.env` fayli: `APP_ENV=staging` bo'lsa Laravel avval `.env.staging` ni qidiradi.

---

## Konfiguratsiya bilan ishlash

```php
config('app.name');                      // o'qish
config('app.name', 'Default');           // qiymat yo'q bo'lsa
config(['app.timezone' => 'Asia/Tashkent']);  // faqat joriy so'rovda
Config::string('app.name');              // tipga ishonch bilan o'qish
Config::integer('app.max_items');
Config::boolean('app.debug');
Config::array('app.providers');
```

`Config::string()` kabi metodlar qiymat kutilgan tipda bo'lmasa istisno tashlaydi — bu typo'larni erta ushlaydi.

Terminalda ko'rish:

```shell
php artisan config:show app
php artisan config:show database.connections.sqlite
```

---

## Yangi konfiguratsiya fayli qo'shish

```shell
php artisan make:config billing
```

```php
// config/billing.php
return [
    'currency' => env('BILLING_CURRENCY', 'UZS'),
    'vat' => (float) env('BILLING_VAT', 12),
];
```

```php
config('billing.vat');
```

Uchinchi tomon xizmatlari uchun alohida fayl yaratmang — `config/services.php` bor:

```php
// config/services.php
'eskiz' => [
    'token' => env('ESKIZ_TOKEN'),
    'from' => env('ESKIZ_FROM', '4546'),
],
```

---

## Keshlash (production uchun majburiy)

```shell
php artisan config:cache     # config'ni bitta faylga yig'adi
php artisan config:clear     # keshni o'chiradi

php artisan optimize         # config + route + event + view keshlari
php artisan optimize:clear   # hammasini tozalaydi
```

> **Diqqat:** lokalda `config:cache` ishlatmang — `.env` o'zgarishlari ko'rinmay qoladi va vaqtingizni yeydi.

---

## Maxfiy qiymatlarni shifrlash

```shell
php artisan env:encrypt          # .env.encrypted yaratadi
php artisan env:decrypt
```

Shifrlangan faylni git'da saqlash mumkin, kalit esa CI/CD sirlarida turadi.

---

## Maintenance rejimi

```shell
php artisan down --secret="maxfiy-token"
php artisan down --render="errors::503" --retry=60
php artisan up
```

`--secret` bergan bo'lsangiz, `https://sayt.uz/maxfiy-token` ga kirgan odam saytni normal ko'radi — deploy paytida tekshirish uchun.

---

## Konfiguratsiya bilan bog'liq tipik xatolar

| Muammo | Sabab |
| --- | --- |
| `.env` o'zgardi, lekin ta'sir qilmadi | `config:cache` yoqilgan → `config:clear` |
| Productionda `env()` `null` qaytaryapti | `env()` config'dan tashqarida ishlatilgan |
| Sana noto'g'ri mintaqada | `config/app.php` → `timezone` |
| Til/format noto'g'ri | `APP_LOCALE`, `APP_FALLBACK_LOCALE` |

---

## Amaliyot

1. `config/billing.php` yarating, ichiga `'vat' => (float) env('BILLING_VAT', 12)` yozing, `.env` ga `BILLING_VAT=15` qo'shing va `php artisan tinker` da `config('billing.vat')` ni tekshiring.
2. `php artisan config:cache` qiling, `.env` da `BILLING_VAT` ni 20 ga o'zgartiring va yana o'qing — o'zgarmasligini ko'ring. Keyin `config:clear`.
3. `config/app.php` da `timezone` ni `Asia/Tashkent` ga o'zgartiring va `now()` natijasini solishtiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/configuration>
- <https://laravel.com/docs/13.x/deployment#optimization>

---

[← Oldingi: Fasadlar va helperlar](07-fasadlar-va-helperlar.md) · [Mundarija](README.md) · [Keyingi: Marshrutlash →](09-marshrutlash.md)
