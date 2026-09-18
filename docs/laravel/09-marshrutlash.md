# 09 — Marshrutlash (Routing)

[← Oldingi: Konfiguratsiya](08-konfiguratsiya-va-muhit.md) · [Mundarija](README.md) · [Keyingi: Middleware →](10-middleware.md)

---

## Route fayllari

| Fayl | Prefiks | Middleware guruhi | Nima uchun |
| --- | --- | --- | --- |
| `routes/web.php` | — | `web` | Brauzer sahifalari (sessiya, CSRF) |
| `routes/api.php` | `/api` | `api` | Stateless API |
| `routes/console.php` | — | — | Closure Artisan buyruqlari, scheduler |

`api.php` yo'q bo'lsa:

```shell
php artisan install:api
```

Prefiksni o'zgartirish `bootstrap/app.php` da:

```php
->withRouting(
    web: __DIR__.'/../routes/web.php',
    api: __DIR__.'/../routes/api.php',
    apiPrefix: 'api/v1',
    commands: __DIR__.'/../routes/console.php',
    health: '/up',
)
```

> **Symfony bilan solishtirish:** Symfony'da route'lar odatda atributlar bilan kontroller ustida yoziladi. Laravel'da esa **alohida faylda, markazlashgan ro'yxat** sifatida. Foydasi: `php artisan route:list` bilan butun API'ni bir joyda ko'rasiz; kamchiligi: route va kontroller ikki faylda.

---

## Asosiy sintaksis

```php
use Illuminate\Support\Facades\Route;

Route::get('/posts', fn () => Post::all());
Route::post('/posts', [PostController::class, 'store']);
Route::put('/posts/{post}', [PostController::class, 'update']);
Route::patch('/posts/{post}', [PostController::class, 'update']);
Route::delete('/posts/{post}', [PostController::class, 'destroy']);

Route::match(['get', 'post'], '/qidiruv', ...);
Route::any('/hammasi', ...);

// Faqat bitta metodli kontroller (invokable)
Route::get('/hisobot', GenerateReport::class);

// Boshqa manzilga yo'naltirish
Route::redirect('/eski', '/yangi', 301);
Route::permanentRedirect('/eski', '/yangi');

// Faqat shablon ko'rsatish
Route::view('/aloqa', 'contact', ['phone' => '+998...']);
```

---

## Parametrlar

```php
Route::get('/posts/{post}', fn (string $post) => $post);            // majburiy
Route::get('/users/{name?}', fn (?string $name = null) => $name);   // ixtiyoriy

// Cheklovlar (regex)
Route::get('/users/{id}', ...)->whereNumber('id');
Route::get('/users/{name}', ...)->whereAlpha('name');
Route::get('/posts/{slug}', ...)->where('slug', '[a-z0-9\-]+');
Route::get('/kategoriya/{type}', ...)->whereIn('type', ['yangilik', 'maqola']);
```

Global cheklov (`AppServiceProvider::boot()` da):

```php
Route::pattern('id', '[0-9]+');
```

---

## Nomlangan route'lar

```php
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');
```

```php
route('posts.show', ['post' => 12]);               // /posts/12
route('posts.show', ['post' => 12, 'ref' => 'tg']); // /posts/12?ref=tg
redirect()->route('posts.show', $post);
```

**Nega nomlangan route ishlatish kerak?** Chunki URL manzili keyin o'zgarishi mumkin (`/posts` → `/maqolalar`). Nom o'zgarmasa, kodning qolgan qismiga tegmaysiz. Symfony'dagi `path('post_show')` bilan aynan bir xil g'oya.

---

## Guruhlar

```php
Route::prefix('admin')
    ->name('admin.')
    ->middleware(['auth', 'can:admin-panel'])
    ->group(function () {
        Route::get('/users', [UserController::class, 'index'])->name('users.index');
        // URL: /admin/users, nomi: admin.users.index
    });

Route::controller(PostController::class)->group(function () {
    Route::get('/posts', 'index');
    Route::post('/posts', 'store');
});

Route::domain('{account}.example.com')->group(function () {
    Route::get('/user/{id}', fn (string $account, string $id) => ...);
});
```

Guruhlarni ichma-ich yozish mumkin — prefiks, nom va middleware'lar birlashadi.

---

## Route model binding — "sehr"ning tushuntirilishi

### Implicit binding

```php
Route::get('/posts/{post}', function (App\Models\Post $post) {
    return $post;   // baza'dan topilgan model
});
```

Nima bo'ldi: parametr nomi (`{post}`) va argument nomi (`$post`) **mos keladi**, argument tipi esa Eloquent model. Shunda `SubstituteBindings` middleware'i `Post::findOrFail($value)` ni bajaradi. Model topilmasa — avtomatik **404**.

Boshqa ustun bo'yicha izlash:

```php
Route::get('/posts/{post:slug}', fn (Post $post) => $post);
```

Yoki modelning o'zida standart kalitni o'zgartirish:

```php
class Post extends Model
{
    public function getRouteKeyName(): string
    {
        return 'slug';
    }
}
```

Ichma-ich (nested) bog'lash — `scopeBindings()` bilan "bu izoh aynan shu postnikimi?" tekshiriladi:

```php
Route::scopeBindings()->group(function () {
    Route::get('/posts/{post}/comments/{comment}', fn (Post $post, Comment $comment) => $comment);
});
```

### Explicit binding

```php
// AppServiceProvider::boot()
Route::model('post', Post::class);

Route::bind('post', function (string $value) {
    return Post::where('slug', $value)->published()->firstOrFail();
});
```

> **Symfony bilan solishtirish:** bu `ParamConverter` / `MapEntity` ning analogi, lekin standart ravishda yoqilgan.

---

## Resource route'lar (CRUD)

```php
Route::resource('posts', PostController::class);
Route::apiResource('posts', PostController::class);          // create/edit formasi yo'q
Route::apiResources(['posts' => PostController::class, 'tags' => TagController::class]);
```

`apiResource` yaratadigan route'lar:

| Metod | URI | Action | Route nomi |
| --- | --- | --- | --- |
| GET | `/posts` | index | posts.index |
| POST | `/posts` | store | posts.store |
| GET | `/posts/{post}` | show | posts.show |
| PUT/PATCH | `/posts/{post}` | update | posts.update |
| DELETE | `/posts/{post}` | destroy | posts.destroy |

Cheklash:

```php
Route::apiResource('posts', PostController::class)->only(['index', 'show']);
Route::apiResource('posts', PostController::class)->except(['destroy']);
Route::apiResource('posts.comments', CommentController::class);  // /posts/{post}/comments
```

Kontrollerni tayyor holda yaratish:

```shell
php artisan make:controller PostController --api --model=Post --requests
```

---

## Rate limiting (so'rovlar chegarasi)

`AppServiceProvider::boot()` da cheklov ta'riflanadi:

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});

RateLimiter::for('login', function (Request $request) {
    return [
        Limit::perMinute(500),
        Limit::perMinute(3)->by($request->input('email')),
    ];
});
```

Route'ga biriktirish:

```php
Route::middleware('throttle:api')->group(/* ... */);
Route::post('/login', ...)->middleware('throttle:login');
Route::post('/upload', ...)->middleware('throttle:10,1');   // 1 daqiqada 10 marta
```

Chegaradan oshsa Laravel avtomatik **429** qaytaradi va `Retry-After` sarlavhasini qo'yadi.

---

## Fallback va CSRF

```php
Route::fallback(fn () => response()->json(['message' => 'Topilmadi'], 404));
```

`web` guruhida POST/PUT/DELETE so'rovlar CSRF tokenini talab qiladi (`PreventRequestForgery` middleware). Blade formada:

```blade
<form method="POST" action="/posts">
    @csrf
    @method('PUT')
</form>
```

API route'larida CSRF yo'q — u yerda token/Sanctum ishlatiladi ([21-bob](21-autentifikatsiya.md)).

---

## Route'larni ko'rish va keshlash

```shell
php artisan route:list
php artisan route:list --path=api --method=GET
php artisan route:list --except-vendor

php artisan route:cache     # production uchun
php artisan route:clear
```

> **Diqqat:** `route:cache` ishlatilsa, route fayllarida **closure** bo'lmasligi kerak — faqat kontroller klasslari. Closure'ni serializatsiya qilib bo'lmaydi.

---

## Amaliyot

1. `routes/api.php` ni yarating (`php artisan install:api`) va quyidagini qo'shing:

```php
Route::get('/status', fn () => ['ok' => true]);
```

`curl http://127.0.0.1:8000/api/status` bilan tekshiring — prefiks `/api` avtomatik qo'shilganini ko'ring.

2. `Route::apiResource('posts', PostController::class)` yozing va `php artisan route:list --path=posts` bilan 5 ta route paydo bo'lganini ko'ring.
3. Model binding'ni sinang: mavjud bo'lmagan ID bilan so'rov yuboring va 404 kelishini tekshiring.
4. `throttle:3,1` qo'yilgan route yarating va 4-marta so'rov yuborib 429 olganingizni ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/routing>
- <https://laravel.com/docs/13.x/controllers#resource-controllers>

---

[← Oldingi: Konfiguratsiya](08-konfiguratsiya-va-muhit.md) · [Mundarija](README.md) · [Keyingi: Middleware →](10-middleware.md)
