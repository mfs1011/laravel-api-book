# 40 — CI/CD, Docker va production operatsiyalari

[← Oldingi: Arxitektura](39-arxitektura.md) · [Mundarija](README.md)

---

## 1. Lokal muhit: Docker (Sail)

```shell
php artisan sail:install        # xizmatlarni tanlaysiz: mysql, redis, meilisearch...
./vendor/bin/sail up -d
./vendor/bin/sail artisan migrate
./vendor/bin/sail test
./vendor/bin/sail npm run dev
```

Qulaylik uchun: `alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'`.

**Nega Docker?** Jamoadagi har bir dasturchi va CI bir xil PHP, MySQL va Redis versiyalarida ishlaydi. "Menda ishlayapti" muammosi yo'qoladi.

---

## 2. CI: GitHub Actions

`.github/workflows/ci.yml`:

```yaml
name: CI

on:
  pull_request:
  push:
    branches: [main, dev]

jobs:
  test:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - uses: shivammathur/setup-php@v2
        with:
          php-version: '8.5'
          extensions: mbstring, pdo_sqlite, intl, bcmath
          coverage: none

      - name: Composer keshi
        uses: actions/cache@v4
        with:
          path: ~/.composer/cache
          key: composer-${{ hashFiles('composer.lock') }}

      - run: composer install --prefer-dist --no-interaction --no-progress

      - name: Muhit
        run: |
          cp .env.example .env
          php artisan key:generate

      - name: Kod uslubi
        run: vendor/bin/pint --test

      - name: Statik tahlil
        run: vendor/bin/phpstan analyse --no-progress
        continue-on-error: true      # larastan o'rnatilgach: false

      - name: Testlar
        run: php artisan test --parallel
        env:
          DB_CONNECTION: sqlite
          DB_DATABASE: ':memory:'

  frontend:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: npm
      - run: npm ci
      - run: npm run build
```

**CI'da nima tekshirilishi kerak:** formatlash (Pint), statik tahlil (PHPStan/Larastan), testlar, frontend build, `composer audit`. Bularning barchasi PR birlashtirilishidan **oldin** yashil bo'lishi kerak.

`composer.json` ga qulay skript qo'shing:

```json
"scripts": {
    "check": ["vendor/bin/pint --test", "vendor/bin/phpstan analyse", "@php artisan test"]
}
```

---

## 3. Production uchun Docker obraz (ixtiyoriy)

```dockerfile
# 1-bosqich: frontend
FROM node:22-alpine AS assets
WORKDIR /app
COPY package*.json vite.config.js ./
RUN npm ci
COPY resources ./resources
RUN npm run build

# 2-bosqich: PHP bog'liqliklari
FROM composer:2 AS vendor
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --prefer-dist --optimize-autoloader

# 3-bosqich: yakuniy obraz
FROM php:8.5-fpm-alpine
RUN docker-php-ext-install pdo_mysql opcache bcmath
WORKDIR /var/www
COPY . .
COPY --from=vendor /app/vendor ./vendor
COPY --from=assets /app/public/build ./public/build
RUN php artisan optimize
CMD ["php-fpm"]
```

Eslatmalar:

- `.env` ni **obraz ichiga qo'ymang** — sirlar orkestratorda (Kubernetes secret, ECS task definition) turadi.
- `php artisan optimize` ni build paytida bajarish mumkin, lekin `config:cache` `.env` qiymatlarini "muzlatadi" — agar sozlamalar ishga tushish paytida kelsa, `optimize` ni **konteyner startida** bajaring.
- Alohida konteynerlar: `php-fpm` (web), `queue:work` (worker), `schedule:work` (cron), `reverb:start` (WebSocket).

---

## 4. Deploy strategiyalari

| Usul | Qachon |
| --- | --- |
| **Laravel Cloud** | Eng tez yo'l: baza, Redis, worker, scheduler — hammasi boshqariladi |
| **Forge + Envoyer** | O'z VPS'ingiz, nol uzilishli deploy |
| **Docker + Kubernetes/ECS** | Katta jamoa, murakkab infratuzilma |
| **Qo'lda VPS** | Kichik loyiha; skript bilan avtomatlashtiring |

Nol uzilishli deploy mantiqi (Envoyer/Deployer shuni qiladi):

```
releases/2026_09_18_120000/   ← yangi versiya shu yerga yig'iladi
storage/  .env                ← umumiy (shared) — symlink
current → releases/...        ← oxirida symlink almashtiriladi
```

Deploy skripti:

```shell
composer install --no-dev --optimize-autoloader
npm ci && npm run build
php artisan migrate --force
php artisan optimize
php artisan queue:restart
php artisan reverb:restart     # broadcasting bo'lsa
```

> **Migratsiya tartibi muhim.** Eski va yangi kod bir muddat birga ishlaydi (deploy davomida). Shuning uchun **buzuvchi migratsiyalarni ikki bosqichda** qiling: avval yangi ustun qo'shish (nullable), kod yangilanishi, keyin eski ustunni o'chirish. Ustunni darhol o'chirsangiz — eski kod 500 beradi.

GitHub Actions'dan deploy:

```yaml
  deploy:
    needs: [test, frontend]
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Envoyer/Forge webhook
        run: curl -sS -X POST "${{ secrets.DEPLOY_WEBHOOK }}"
```

---

## 5. Serverda ishlashi kerak bo'lgan jarayonlar

| Jarayon | Buyruq | Qanday ishlaydi |
| --- | --- | --- |
| Web | `php-fpm` / `nginx` | Doimiy |
| Navbat | `php artisan queue:work --max-time=3600` | Supervisor, bir nechta nusxa |
| Scheduler | `php artisan schedule:run` | Cron, har daqiqada |
| WebSocket | `php artisan reverb:start` | Supervisor |
| Horizon (Redis) | `php artisan horizon` | Supervisor (queue:work o'rniga) |

Supervisor namunasi:

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/current/artisan queue:work --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopwaitsecs=3600
numprocs=4
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/worker.log
```

Cron:

```cron
* * * * * cd /var/www/current && php artisan schedule:run >> /dev/null 2>&1
```

> **`stopwaitsecs` nega katta?** Deploy paytida worker joriy job'ni tugatishga ulgursin — o'rtada uzilsa, ish yarim bajarilgan holatda qoladi.

---

## 6. Sirlar (secrets) bilan ishlash

- `.env` git'ga **hech qachon** tushmaydi; `.env.example` esa doim yangilanib boradi.
- CI/CD'da — platformaning secret saqlagichi (GitHub Secrets, Cloud env).
- Repozitoriyda shifrlangan holda saqlash kerak bo'lsa: `php artisan env:encrypt` ([08-bob](08-konfiguratsiya-va-muhit.md)).
- Kalit sizib chiqsa: darhol rotatsiya (`APP_KEY` almashtirilsa, eski shifrlangan ma'lumot o'qilmay qoladi — avval qayta shifrlash rejasini tuzing).
- Kodga tasodifan tushgan sirni topish uchun CI'ga `gitleaks` kabi skanerni qo'shing.

---

## 7. Zaxira (backup) va tiklash

```shell
composer require spatie/laravel-backup
php artisan backup:run
```

```php
Schedule::command('backup:clean')->daily()->at('01:00');
Schedule::command('backup:run')->daily()->at('02:00');
Schedule::command('backup:monitor')->daily()->at('09:00');
```

> **Eng muhim qoida:** tiklashni sinab ko'rmagan zaxira — zaxira emas. Har chorakda "bazani noldan tiklash" mashqini o'tkazing va qancha vaqt ketishini o'lchang.

Saqlash: kamida bitta nusxa **boshqa provayderda** bo'lsin.

---

## 8. Monitoring va ogohlantirish

| Nima kuzatiladi | Vosita |
| --- | --- |
| Ilova tirikmi | `/up` health endpoint + tashqi uptime monitor |
| Xatolar (5xx, istisnolar) | Sentry / Flare / Nightwatch |
| Ish unumdorligi (p95, sekin so'rovlar) | Pulse, APM |
| Navbat holati | Horizon, `queue:monitor` |
| Server resurslari | CPU, RAM, disk (disk to'lishi — eng ko'p uchraydigan avariya) |
| Loglar | `stderr` → markazlashgan log yig'gich |

Ogohlantirish (alert) sozlash:

```php
Schedule::command('queue:monitor database:default --max=200')->everyFiveMinutes();
```

```php
// AppServiceProvider::boot()
Queue::failing(fn ($event) => Log::channel('slack')->critical('Job tushdi', [
    'job' => $event->job->resolveName(),
]));
```

Alert **harakatga chaqiradigan** bo'lsin: agar hech kim reaksiya qilmasa — u alert emas, shovqin.

---

## 9. Xatoni tez topish uchun (production runbook)

| Simptom | Birinchi qadamlar |
| --- | --- |
| 500 xatolar ko'paydi | Sentry/loglar → `storage/logs` → oxirgi deploy nima o'zgartirgan |
| Sayt sekin | `DB::whenQueryingForLongerThan` loglari, N+1, indekslar, Pulse |
| Navbat to'planib qoldi | `queue:monitor`, worker tirikmi, `failed_jobs`, tashqi API sekinmi |
| Xatlar bormayapti | mail drayveri, navbat worker'i, provayder limiti |
| Disk to'ldi | `storage/logs` (log rotatsiyasi), eski backup, `failed_jobs` |
| Deploy'dan keyin eski kod ishlayapti | `queue:restart`, `optimize:clear`, opcache |
| Sessiya/login "uchib ketdi" | `APP_KEY` o'zgargan, sessiya drayveri, bir nechta serverda umumiy store yo'q |

Foydali buyruqlar:

```shell
php artisan about
php artisan queue:failed
php artisan pail --level=error
php artisan down --secret=token     # tekshirish uchun
```

---

## 10. Yakuniy production ro'yxati

### Kod va sifat
- [ ] CI yashil: Pint, PHPStan, testlar, frontend build
- [ ] Muhim oqimlar test bilan qoplangan (auth, to'lov, asosiy CRUD)
- [ ] `dd()`, `dump()`, `ray()` qolmagan

### Konfiguratsiya
- [ ] `APP_ENV=production`, `APP_DEBUG=false`, `APP_KEY` o'rnatilgan
- [ ] `php artisan optimize` deploy'da bajariladi
- [ ] `env()` faqat `config/` ichida ishlatilgan
- [ ] `LOG_LEVEL` production uchun mos (`warning`+)

### Xavfsizlik
- [ ] HTTPS majburiy, veb-server root — `public/`
- [ ] Barcha yozuv endpointlarida validatsiya va avtorizatsiya
- [ ] `throttle` login/register/parol tiklashda
- [ ] Mass assignment himoyasi (`#[Fillable]`)
- [ ] CORS'da aniq domenlar; sirlar repozitoriyda yo'q
- [ ] `composer audit` / `npm audit` toza

### Ma'lumotlar bazasi
- [ ] Indekslar qo'yilgan; N+1 yo'q (`Model::shouldBeStrict`)
- [ ] Migratsiyalar orqaga mos (ikki bosqichli buzuvchi o'zgarishlar)
- [ ] Zaxira ishlaydi **va tiklash sinab ko'rilgan**

### Ishlash
- [ ] Ro'yxatlar sahifalangan, `per_page` chegaralangan
- [ ] Og'ir ishlar navbatda; worker supervisor ostida
- [ ] Kesh strategiyasi va bekor qilish (invalidation) aniq

### Operatsiyalar
- [ ] `/up` monitoring ostida
- [ ] Xatolar Sentry/Flare'ga tushadi; alertlar kimgadir boradi
- [ ] Scheduler cron'i o'rnatilgan va `schedule:list` to'g'ri
- [ ] Deploy skripti `queue:restart` ni chaqiradi
- [ ] Rollback rejasi bor (oldingi relizga qaytish)

---

## Yakun

Shu ro'yxatni to'ldira olsangiz — loyihangiz production darajasida. Undan keyingi bilim real trafik, real xatolar va real jamoadan keladi.

Boshidan takrorlash kerak bo'lsa: [Mundarija](README.md).

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/deployment>
- <https://laravel.com/docs/13.x/sail>
- <https://laravel.com/docs/13.x/horizon>
- <https://laravel.com/docs/13.x/pulse>

---

[← Oldingi: Arxitektura](39-arxitektura.md) · [Mundarija](README.md)
