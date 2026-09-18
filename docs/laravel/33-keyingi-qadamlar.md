# 33 — Keyingi qadamlar va ekotizim

[← Oldingi: Optimallashtirish va deploy](32-optimallashtirish-va-deploy.md) · [Mundarija](README.md) · [Keyingi: Symfony → Laravel lug'ati →](34-symfony-laravel-lugat.md)

---

## Rasmiy paketlar — qachon qaysi biri

### Autentifikatsiya

| Paket | Qachon |
| --- | --- |
| **Sanctum** | API tokenlari, SPA — standart tanlov ([21-bob](21-autentifikatsiya.md)) |
| **Passport** | To'liq OAuth2 server kerak bo'lsa (uchinchi tomon ilovalari) |
| **Fortify** | Auth backend (ro'yxat, parol tiklash, 2FA) — UI o'zingizniki |
| **Socialite** | Google/GitHub/Facebook orqali kirish |

### Navbat va fon ishlari

| Paket | Qachon |
| --- | --- |
| **Horizon** | Redis navbatlari uchun panel va metrikalar |

### Qidiruv

| Paket | Qachon |
| --- | --- |
| **Scout** | To'liq matnli qidiruv (Meilisearch, Algolia, Typesense yoki baza drayveri) |

### Kuzatuv

| Paket | Qachon |
| --- | --- |
| **Telescope** | Lokal/staging debug paneli: so'rovlar, SQL, joblar, xatolar |
| **Pulse** | Production salomatligi: sekin so'rovlar, sekin joblar, xotira |
| **Nightwatch** | Laravel'ning monitoring xizmati |

### Tezlik

| Paket | Qachon |
| --- | --- |
| **Octane** | Swoole/FrankenPHP bilan doimiy ishlaydigan server |

### Frontend

| Paket | Qachon |
| --- | --- |
| **Livewire** | Dinamik UI'ni PHP'da yozish |
| **Inertia** | Vue/React komponentlari, API yozmasdan |
| **Starter kits** | Tayyor auth + UI bilan boshlash |

### To'lovlar va boshqalar

| Paket | Qachon |
| --- | --- |
| **Cashier** | Stripe/Paddle obunalari |
| **Pennant** | Feature flag'lar |
| **Reverb** | WebSocket server (real-time) |
| **Pint** | Kod formatlash (bu loyihada bor) |
| **Boost** | AI agentlari uchun MCP server (bu loyihada bor) |

Jamoa paketlari orasida eng ko'p ishlatiladiganlar: `spatie/laravel-permission` (rol/ruxsat), `spatie/laravel-medialibrary` (fayllar), `spatie/laravel-query-builder` (filter/sort), `barryvdh/laravel-debugbar`, `laravel/telescope`.

---

## Kod sifati vositalari

```shell
vendor/bin/pint                     # formatlash (bu loyihada bor)
vendor/bin/pint --dirty             # faqat o'zgargan fayllar
composer require --dev larastan/larastan   # statik tahlil (PHPStan)
vendor/bin/phpstan analyse
vendor/bin/pest --mutate            # mutatsion testlash
```

Statik tahlil (Larastan) — Symfony'dan kelgan odam uchun ayniqsa foydali: Eloquent'ning dinamik xossalarini tekshiradi va typo'larni ushlaydi.

---

## Loyihani yaxshi tuzish bo'yicha maslahatlar

1. **Kontroller yupqa bo'lsin.** Biznes mantiq — Action/Service klasslarida.

```
app/
├── Actions/          PlaceOrder, PublishPost — bitta vazifali klasslar
├── Http/
│   ├── Controllers/
│   ├── Requests/
│   ├── Resources/
│   └── Middleware/
├── Models/
├── Policies/
├── Jobs/
├── Events/ Listeners/ Observers/
└── Services/         tashqi tizimlar bilan ishlash (SMS, to'lov)
```

2. **Modelga hamma narsani tiqishtirmang.** Model — ma'lumot va aloqalar; murakkab jarayon — alohida klass.
3. **Interfeys faqat kerak bo'lganda.** Bitta implementatsiya uchun interfeys yozish — ortiqcha.
4. **Testlarni birinchi kundan yozing.** Keyinchalik yozish har doim qiyinroq.
5. **Qat'iy rejim yoqilgan bo'lsin:** `Model::shouldBeStrict()`.
6. **Har bir yangi funksiya uchun:** migratsiya → model → policy → request → resource → kontroller → route → test.

---

## O'rganishni davom ettirish

- **Rasmiy hujjat**: <https://laravel.com/docs/13.x> — eng ishonchli manba, boshidan oxirigacha o'qishga arziydi.
- **Laravel News**: <https://laravel-news.com> — yangiliklar, paketlar.
- **Laracasts**: <https://laracasts.com> — videodarslar.
- **Laravel Bootcamp**: <https://bootcamp.laravel.com> — rasmiy amaliy qo'llanma.
- **Framework kodi**: `vendor/laravel/framework/src/Illuminate/` — sehr qanday ishlashini tushunishning eng yaxshi yo'li.
- **Upgrade guide**: <https://laravel.com/docs/13.x/upgrade> — yangi versiyaga o'tishda.

---

## Symfony tajribangizni qanday ishlatish

| Symfony'dagi kuchli tomoningiz | Laravel'da qayerda asqotadi |
| --- | --- |
| DI va servis arxitekturasi | Konteyner, provider'lar, Action klasslar |
| Event-driven yondashuv | Event/Listener, Observer |
| Messenger tajribasi | Queue va Job'lar |
| Voter'lar | Policy'lar |
| Doctrine bilim | Eloquent aloqalari, N+1 muammosi |
| Console komponenti | Artisan buyruqlari |
| Test madaniyati | Pest feature testlari |

Asosiy "moslashish" nuqtasi — Active Record va konvensiyalarga ishonish. Laravel'da kamroq yozib, ko'proq ish qilish odatiy hol; buning evaziga "qaerda sozlangan?" degan savol paydo bo'ladi. Shu savolga javob — [05](05-service-container.md), [06](06-service-provider.md) va [07-boblar](07-fasadlar-va-helperlar.md).

---

## Yakuniy nazorat ro'yxati

Quyidagilarni o'zingiz yozib chiqa olsangiz — asos mustahkam:

- [ ] Yangi loyiha yaratish, `.env` sozlash, `migrate` qilish
- [ ] Model + migratsiya + factory + seeder yozish
- [ ] `apiResource` route va kontroller
- [ ] Form Request bilan validatsiya
- [ ] Policy bilan ruxsat tekshirish
- [ ] API Resource bilan JSON formatlash
- [ ] Sanctum tokeni bilan autentifikatsiya
- [ ] Job'ni navbatga qo'yish va worker ishga tushirish
- [ ] Event + Listener yozish
- [ ] Feature test yozish (200/401/403/422 holatlari)
- [ ] `optimize` bilan deploy qilish

---

[← Oldingi: Optimallashtirish va deploy](32-optimallashtirish-va-deploy.md) · [Mundarija](README.md) · [Keyingi: Symfony → Laravel lug'ati →](34-symfony-laravel-lugat.md)
