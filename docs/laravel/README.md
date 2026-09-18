# Laravel 13 — noldan to'liq qo'llanma (Symfony dasturchilari uchun)

Bu qo'llanma **Laravel 13.x** rasmiy hujjatlari asosida yozilgan va shu loyihadagi haqiqiy versiyalarga moslangan:

| Narsa | Versiya |
| --- | --- |
| PHP | 8.5 |
| Laravel Framework | 13.32 |
| Pest (test) | 5.x |
| Vite / Tailwind | 8.x / 4.x |
| Ma'lumotlar bazasi | SQLite (standart) |

> Har bir bo'lim oxirida **"Rasmiy hujjat"** havolasi bor: `https://laravel.com/docs/13.x/...`.
> Agar biror narsa shubhali bo'lsa, doim rasmiy hujjat birinchi manba.

---

## Bu qo'llanma kimga

Sizda Symfony tajribasi bor — ya'ni siz allaqachon bilasiz: Router, Controller, DI Container, Middleware (Symfony'da "Event listener / kernel.request"), Doctrine, Console commands, Twig. Demak Laravel'da yangi bo'lgan narsa **tushunchalar emas, balki ularning ko'rinishi va konvensiyalari**.

Shuning uchun har bir bobda ikki qism bor:

- **"Nega bunday?"** — Laravel shunday qilishining sababi (bu savol ochiq qolmasligi uchun).
- **"Symfony bilan solishtirish"** — sizdagi bilimni Laravel'ga ko'chirish uchun jadval.

---

## O'rganish ketma-ketligi

Boblarni **tartib bilan** o'qing. Har bir bob oldingisiga tayanadi.

### I qism — Poydevor (1–8)

| № | Bob | Nima o'rganasiz |
| --- | --- | --- |
| 01 | [Kirish: Laravel nima va nega](01-kirish.md) | Falsafa, versiyalar, ekotizim |
| 02 | [O'rnatish va ishga tushirish](02-ornatish-va-ishga-tushirish.md) | Composer, installer, `artisan dev`, `.env` |
| 03 | [Papkalar tuzilmasi](03-papkalar-tuzilmasi.md) | `app/`, `bootstrap/app.php`, `routes/`, `config/` |
| 04 | [So'rovning hayot sikli](04-sorov-hayot-sikli.md) | `index.php` dan javobgacha bo'lgan yo'l |
| 05 | [Service Container](05-service-container.md) | DI, binding, singleton, avtomatik resolve |
| 06 | [Service Provider](06-service-provider.md) | `register()` va `boot()`, ilovani yig'ish |
| 07 | [Fasadlar va helperlar](07-fasadlar-va-helperlar.md) | `Cache::get()` aslida nima qiladi |
| 08 | [Konfiguratsiya va muhit](08-konfiguratsiya-va-muhit.md) | `.env`, `config()`, `config:cache` |

### II qism — HTTP qatlami (9–13)

| № | Bob | Nima o'rganasiz |
| --- | --- | --- |
| 09 | [Marshrutlash (Routing)](09-marshrutlash.md) | Route'lar, parametrlar, guruhlar, model binding |
| 10 | [Middleware](10-middleware.md) | So'rovni filtrlash, guruhlar, tartib |
| 11 | [Kontrollerlar](11-kontrollerlar.md) | Resource controller, DI, invokable |
| 12 | [So'rov va javob](12-sorov-va-javob.md) | `Request`, `Response`, JSON, fayllar |
| 13 | [Validatsiya](13-validatsiya.md) | `validate()`, Form Request, xato formati |

### III qism — Ma'lumotlar bazasi (14–19)

| № | Bob | Nima o'rganasiz |
| --- | --- | --- |
| 14 | [Ma'lumotlar bazasi va Query Builder](14-malumotlar-bazasi.md) | Ulanish, so'rovlar, tranzaksiya |
| 15 | [Migratsiyalar](15-migratsiyalar.md) | Schema, ustunlar, tashqi kalitlar |
| 16 | [Eloquent asoslari](16-eloquent-asoslari.md) | Model, CRUD, cast, scope, soft delete |
| 17 | [Eloquent aloqalari](17-eloquent-aloqalar.md) | `hasMany`, `belongsTo`, N+1 muammosi |
| 18 | [Factory va Seeder](18-factory-va-seeder.md) | Test ma'lumotlari |
| 19 | [Kolleksiyalar](19-kolleksiyalar.md) | `Collection` API, lazy collection |

### IV qism — API qurish (20–23)

| № | Bob | Nima o'rganasiz |
| --- | --- | --- |
| 20 | [API Resurslar](20-api-resurslar.md) | JSON transformatsiya, paginatsiya, JSON:API |
| 21 | [Autentifikatsiya](21-autentifikatsiya.md) | Guard, provider, Sanctum tokenlari |
| 22 | [Avtorizatsiya](22-avtorizatsiya.md) | Gate, Policy, `#[Authorize]` |
| 23 | [Xatoliklar va loglar](23-xatoliklar-va-loglar.md) | `withExceptions`, JSON xato, Monolog |

### V qism — Fon ishlari va xizmatlar (24–28)

| № | Bob | Nima o'rganasiz |
| --- | --- | --- |
| 24 | [Artisan va konsol](24-artisan-va-konsol.md) | Buyruqlar, Tinker, Scheduler |
| 25 | [Navbatlar (Queue)](25-navbatlar.md) | Job, worker, retry, batch |
| 26 | [Hodisalar va Observerlar](26-hodisalar-va-observerlar.md) | Event, Listener, Model observer |
| 27 | [Kesh va fayllar](27-kesh-va-fayllar.md) | Cache, Storage, HTTP client |
| 28 | [Mail va bildirishnomalar](28-mail-va-bildirishnoma.md) | Mailable, Notification |

### VI qism — Sifat va yakun (29–34)

| № | Bob | Nima o'rganasiz |
| --- | --- | --- |
| 29 | [Testlash (Pest)](29-testlash.md) | Feature test, fake'lar, HTTP testlari |
| 30 | [Blade va frontend](30-blade-va-frontend.md) | Blade, Vite, Tailwind |
| 31 | [Amaliy loyiha: to'liq API](31-amaliy-loyiha-api.md) | Noldan CRUD + auth + testlar |
| 32 | [Optimallashtirish va deploy](32-optimallashtirish-va-deploy.md) | `optimize`, kesh, serverga chiqarish |
| 33 | [Keyingi qadamlar](33-keyingi-qadamlar.md) | Ekotizim: Horizon, Telescope, Octane... |
| 34 | [Symfony → Laravel lug'ati](34-symfony-laravel-lugat.md) | Tez ma'lumotnoma jadvali |

---

## Qanday o'rganish kerak (amaliy maslahat)

1. **Har bobni o'qing, keyin kodini o'zingiz terib, terminalda ishlating.** O'qib chiqish yetarli emas.
2. Har bobdagi **"Amaliyot"** bo'limini bajaring — ular bir-biriga bog'langan: 31-bobdagi loyiha shu mashqlardan yig'iladi.
3. Tushunmagan joyda **`php artisan tinker`** ochib tajriba qiling.
4. Rasmiy hujjatni **yon oyna**da ochib turing: <https://laravel.com/docs/13.x>.

Boshlash: [01 — Kirish: Laravel nima va nega →](01-kirish.md)
