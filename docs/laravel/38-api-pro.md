# 38 — API pro darajasi

[← Oldingi: Ilg'or Eloquent](37-ilgor-eloquent-va-unumdorlik.md) · [Mundarija](README.md) · [Keyingi: Arxitektura →](39-arxitektura.md)

---

Bu bob — API'ni "ishlaydi" holatidan **"boshqa jamoalar ishonch bilan ulanadigan"** holatga olib chiqadigan narsalar haqida.

---

## 1. Xato shartnomasi (error contract)

API'ning eng muhim qismi — xatolarning **bashorat qilinadigan** bo'lishi. Laravel standarti allaqachon yaxshi:

```json
{
  "message": "Ma'lumotlar noto'g'ri.",
  "errors": { "title": ["Sarlavha majburiy."] }
}
```

Buni kengaytirish kerak bo'lsa, mijoz uchun **mashina o'qiydigan kod** qo'shing:

```php
// bootstrap/app.php
$exceptions->render(function (DomainException $e, Request $request) {
    if (! $request->expectsJson()) {
        return null;
    }

    return response()->json([
        'message' => $e->getMessage(),
        'code' => $e->errorCode(),          // 'insufficient_balance'
        'meta' => $e->context(),
    ], $e->statusCode());
});
```

**Qoidalar:**

- Status kodi **doim to'g'ri** bo'lsin (422 validatsiya, 403 ruxsat, 409 konflikt, 404 yo'q) — mijoz `message` matnini tahlil qilishga majbur bo'lmasin.
- Xato matni o'zgarishi mumkin, **kod o'zgarmasin**.
- Productionda 500 xatoning ichki tafsilotlari chiqmasin ([23-bob](23-xatoliklar-va-loglar.md)).
- Har javobga `X-Request-Id` qo'shing — mijoz shu ID bilan murojaat qiladi, siz loglardan topasiz.

```php
// middleware
$requestId = $request->header('X-Request-Id') ?? (string) Str::uuid();
Context::add('request_id', $requestId);
$response = $next($request);
$response->headers->set('X-Request-Id', $requestId);
```

---

## 2. Idempotentlik (takroriy so'rovlar)

Mijozning interneti uzildi, u `POST /payments` ni **qayta** yubordi. Pul ikki marta yechilmasligi kerak.

Yechim — **Idempotency-Key** sarlavhasi:

```php
namespace App\Http\Middleware;

class Idempotent
{
    public function handle(Request $request, Closure $next): Response
    {
        $key = $request->header('Idempotency-Key');

        if (! $key || ! $request->isMethod('post')) {
            return $next($request);
        }

        $cacheKey = "idem:{$request->user()->id}:{$key}";

        if ($cached = Cache::get($cacheKey)) {
            return response($cached['body'], $cached['status'])
                ->header('Idempotent-Replay', 'true');
        }

        // bir vaqtda ikkita bir xil so'rov kelsa
        $lock = Cache::lock("{$cacheKey}:lock", 30);

        if (! $lock->get()) {
            return response()->json(['message' => 'So\'rov bajarilmoqda'], 409);
        }

        try {
            $response = $next($request);

            if ($response->isSuccessful()) {
                Cache::put($cacheKey, [
                    'body' => $response->getContent(),
                    'status' => $response->getStatusCode(),
                ], now()->addHours(24));
            }

            return $response;
        } finally {
            $lock->release();
        }
    }
}
```

Bu naqsh Stripe, PayPal va boshqa to'lov API'larida standart. To'lov, buyurtma yaratish, SMS yuborish kabi **takrorlanmasligi kerak** bo'lgan endpointlarga qo'ying.

---

## 3. Shartli so'rovlar: ETag va 304

Mijoz o'zgarmagan ma'lumotni qayta yuklamasin:

```php
Route::middleware('cache.headers:private;max_age=60;etag')->group(function () {
    Route::get('/posts', [PostController::class, 'index']);
});
```

Mijoz keyingi safar `If-None-Match: "<etag>"` yuboradi va o'zgarmagan bo'lsa **304 Not Modified** oladi — tana umuman uzatilmaydi.

Yozishda ham foydali — "kim oldin ulgursa" muammosiga qarshi:

```php
// mijoz: If-Match: "<etag>"
if ($request->header('If-Match') && $request->header('If-Match') !== $post->etag()) {
    abort(412, 'Yozuv o\'zgargan.');       // 412 Precondition Failed
}
```

`Last-Modified` + `If-Modified-Since` ham xuddi shunday ishlaydi.

---

## 4. Versiyalash va eskirtirish (deprecation)

```php
// routes/api.php
Route::prefix('v1')->group(base_path('routes/api_v1.php'));
Route::prefix('v2')->group(base_path('routes/api_v2.php'));
```

Papkalar: `app/Http/Controllers/Api/V1/`, `app/Http/Resources/V1/`.

Eski versiyani o'chirishdan oldin **ogohlantiring**:

```php
$response->headers->set('Deprecation', 'true');
$response->headers->set('Sunset', 'Sat, 01 Nov 2026 00:00:00 GMT');
$response->headers->set('Link', '<https://docs.example.uz/v2>; rel="deprecation"');
```

**Buzuvchi o'zgarish nima hisoblanadi:**

| Buzadi | Buzmaydi |
| --- | --- |
| Maydonni o'chirish/nomini o'zgartirish | Yangi ixtiyoriy maydon qo'shish |
| Maydon tipini o'zgartirish | Yangi endpoint qo'shish |
| Majburiy parametr qo'shish | Yangi ixtiyoriy parametr |
| Status kodini o'zgartirish | Xato **matnini** yaxshilash |
| Standart tartibni o'zgartirish | Yangi `include` imkoniyati |

---

## 5. Filtrlash, saralash, sahifalash — bir xil shartnoma

Mijozlarga bir xil qoidalar bering:

```
GET /api/v1/posts?filter[status]=published&filter[author]=5&sort=-published_at&include=author&per_page=25
```

Qo'lda:

```php
$posts = Post::query()
    ->when($request->input('filter.status'), fn ($q, $v) => $q->where('status', $v))
    ->when($request->input('filter.author'), fn ($q, $v) => $q->where('user_id', $v))
    ->when($request->string('sort')->toString(), function ($q, $sort) {
        $direction = str_starts_with($sort, '-') ? 'desc' : 'asc';
        $column = ltrim($sort, '-');

        abort_unless(in_array($column, ['published_at', 'title'], true), 400, 'Noto\'g\'ri sort');

        $q->orderBy($column, $direction);
    })
    ->paginate(min($request->integer('per_page', 15), 100));
```

> **Xavfsizlik:** `orderBy($request->input('sort'))` deb to'g'ridan-to'g'ri yozmang — bu SQL injection va ma'lumot sizib chiqish yo'li. Ruxsat etilgan ustunlar **oq ro'yxati** bo'lsin.

Paket bilan: `spatie/laravel-query-builder` shu naqshni tayyor beradi (`allowedFilters`, `allowedSorts`, `allowedIncludes`).

---

## 6. Webhook — chiqish (siz yuborasiz)

```php
class DeliverWebhook implements ShouldQueue
{
    use Queueable;

    public int $tries = 5;
    public array $backoff = [10, 60, 300, 1800];

    public function __construct(
        public string $url,
        public array $payload,
        public string $secret,
    ) {}

    public function handle(): void
    {
        $body = json_encode($this->payload);
        $timestamp = now()->timestamp;
        $signature = hash_hmac('sha256', "{$timestamp}.{$body}", $this->secret);

        Http::withHeaders([
            'X-Signature' => "t={$timestamp},v1={$signature}",
            'Content-Type' => 'application/json',
        ])->timeout(10)->withBody($body, 'application/json')->post($this->url)->throw();
    }
}
```

Qoidalar:

1. **Imzo** (HMAC) qo'shing — qabul qiluvchi haqiqiyligini tekshirsin.
2. Timestamp qo'shing — eski so'rovni qayta yuborishga (replay) qarshi.
3. **Qayta urinish** siyosati (progressiv kutish) va oxirida "o'lik xat" jurnali.
4. Har bir yuborishni loglang (status, urinish, javob).

## 7. Webhook — kirish (sizga yuborishadi)

```php
Route::post('/webhooks/payme', PaymeWebhookController::class)
    ->withoutMiddleware(['throttle:api']);       // provayder ko'p so'rov yuborishi mumkin
```

```php
public function __invoke(Request $request)
{
    // 1) imzoni tekshirish
    $signature = hash_hmac('sha256', $request->getContent(), config('services.payme.secret'));

    abort_unless(hash_equals($signature, (string) $request->header('X-Signature')), 401);

    // 2) idempotentlik — bir xil hodisani ikki marta qayta ishlamaslik
    $eventId = $request->input('event_id');

    if (WebhookEvent::where('external_id', $eventId)->exists()) {
        return response()->noContent();
    }

    WebhookEvent::create(['external_id' => $eventId, 'payload' => $request->all()]);

    // 3) TEZ javob qaytaring, ishni navbatga qo'ying
    ProcessPaymeWebhook::dispatch($eventId);

    return response()->noContent();
}
```

> **`hash_equals` nega?** Oddiy `===` solishtiruv vaqti farq qiladi va nazariy jihatdan imzoni "taxmin qilish" imkonini beradi (timing attack). `hash_equals` doimiy vaqtda solishtiradi.

> **Tez javob bering.** Ko'p provayderlar 5-10 soniyada javob kutadi; sekin bo'lsa qayta yuboradi. Butun ishni webhook ichida bajarmang — navbatga qo'ying.

Kirish webhook'lari uchun CSRF muammo emas: ular `routes/api.php` da bo'ladi (`web` guruhi yo'q).

---

## 8. Hujjat: OpenAPI

API hujjatsiz — ishlatib bo'lmaydigan API. Uch yo'l:

| Vosita | Qanday ishlaydi |
| --- | --- |
| **dedoc/scramble** | Kodni tahlil qilib OpenAPI'ni **avtomatik** yaratadi (annotatsiya yozilmaydi) |
| **knuckleswtf/scribe** | Kod + annotatsiyadan hujjat va misollar yaratadi |
| **darkaonline/l5-swagger** | PHP atributlari/annotatsiyalari bilan qo'lda yoziladi |

```shell
composer require dedoc/scramble
# /docs/api manzilida interaktiv hujjat paydo bo'ladi
```

Hujjat foydali bo'lishi uchun **javob misollari** bo'lishi shart. Shuning uchun Form Request va Resource klasslarini to'g'ri yozish — hujjatning ham asosi.

Muqobil: Postman kolleksiyasini eksport qilib, jamoaga berish.

---

## 9. Xavfsizlik nuqtalari (API uchun)

- **Rate limiting** har bir endpointga mos: login — qat'iy, o'qish — yumshoq ([09-bob](09-marshrutlash.md)).
- **Massiv/hajm chegaralari**: `'items' => ['array', 'max:100']`, `post_max_size`, fayl `max:2048`.
- **Sahifa hajmi chegarasi**: `min($request->integer('per_page', 15), 100)`.
- **Ma'lumot sizib chiqishi**: Resource'da faqat kerakli maydonlar; `whenLoaded` bilan aloqalar.
- **Enumeratsiya hujumi**: "bunday email yo'q" o'rniga umumiy xabar; 404 larga ham rate limit qo'ying.
- **Tokenlar**: qobiliyat (ability) bilan cheklang, muddat qo'ying, `sanctum:prune-expired` rejalashtiring ([21-bob](21-autentifikatsiya.md)).
- **CORS**: aniq domenlar ([12-bob](12-sorov-va-javob.md)).
- **HTTPS majburiy**: productionda `URL::forceScheme('https')` va HSTS.

---

## 10. Kuzatuv (observability)

Har bir so'rov uchun: `request_id`, `user_id`, davomiylik, status. `Context` bilan bu loglarga avtomatik qo'shiladi ([23-bob](23-xatoliklar-va-loglar.md)).

```php
Context::add(['request_id' => $requestId, 'user_id' => auth()->id()]);
```

Ko'rish kerak bo'lgan metrikalar: p95 javob vaqti, 5xx foizi, navbat kutish vaqti, sekin so'rovlar soni. Vositalar: Pulse, Telescope (lokal), Sentry/Flare, Nightwatch.

---

## Amaliyot

1. `Idempotent` middleware'ini yozing va bir xil `Idempotency-Key` bilan ikki marta `POST` yuborib, ikkinchi javob keshdan kelganini tekshiring.
2. `cache.headers:...;etag` qo'ying va `If-None-Match` bilan 304 olganingizni ko'ring.
3. Kirish webhook endpointini yozing: HMAC imzo + `hash_equals` + navbatga qo'yish.
4. `dedoc/scramble` ni o'rnatib, `/docs/api` da hujjatni oching.
5. `sort` parametri uchun oq ro'yxat yozing va ruxsat etilmagan ustunda 400 qaytaring.
6. `X-Request-Id` middleware'ini qo'shing va uni loglarda ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/responses>
- <https://laravel.com/docs/13.x/routing#rate-limiting>
- <https://laravel.com/docs/13.x/http-client>
- <https://laravel.com/docs/13.x/context>

---

[← Oldingi: Ilg'or Eloquent](37-ilgor-eloquent-va-unumdorlik.md) · [Mundarija](README.md) · [Keyingi: Arxitektura →](39-arxitektura.md)
