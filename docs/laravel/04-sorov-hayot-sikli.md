# 04 — So'rovning hayot sikli (Request Lifecycle)

[← Oldingi: Papkalar tuzilmasi](03-papkalar-tuzilmasi.md) · [Mundarija](README.md) · [Keyingi: Service Container →](05-service-container.md)

---

## Nega bu bobni o'tkazib yubormaslik kerak

Laravel'da ko'p narsa "o'zidan-o'zi" ishlaydi: model route parametridan topiladi, validatsiya xatosi JSON bo'lib qaytadi, foydalanuvchi `auth()->user()` da paydo bo'ladi. Agar siz **bu qadamlar qayerda sodir bo'lishini** bilsangiz, sehr tugaydi va debug qilish oson bo'ladi.

---

## To'liq yo'l — bosqichma-bosqich

```
1. public/index.php          ← veb-server har bir so'rovni shu yerga yuboradi
2. vendor/autoload.php       ← Composer autoloader
3. bootstrap/app.php         ← Application ob'ekti yig'iladi (konteyner)
4. HTTP Kernel               ← bootstrap'lar: .env, config, loglar, provider'lar
5. Service Provider'lar      ← register() → keyin boot()
6. Global middleware         ← so'rov router'gacha bu yerdan o'tadi
7. Router                    ← mos route topiladi
8. Route/guruh middleware    ← web/api guruhi, auth, throttle...
9. Controller (yoki closure) ← sizning kodingiz
10. Response                 ← javob qaytadi
11. Middleware teskari tartibda javobni ko'radi
12. index.php javobni yuboradi  →  terminable middleware ishlaydi
```

---

## 1-2. Kirish nuqtasi

`public/index.php` — **yagona** kirish nuqtasi. Barcha URL'lar shu faylga tushadi (veb-server rewrite qoidasi orqali). Uning ichida uch ish bo'ladi: autoloader ulanadi, `bootstrap/app.php` dan ilova olinadi, so'rov yuboriladi va javob chiqariladi.

> **Symfony bilan solishtirish:** bir xil g'oya — `public/index.php` + `Kernel::handle()`.

## 3. Application ob'ekti = konteyner

`bootstrap/app.php` `Illuminate\Foundation\Application` ob'ektini qaytaradi. Bu ob'ekt bir vaqtning o'zida:

- **service container** (DI konteyner) — [05-bob](05-service-container.md)
- ilova sozlamalari (route fayllari, middleware, exception handling)

## 4. Kernel va "bootstrapper"lar

HTTP Kernel so'rovni qabul qilishdan oldin quyidagilarni ketma-ket bajaradi:

1. `.env` o'qiladi
2. `config/` yuklanadi (yoki `bootstrap/cache/config.php` keshidan)
3. Xatoliklarni ushlash sozlanadi
4. Fasadlar ro'yxatdan o'tadi
5. Service provider'lar `register()` → keyin `boot()`

## 5. Service provider'lar — eng muhim bosqich

Bu yerda ilova "yig'iladi": DB ulanishi, kesh, navbat, validatsiya, autentifikatsiya — hammasi provider'lar orqali konteynerga bog'lanadi. Sizning `AppServiceProvider` ham shu yerda ishlaydi.

**Qoida:** `register()` da faqat konteynerga bog'lash; boshqa xizmatlardan foydalanish `boot()` da. Nega — [06-bob](06-service-provider.md) da.

## 6-8. Middleware va router

So'rov middleware "quvuri"dan (pipeline) o'tadi. Har bir middleware so'rovni ko'radi, keyin `$next($request)` orqali navbatdagisiga uzatadi:

```php
public function handle(Request $request, Closure $next): Response
{
    // so'rovdan OLDIN

    $response = $next($request);

    // javobdan KEYIN

    return $response;
}
```

Shuning uchun middleware'lar **ikki tomonlama** ishlaydi: kirishda birinchidan oxirgisiga, chiqishda teskari tartibda.

`routes/web.php` dagi route'lar avtomatik `web` guruhiga, `routes/api.php` dagilar `api` guruhiga tushadi. Laravel 13 da `web` guruhi tarkibi:

| `web` guruhi |
| --- |
| `EncryptCookies` |
| `AddQueuedCookiesToResponse` |
| `StartSession` |
| `ShareErrorsFromSession` |
| `PreventRequestForgery` (CSRF himoyasi) |
| `SubstituteBindings` (route model binding) |

`api` guruhi esa faqat `SubstituteBindings` dan iborat — **stateless**, sessiya ham, CSRF ham yo'q.

> **Nega API da sessiya yo'q?** Chunki API token bilan autentifikatsiya qilinadi; har so'rov mustaqil bo'lishi kerak. Sessiya bo'lsa, u ortiqcha holat (state) va ortiqcha so'rov (session store) demakdir.

## 9-10. Controller va javob

Controller metodi qaytargan narsa avtomatik `Response` ga aylantiriladi:

| Siz qaytarasiz | Laravel qiladi |
| --- | --- |
| `string` | HTML javob |
| `array` / `Collection` | JSON javob (`Content-Type: application/json`) |
| Eloquent model | JSON (model `toJson()` orqali) |
| `view('...')` | Blade render |
| `JsonResource` | JSON (`data` kaliti bilan) |
| `response()->json([...], 201)` | Aniq status bilan JSON |

## 11-12. Javob va terminable middleware

Javob mijozga yuborilgandan **keyin** ham kod ishlashi mumkin — bu `terminate()` metodi bo'lgan middleware. Masalan sessiyani yozib qo'yish shu yerda bo'ladi. Foydasi: foydalanuvchi javobni tezroq oladi.

---

## Konsol (CLI) so'rovi qanday ketadi

`php artisan ...` uchun yo'l deyarli bir xil, faqat HTTP Kernel o'rniga Console Kernel ishlaydi va middleware bo'lmaydi:

```
artisan → bootstrap/app.php → Console Kernel → provider'lar → Command → exit code
```

Shuning uchun `artisan tinker` ichida sizning barcha konfiguratsiyangiz, DB ulanishingiz va servislaringiz mavjud bo'ladi.

---

## Debug qilishda foydali nuqtalar

```shell
php artisan route:list --path=api      # qaysi route bor va qanday middleware'da
php artisan about                      # drayverlar va muhit
php artisan event:list                 # qaysi listener nimaga obuna
php artisan pail                       # loglarni jonli ko'rish
```

Kod ichida:

```php
dd($request->all());   // dump and die
dump($user);           // to'xtatmasdan chiqarish
logger()->info('bu yerga yetdi', ['id' => $id]);
```

---

## Amaliyot

1. `routes/web.php` ga quyidagini qo'shing va brauzerda `/hayot-sikli` ni oching:

```php
Route::get('/hayot-sikli', function (Illuminate\Http\Request $request) {
    return [
        'url' => $request->url(),
        'middleware' => 'web guruhi',
        'user' => $request->user(),
    ];
});
```

Massiv qaytganini va javob JSON bo'lganini ko'ring — bu 10-qadamning isboti.

2. `php artisan route:list` da shu route qatoriga qarang: `web` guruhi ko'rinadi.
3. `app/Providers/AppServiceProvider.php` ning `boot()` metodiga `logger()->info('boot ishladi');` qo'ying, sahifani yangilang va `storage/logs/laravel.log` ni tekshiring. Keyin uni o'chirib tashlang.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/lifecycle>
- <https://laravel.com/docs/13.x/middleware>

---

[← Oldingi: Papkalar tuzilmasi](03-papkalar-tuzilmasi.md) · [Mundarija](README.md) · [Keyingi: Service Container →](05-service-container.md)
