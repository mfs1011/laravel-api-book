# 02 — O'rnatish va ishga tushirish

[← Oldingi: Kirish](01-kirish.md) · [Mundarija](README.md) · [Keyingi: Papkalar tuzilmasi →](03-papkalar-tuzilmasi.md)

---

## Talablar

- **PHP 8.3+** (bu loyiha PHP 8.5 da ishlaydi)
- **Composer 2**
- **Node.js + npm** — faqat frontend (Vite) uchun
- Ma'lumotlar bazasi: SQLite (hech narsa o'rnatish shart emas), MySQL, PostgreSQL, MariaDB yoki SQL Server

PHP kengaytmalari (odatda allaqachon bor): `pdo`, `mbstring`, `openssl`, `tokenizer`, `xml`, `ctype`, `json`, `curl`, `fileinfo`.

Tekshirish:

```shell
php -v
composer -V
node -v
```

---

## Yangi loyiha yaratishning 3 usuli

### 1) Laravel installer (rasmiy tavsiya)

```shell
composer global require laravel/installer

laravel new mening-loyiham
```

Installer savol beradi: qaysi starter kit (React / Vue / Livewire / hech qaysi), qaysi test freymvorki (Pest / PHPUnit), qaysi ma'lumotlar bazasi. **API o'rganish uchun "none" starter kit + Pest** yetarli.

### 2) Composer orqali (installersiz)

```shell
composer create-project laravel/laravel mening-loyiham
```

### 3) Mavjud loyihani ko'tarish (aynan shu loyihadagi holat)

```shell
composer install
cp .env.example .env
php artisan key:generate
touch database/database.sqlite   # SQLite ishlatilsa
php artisan migrate
npm install && npm run build
```

Bu loyihada shu ketma-ketlik `composer.json` ichidagi skriptga yig'ilgan:

```shell
composer setup
```

> **Nega `key:generate` kerak?** `APP_KEY` — sessiya, cookie va `encrypt()` uchun ishlatiladigan shifrlash kaliti. U bo'lmasa Laravel ishga tushmaydi va "No application encryption key has been specified" xatosi chiqadi. Kalit `.env` da turadi va **git'ga tushmaydi** — har muhitda o'ziniki bo'ladi.

---

## Ishga tushirish

Eng oddiy:

```shell
php artisan serve
```

`http://127.0.0.1:8000` da ochiladi.

Lekin real ishda sizga bir vaqtda 3-4 ta jarayon kerak: veb-server, Vite (frontend), navbat worker'i va loglar. Laravel 13 da buning uchun bitta buyruq bor:

```shell
php artisan dev
```

yoki `composer run dev`. Bu buyruq mavjud jarayonlarni bitta terminalda birga ishga tushiradi. Qaysi jarayonlar borligini ko'rish uchun:

```shell
php artisan dev:list
```

Foydali qo'shimcha buyruqlar:

```shell
php artisan about        # ilova haqida umumiy ma'lumot
php artisan pail         # loglarni jonli kuzatish (laravel/pail)
php artisan tinker       # REPL: ilova ichida PHP kod yozish
php artisan docs routing # rasmiy hujjatning kerakli bo'limini ochish
```

> **Symfony bilan solishtirish:** `php artisan serve` ≈ `symfony serve`. Farqi shundaki, `artisan dev` frontend build va queue worker'ni ham o'zi ko'taradi.

---

## `.env` fayli — muhit sozlamalari

`.env` — bu **muhitga bog'liq** qiymatlar: parollar, kalitlar, URL'lar. U git'ga **tushmaydi** (`.gitignore` da bor). `.env.example` esa git'da bo'ladi va "qanday o'zgaruvchilar kerak" degan ro'yxat vazifasini bajaradi.

Bu loyihadagi muhim qatorlar:

```ini
APP_NAME=Laravel
APP_ENV=local          # local | production | testing
APP_KEY=base64:...     # key:generate qo'yadi
APP_DEBUG=true         # productionda ALBATTA false
APP_URL=http://localhost:8000

DB_CONNECTION=sqlite   # mysql | pgsql | sqlsrv | sqlite

SESSION_DRIVER=database
QUEUE_CONNECTION=database
CACHE_STORE=database
MAIL_MAILER=log        # xatlar log faylga yoziladi (lokalda qulay)
```

MySQL ishlatmoqchi bo'lsangiz:

```ini
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=secret
```

> **Nega `APP_DEBUG=true` productionda xavfli?** Chunki xatolik sahifasi stack trace, fayl yo'llari va `.env` qiymatlarining bir qismini ko'rsatadi. Bu to'g'ridan-to'g'ri xavfsizlik teshigi.

`.env` ni kodda **to'g'ridan-to'g'ri o'qimang** (`env()` faqat `config/` fayllari ichida ishlatiladi). Sababi 08-bobda: `config:cache` dan keyin `env()` `null` qaytaradi.

---

## Lokal muhit variantlari

| Vosita | Qachon |
| --- | --- |
| `php artisan serve` | Eng oddiy, hech narsa kerak emas |
| **Herd** (macOS/Windows) | Grafik interfeys, `.test` domenlar, bir nechta PHP versiyasi |
| **Sail** (Docker) | Jamoada bir xil muhit kerak bo'lsa: `./vendor/bin/sail up` |
| Nginx/Apache + PHP-FPM | Serverga eng yaqin muhit |

Veb-server sozlashda **root papka — `public/`** bo'lishi shart (`index.php` shu yerda). Boshqa papkalar internetdan ko'rinmasligi kerak — bu xavfsizlik uchun asosiy qoida.

---

## Muammolarni tez hal qilish

| Xato | Sabab / yechim |
| --- | --- |
| `No application encryption key has been specified` | `php artisan key:generate` |
| `SQLSTATE[HY000] [14] unable to open database file` | `touch database/database.sqlite` va `.env` dagi yo'lni tekshiring |
| `Unable to locate file in Vite manifest` | `npm run build` yoki `npm run dev` ishlamayapti |
| `403 Forbidden` / bo'sh sahifa | Veb-server root'i `public/` ga qaratilmagan |
| Konfiguratsiya o'zgarishi ko'rinmayapti | `php artisan config:clear` (yoki `optimize:clear`) |
| Route topilmayapti | `php artisan route:list` bilan tekshiring, keyin `route:clear` |

Barcha keshlarni bir yo'la tozalash:

```shell
php artisan optimize:clear
```

---

## Amaliyot

1. `php artisan dev` ni ishga tushiring va brauzerda bosh sahifani oching.
2. `.env` da `APP_NAME` ni o'zgartiring, keyin `php artisan tinker` da `config('app.name')` ni chaqirib natijani ko'ring.
3. `php artisan about` chiqishidan `Cache`, `Queue`, `Session` drayverlarini toping — ular `.env` dagi qiymatlarga mos kelayotganini tekshiring.
4. `php artisan optimize:clear` ni ishlating va u qanday keshlarni tozalaganini o'qing.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/installation>
- <https://laravel.com/docs/13.x/configuration>
- <https://laravel.com/docs/13.x/deployment#server-configuration> — veb-server sozlamalari

---

[← Oldingi: Kirish](01-kirish.md) · [Mundarija](README.md) · [Keyingi: Papkalar tuzilmasi →](03-papkalar-tuzilmasi.md)
