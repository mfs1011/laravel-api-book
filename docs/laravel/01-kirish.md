# 01 — Kirish: Laravel nima va nega

[← Mundarija](README.md) · [Keyingi: O'rnatish →](02-ornatish-va-ishga-tushirish.md)

---

## Laravel nima

Laravel — PHP uchun full-stack veb-freymvork. Uning ichida marshrutlash, ORM, navbatlar, kesh, pochta, testlash, konsol buyruqlari, frontend build — hammasi **bitta paketda va bitta uslubda** keladi.

Muhim fakt: Laravel **Symfony komponentlari ustiga** qurilgan. Shu loyihaning `composer.lock` faylida `symfony/http-foundation`, `symfony/http-kernel`, `symfony/routing`, `symfony/console`, `symfony/mailer` bor. Ya'ni:

- `Illuminate\Http\Request` — aslida `Symfony\Component\HttpFoundation\Request` ning kengaytmasi.
- Laravel kontrollerlaridan qaytgan `Response` — `Symfony\...\Response` avlodi.
- `php artisan` — `symfony/console` ustida ishlaydi.

**Nega bu siz uchun yaxshi xabar?** Chunki pastki qatlam sizga tanish. Siz faqat Laravel qo'shgan "yuqori qatlam"ni — konvensiyalar, fasadlar, Eloquent va Artisan generatorlarini o'rganasiz.

---

## Laravel falsafasi (nega kod shunday ko'rinadi)

Symfony "aniq sozlash" (explicit configuration) tarafdori: nima qaerda bog'lanishini siz YAML/PHP da yozasiz. Laravel esa **"konvensiya konfiguratsiyadan ustun"** tamoyilini tanlagan:

| Qoida | Laravel nima qiladi |
| --- | --- |
| `Post` modeli | avtomatik `posts` jadvaliga ulanadi |
| `PostController@show(Post $post)` | route parametridan modelni **o'zi** topib beradi |
| `app/Listeners/` ichidagi klass | hodisaga **o'zi** obuna bo'ladi |
| `App\Models\Post` | `App\Policies\PostPolicy` ni **o'zi** topadi |

**Nega bunday?** Chunki ilovalarning 90% i bir xil naqshga ega. Laravel shu 90% uchun kodni yo'q qiladi, qolgan 10% da esa konvensiyani bekor qilish imkonini beradi (masalan `protected $table = 'my_posts';`).

Ikkinchi tamoyil — **"developer experience"**: bir ishni qilishning qisqa yo'li bo'lishi kerak. Shundan kelib chiqadi fasadlar (`Cache::get()`), helperlar (`route()`, `now()`, `str()`), va `make:` generatorlari.

---

## Symfony bilan yuqori darajadagi solishtirish

| Symfony | Laravel | Izoh |
| --- | --- | --- |
| `bin/console` | `php artisan` | Bir xil g'oya, Laravel'da generator buyruqlari ko'proq |
| `config/services.yaml` | `App\Providers\AppServiceProvider` (PHP kodi) | Laravel'da DI sozlash — YAML emas, PHP |
| Autowiring | Bir xil — konteyner tip bo'yicha avtomatik hal qiladi | [05-bob](05-service-container.md) |
| Doctrine (Data Mapper) | Eloquent (Active Record) | **Eng katta farq**, [16-bob](16-eloquent-asoslari.md) |
| Twig | Blade | [30-bob](30-blade-va-frontend.md) |
| `kernel.request` listener | Middleware | [10-bob](10-middleware.md) |
| Security bundle (firewall, voters) | Guard + Policy | [21](21-autentifikatsiya.md), [22](22-avtorizatsiya.md) |
| Messenger | Queue + Job | [25-bob](25-navbatlar.md) |
| Form component | Form Request + Validator | [13-bob](13-validatsiya.md) |
| Serializer / API Platform | Eloquent API Resource | [20-bob](20-api-resurslar.md) |
| `.env` (Dotenv) | `.env` (bir xil kutubxona: `vlucas/phpdotenv`) | [08-bob](08-konfiguratsiya-va-muhit.md) |

Eng katta psixologik farq: **Doctrine → Eloquent**. Doctrine'da entity "ahmoq" ob'ekt, saqlash `EntityManager` ishi. Eloquent'da model o'zi so'rov yozadi va o'zini saqlaydi: `$post->save()`. Bu Active Record naqshi. Nega? Chunki tipik CRUD ilovada bu kamroq kod va tezroq o'qiladi. Murakkab domenda esa siz baribir alohida "Service"/"Action" klasslar yozasiz — Laravel buni taqiqlamaydi.

---

## Versiyalar va qo'llab-quvvatlash

- Laravel har yili bitta katta versiya chiqaradi (masalan 12 → 13).
- Har bir versiya: ~18 oy bug-fix, ~2 yil xavfsizlik tuzatishlari.
- **Bu loyihada: Laravel 13.32, PHP 8.5.** Hujjat o'qiyotganda doim `laravel.com/docs/13.x` versiyasini tanlang, chunki 11/12 versiyalaridagi misollar farq qiladi.

Laravel 13 da uchraydigan, eski darsliklardan farq qiladigan narsalar (ular tegishli boblarda tushuntiriladi):

- Modelda `protected $fillable = [...]` o'rniga `#[Fillable([...])]` atributi ham ishlaydi → [16-bob](16-eloquent-asoslari.md).
- Scope'lar `scopePopular()` o'rniga `#[Scope]` atributi bilan → [16-bob](16-eloquent-asoslari.md).
- CSRF middleware nomi `VerifyCsrfToken` emas, `PreventRequestForgery` → [10-bob](10-middleware.md).
- `JsonApiResource` (JSON:API standarti) freymvork ichida → [20-bob](20-api-resurslar.md).
- Sana API: `now()->addDays(3)` bilan bir qatorda `now()->plus(days: 3)` (Carbon 3) ham bor.

---

## Ekotizim — nima freymvork ichida, nima alohida

Freymvork ichida (qo'shimcha o'rnatishsiz): routing, Eloquent, validatsiya, queue, cache, mail, notifications, events, scheduler, testing, storage, HTTP client.

Rasmiy, lekin alohida o'rnatiladigan paketlar (kerak bo'lganda):

| Paket | Vazifasi |
| --- | --- |
| **Sanctum** | API token / SPA autentifikatsiyasi (`php artisan install:api` o'rnatadi) |
| **Passport** | To'liq OAuth2 server |
| **Fortify** | Backend auth logikasi (UI siz) |
| **Horizon** | Redis navbatlari uchun boshqaruv paneli |
| **Telescope** | Lokal debug paneli (so'rovlar, so'rovlar, joblar) |
| **Scout** | To'liq matnli qidiruv |
| **Octane** | Doimiy ishlaydigan server (Swoole/FrankenPHP) orqali tezlik |
| **Pint** | Kod formatlash (bu loyihada bor) |
| **Boost** | AI agentlar uchun MCP server (bu loyihada bor) |

33-bobda ularning har biri qachon kerakligini ko'rasiz.

---

## Nima o'qish tartibi to'g'ri

Ko'p odam Eloquent'dan boshlaydi va keyin "bu sehr qayerdan keldi?" degan savolga tushib qoladi. Shuning uchun bu qo'llanmada tartib shunday:

1. **Avval poydevor** (konteyner, provider, fasad) — shunda hech narsa "sehr" bo'lib ko'rinmaydi.
2. Keyin HTTP qatlami (route → middleware → controller → response).
3. Keyin ma'lumotlar bazasi.
4. Keyin API, auth, fon ishlari.
5. Oxirida — to'liq amaliy loyiha.

---

## Amaliyot

1. `composer show --direct` buyrug'ini ishga tushiring va loyihada qanday to'g'ridan-to'g'ri paketlar borligini ko'ring.
2. `php artisan about` buyrug'ini ishga tushiring — PHP versiyasi, kesh drayveri, muhit ko'rinadi.
3. `composer show laravel/framework` chiqishida `symfony/*` paketlarni toping — Laravel qaysi Symfony komponentlariga tayanishini o'z ko'zingiz bilan ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/installation> — umumiy kirish
- <https://laravel.com/docs/13.x/releases> — versiya siyosati va yangiliklar

---

[← Mundarija](README.md) · [Keyingi: O'rnatish va ishga tushirish →](02-ornatish-va-ishga-tushirish.md)
