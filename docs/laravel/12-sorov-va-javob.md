# 12 — So'rov va javob (Request / Response)

[← Oldingi: Kontrollerlar](11-kontrollerlar.md) · [Mundarija](README.md) · [Keyingi: Validatsiya →](13-validatsiya.md)

---

## `Illuminate\Http\Request`

Bu klass `Symfony\Component\HttpFoundation\Request` dan meros oladi — ya'ni Symfony'dan bilgan hamma narsangiz (`$request->headers`, `$request->query`, `$request->files`) shu yerda ham bor, ustiga Laravel qulayliklari qo'shilgan.

Olish yo'llari:

```php
public function store(Request $request) { }   // tip ko'rsatish (tavsiya etiladi)
$request = request();                         // helper
```

---

## Ma'lumot o'qish

```php
$request->input('title');                 // query + body (JSON ham)
$request->input('title', 'Sarlavhasiz');  // standart qiymat
$request->input('user.name');             // ichma-ich (JSON/massiv) — nuqtali sintaksis
$request->query('page', 1);               // faqat query string
$request->post('title');                  // faqat body
$request->title;                          // dinamik xossa (input bilan bir xil)

$request->all();                          // hammasi
$request->only(['title', 'body']);
$request->except(['password']);
$request->collect('items');               // Collection qaytaradi
```

Tipga ishonch bilan o'qish (validatsiyadan keyin ham foydali):

```php
$request->string('title')->trim();        // Stringable
$request->integer('page');
$request->boolean('published');            // "1", "true", "on", "yes" → true
$request->float('price');
$request->date('published_at');            // Carbon
$request->enum('status', PostStatus::class);
$request->array('tags');
```

Mavjudligini tekshirish:

```php
$request->has('title');           // kalit bor
$request->hasAny(['a', 'b']);
$request->filled('title');        // bor VA bo'sh emas
$request->missing('title');
$request->whenFilled('title', fn ($value) => /* ... */);
```

> **`has` va `filled` farqi:** `?title=` bo'lsa `has('title')` → `true`, `filled('title')` → `false`. Bu farqni bilmaslik tipik bug manbai.

---

## So'rov haqida ma'lumot

```php
$request->path();          // "posts/12"
$request->url();           // "https://sayt.uz/posts/12"
$request->fullUrl();       // query bilan
$request->method();        // "POST"
$request->isMethod('post');
$request->is('admin/*');           // path shablon bo'yicha
$request->routeIs('posts.*');      // route nomi bo'yicha
$request->ip();
$request->userAgent();
$request->header('X-Api-Token');
$request->bearerToken();           // Authorization: Bearer ...
$request->expectsJson();
$request->wantsJson();
$request->user();                  // autentifikatsiyadan o'tgan foydalanuvchi
$request->route('post');           // route parametri
```

---

## Fayllar

```php
if ($request->hasFile('avatar') && $request->file('avatar')->isValid()) {
    $file = $request->file('avatar');

    $file->getClientOriginalName();
    $file->extension();
    $file->getSize();

    // saqlash
    $path = $file->store('avatars');                       // storage/app/private/avatars/...
    $path = $file->store('avatars', 'public');             // public disk
    $path = $file->storeAs('avatars', "user-{$id}.jpg", 'public');
}
```

Fayl validatsiyasi [13-bobda](13-validatsiya.md), disklar [27-bobda](27-kesh-va-fayllar.md).

---

## Javoblar

```php
return 'Salom';                                    // matn
return ['ok' => true];                             // JSON
return $post;                                      // model → JSON
return response('Salom', 200)->header('X-Foo', 'bar');
return response()->json(['message' => 'Yaratildi'], 201);
return response()->noContent();                    // 204
return response()->json($data)
    ->withHeaders(['X-Total' => 42]);
```

Fayl javoblari:

```php
return response()->download($pathToFile, 'hisobot.pdf');
return response()->file($pathToImage);            // brauzerda ko'rsatish
return response()->streamDownload(function () use ($csv) { echo $csv; }, 'export.csv');
```

Yo'naltirish (web uchun):

```php
return redirect('/posts');
return redirect()->route('posts.show', $post);
return redirect()->back()->withInput();
return redirect()->intended('/dashboard');         // login'dan keyin
return back()->with('status', 'Saqlandi');         // flash xabar
```

---

## HTTP status kodlari — API uchun to'g'ri tanlash

| Kod | Qachon |
| --- | --- |
| 200 OK | Muvaffaqiyatli o'qish/yangilash |
| 201 Created | Yangi resurs yaratildi |
| 202 Accepted | Qabul qilindi, fonda bajariladi (queue) |
| 204 No Content | O'chirildi / javob tanasi yo'q |
| 400 Bad Request | So'rov shakli noto'g'ri |
| 401 Unauthorized | Autentifikatsiya yo'q / token noto'g'ri |
| 403 Forbidden | Kirgan, lekin ruxsat yo'q |
| 404 Not Found | Resurs topilmadi |
| 409 Conflict | Holat qarama-qarshiligi (masalan takroriy amal) |
| 422 Unprocessable Entity | **Validatsiya xatosi** (Laravel standarti) |
| 429 Too Many Requests | Rate limit |
| 500 Internal Server Error | Kutilmagan xato |

Laravel'da:

```php
use Symfony\Component\HttpFoundation\Response as HttpStatus;

return response()->json($data, HttpStatus::HTTP_CREATED);
abort(404, 'Post topilmadi');
abort_if($post->user_id !== $request->user()->id, 403);
abort_unless($user->is_admin, 403);
```

---

## Cookie va sessiya (faqat `web`)

```php
return response('ok')->cookie('name', 'value', minutes: 60);
$value = $request->cookie('name');

session(['key' => 'value']);
$value = session('key', 'default');
$request->session()->forget('key');
$request->session()->flash('status', 'Saqlandi');   // faqat keyingi so'rovda
```

Cookie'lar `EncryptCookies` middleware'i tufayli shifrlangan holda yuboriladi.

---

## CORS

CORS `HandleCors` middleware'i orqali global ishlaydi. Sozlash uchun config faylini chiqarish kerak:

```shell
php artisan config:publish cors
```

```php
// config/cors.php
return [
    'paths' => ['api/*'],
    'allowed_methods' => ['*'],
    'allowed_origins' => ['https://frontend.uz'],
    'allowed_headers' => ['*'],
    'supports_credentials' => false,   // Sanctum SPA uchun true
];
```

> **Nega `allowed_origins` ga `*` qo'yish yomon?** Chunki cookie bilan ishlaydigan SPA'da `supports_credentials: true` bilan `*` birga ishlamaydi va bu xavfsizlik nuqtai nazaridan ham noto'g'ri. Aniq domenlarni yozing.

---

## Javob makrolari (takrorlanuvchi format)

```php
// AppServiceProvider::boot()
Response::macro('success', function (mixed $data, int $status = 200) {
    return response()->json(['data' => $data, 'error' => null], $status);
});
```

```php
return response()->success($post, 201);
```

---

## Amaliyot

1. Quyidagi route'ni yozing va turli query'lar bilan sinab ko'ring:

```php
Route::get('/echo', function (Illuminate\Http\Request $request) {
    return [
        'page' => $request->integer('page'),
        'active' => $request->boolean('active'),
        'has_q' => $request->has('q'),
        'filled_q' => $request->filled('q'),
    ];
});
```

`/echo?page=2&active=yes&q=` bilan chaqiring va `has_q` bilan `filled_q` farqini ko'ring.

2. 201 status va `Location` sarlavhasi bilan javob qaytaradigan route yozing.
3. `response()->noContent()` qaytaring va `curl -i` bilan 204 ekanini tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/requests>
- <https://laravel.com/docs/13.x/responses>

---

[← Oldingi: Kontrollerlar](11-kontrollerlar.md) · [Mundarija](README.md) · [Keyingi: Validatsiya →](13-validatsiya.md)
