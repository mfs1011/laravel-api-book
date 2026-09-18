# 32 — Optimallashtirish va deploy

[← Oldingi: Amaliy loyiha](31-amaliy-loyiha-api.md) · [Mundarija](README.md) · [Keyingi: Keyingi qadamlar →](33-keyingi-qadamlar.md)

---

## Serverga talablar

- PHP 8.3+ (kengaytmalar bilan), Composer
- Veb-server root'i — **`public/`** papkasi
- `storage/` va `bootstrap/cache/` ga yozish huquqi
- Navbat uchun doimiy jarayon (supervisor/systemd) va bitta cron yozuvi

Nginx uchun asosiy qism:

```nginx
root /var/www/loyiha/public;
index index.php;

location / {
    try_files $uri $uri/ /index.php?$query_string;
}

location ~ \.php$ {
    fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
}
```

---

## Deploy ketma-ketligi

```shell
# 1) kodni olish
git pull origin main

# 2) bog'liqliklar (dev paketlarsiz)
composer install --no-dev --optimize-autoloader

# 3) frontend
npm ci && npm run build

# 4) migratsiyalar
php artisan migrate --force

# 5) keshlarni yangilash
php artisan optimize

# 6) worker'larni qayta ishga tushirish
php artisan queue:restart
```

> **`--force` nega kerak?** `migrate` productionda tasdiq so'raydi. Avtomatik skriptda tasdiq berib bo'lmaydi, shuning uchun `--force`.

Nol uzilishli (zero-downtime) deploy uchun Envoyer, Deployer, Laravel Cloud yoki GitHub Actions ishlatiladi: yangi versiya alohida papkaga yig'iladi, keyin symlink almashtiriladi.

Xizmat vaqtida:

```shell
php artisan down --secret="tekshiruv-uchun-token" --retry=60
# ... deploy ...
php artisan up
```

---

## `php artisan optimize` nima qiladi

| Buyruq | Ta'siri |
| --- | --- |
| `config:cache` | Barcha config bitta faylga — `.env` o'qilmaydi |
| `route:cache` | Route'lar oldindan kompilyatsiya qilinadi |
| `event:cache` | Listener'lar manifesti keshlanadi (skanerlash yo'q) |
| `view:cache` | Blade shablonlari oldindan kompilyatsiya qilinadi |

Tozalash: `php artisan optimize:clear`.

Eslatmalar:

- `route:cache` bilan route fayllarida **closure bo'lmasligi** kerak.
- `config:cache` dan keyin koddagi `env()` `null` qaytaradi ([08-bob](08-konfiguratsiya-va-muhit.md)).

---

## Productionda `.env`

```ini
APP_ENV=production
APP_DEBUG=false          # ALBATTA false
APP_URL=https://sayt.uz

LOG_CHANNEL=stack
LOG_LEVEL=warning        # debug loglar diskni to'ldirmasin

CACHE_STORE=redis
SESSION_DRIVER=redis
QUEUE_CONNECTION=redis

DB_CONNECTION=mysql
```

---

## Ishlash unumdorligini oshirish

### 1. Baza — eng ko'p muammo shu yerda

- **N+1 so'rovlarni yo'q qiling**: `with()`, `withCount()`, `Model::preventLazyLoading()` ([17-bob](17-eloquent-aloqalar.md)).
- **Indekslar**: `where`, `join`, `order by` da qatnashadigan ustunlarga.
- **`select()`**: kerakli ustunlarnigina oling.
- **`paginate()`**: hech qachon cheksiz ro'yxat qaytarmang; katta jadvallarda `cursorPaginate()`.
- **`chunk`/`lazy`**: ommaviy qayta ishlashda.

### 2. Kesh

```php
Cache::remember('stats', 300, fn () => /* og'ir so'rov */);
```

Redis tavsiya etiladi. HTTP darajasida: `ETag`, `Cache-Control` (`cache.headers` middleware).

### 3. Navbatlar

Sekin ishlarni (pochta, rasm, tashqi API) so'rov ichida bajarmang ([25-bob](25-navbatlar.md)).

### 4. PHP darajasida

```shell
composer install --no-dev --optimize-autoloader
```

`opcache` ni yoqing (`opcache.enable=1`, productionda `opcache.validate_timestamps=0`).

### 5. Octane (ixtiyoriy)

```shell
composer require laravel/octane
php artisan octane:install
php artisan octane:start
```

Ilova xotirada doimiy turadi — har so'rovda bootstrap qilinmaydi. Tezlik sezilarli oshadi, lekin kod "holatga toza" bo'lishi kerak: statik xossalarda so'rov ma'lumotini saqlamang, `singleton` o'rniga `scoped` ishlating ([05-bob](05-service-container.md)).

---

## Monitoring va kuzatuv

| Vosita | Nima beradi |
| --- | --- |
| **Telescope** (lokal/staging) | So'rovlar, SQL, joblar, xatolar tarixi |
| **Horizon** (Redis navbatlari) | Navbat paneli, metrikalar |
| **Pulse** | Ilova salomatligi: sekin so'rovlar, sekin joblar |
| **Nightwatch / Sentry / Flare** | Productionda xatoliklarni yig'ish |
| `/up` endpoint | Health check (`bootstrap/app.php` da yoqilgan) |

```php
// sekin so'rovlarni aniqlash — AppServiceProvider::boot()
DB::whenQueryingForLongerThan(500, fn ($connection) =>
    logger()->warning('Sekin so\'rov', ['connection' => $connection->getName()])
);
```

---

## Xavfsizlik ro'yxati (deploy oldidan)

- [ ] `APP_DEBUG=false`, `APP_ENV=production`
- [ ] `.env` va `storage/` internetdan ko'rinmaydi (root — `public/`)
- [ ] HTTPS yoqilgan, `APP_URL` https bilan
- [ ] Barcha yozuv endpointlarida validatsiya (`FormRequest`)
- [ ] Ruxsatlar Policy bilan tekshiriladi ([22-bob](22-avtorizatsiya.md))
- [ ] `login`/`register` endpointlarida `throttle`
- [ ] Mass assignment himoyasi (`$fillable` / `#[Fillable]`)
- [ ] SQL faqat parametrlar bilan (`whereRaw` da ham)
- [ ] Fayl yuklashda `mimes` va `max` tekshiruvi
- [ ] CORS'da aniq domenlar
- [ ] Bog'liqliklar yangilangan: `composer audit`, `npm audit`
- [ ] Zaxira nusxa (backup) sozlangan va **tiklash sinab ko'rilgan**

---

## Laravel Cloud

Laravel'ning rasmiy hosting platformasi: bazasi, Redis'i, navbat worker'lari va scheduler'i bilan birga keladi, deploy git push orqali bo'ladi.

```shell
composer global require laravel/cloud-cli
cloud deploy
```

Muqobillar: Forge (o'z serveringizni boshqarish), Vapor (AWS serverless), oddiy VPS + supervisor.

---

## Amaliyot

1. Lokalda deploy ketma-ketligini bajarib ko'ring: `composer install --no-dev`, `php artisan optimize`, keyin `optimize:clear`.
2. `config:cache` yoqilganda `env()` `null` qaytarishini o'z ko'zingiz bilan ko'ring.
3. `DB::whenQueryingForLongerThan()` ni qo'shing va ataylab sekin so'rov yozib, ogohlantirish chiqishini tekshiring.
4. Yuqoridagi xavfsizlik ro'yxatini o'z loyihangiz uchun to'ldiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/deployment>
- <https://laravel.com/docs/13.x/octane>
- <https://laravel.com/docs/13.x/telescope>
- <https://laravel.com/docs/13.x/pulse>

---

[← Oldingi: Amaliy loyiha](31-amaliy-loyiha-api.md) · [Mundarija](README.md) · [Keyingi: Keyingi qadamlar →](33-keyingi-qadamlar.md)
