# 11 — Kontrollerlar

[← Oldingi: Middleware](10-middleware.md) · [Mundarija](README.md) · [Keyingi: So'rov va javob →](12-sorov-va-javob.md)

---

## Oddiy kontroller

```shell
php artisan make:controller PostController
```

```php
namespace App\Http\Controllers;

use App\Models\Post;
use Illuminate\Http\JsonResponse;

class PostController extends Controller
{
    public function show(Post $post): JsonResponse
    {
        return response()->json($post);
    }
}
```

```php
Route::get('/posts/{post}', [PostController::class, 'show']);
```

Kontroller **majburiy ravishda** `Controller` bazasidan meros olishi shart emas — oddiy klass ham ishlaydi. Baza klass faqat umumiy metodlar uchun qulaylik.

> **Symfony bilan solishtirish:** g'oya bir xil. Farqi — route atributlar bilan emas, `routes/` faylida yoziladi ([09-bob](09-marshrutlash.md)).

---

## Resource kontroller (CRUD)

```shell
php artisan make:controller PostController --api --model=Post --requests
```

Bayroqlar:

| Bayroq | Nima qiladi |
| --- | --- |
| `--api` | `create` va `edit` metodlarisiz (API uchun) |
| `--resource` | To'liq 7 metodli CRUD |
| `--model=Post` | Metodlarga `Post $post` tipini qo'yadi |
| `--requests` | `StorePostRequest` va `UpdatePostRequest` ni ham yaratadi |
| `--invokable` | Bitta `__invoke()` metodli kontroller |

Natija:

```php
class PostController extends Controller
{
    public function index() {}
    public function store(StorePostRequest $request) {}
    public function show(Post $post) {}
    public function update(UpdatePostRequest $request, Post $post) {}
    public function destroy(Post $post) {}
}
```

To'liq misol:

```php
namespace App\Http\Controllers;

use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Http\Resources\PostResource;
use App\Models\Post;
use Illuminate\Http\Response;

class PostController extends Controller
{
    public function index()
    {
        return PostResource::collection(
            Post::with('author')->latest()->paginate(15)
        );
    }

    public function store(StorePostRequest $request)
    {
        $post = $request->user()->posts()->create($request->validated());

        return PostResource::make($post)
            ->response()
            ->setStatusCode(Response::HTTP_CREATED);   // 201
    }

    public function show(Post $post)
    {
        return PostResource::make($post->load('author'));
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        $post->update($request->validated());

        return PostResource::make($post);
    }

    public function destroy(Post $post)
    {
        $post->delete();

        return response()->noContent();   // 204
    }
}
```

---

## Invokable kontroller (bitta amal)

```shell
php artisan make:controller GenerateMonthlyReport --invokable
```

```php
class GenerateMonthlyReport extends Controller
{
    public function __invoke(Request $request)
    {
        // ...
    }
}
```

```php
Route::post('/hisobot', GenerateMonthlyReport::class);
```

**Nega foydali?** Murakkab amallarni "fat controller"ga tiqishtirmaslik uchun. Har bir murakkab amal — alohida klass. Symfony'dagi "invokable controller" bilan bir xil.

---

## Dependency injection

Konstruktorda:

```php
class PostController extends Controller
{
    public function __construct(
        private PostRepository $posts,
    ) {}
}
```

Metodda:

```php
public function store(StorePostRequest $request, PostService $service)
{
    // Request ham, PostService ham konteynerdan keladi
}
```

Route parametrlari **argumentlardan keyin** keladi:

```php
// Route::put('/posts/{post}', ...)
public function update(Request $request, Post $post) { }
```

---

## Kontrollerga middleware biriktirish

Uch xil usul:

```php
// 1) Route ta'rifida (eng aniq va o'qiladigan)
Route::apiResource('posts', PostController::class)->middleware('auth:sanctum');
```

```php
// 2) Kontroller ichida statik metod orqali
use Illuminate\Routing\Controllers\HasMiddleware;
use Illuminate\Routing\Controllers\Middleware;

class PostController extends Controller implements HasMiddleware
{
    public static function middleware(): array
    {
        return [
            'auth:sanctum',
            new Middleware('throttle:60,1', only: ['store', 'update']),
            new Middleware('can:admin', except: ['index', 'show']),
        ];
    }
}
```

```php
// 3) Atributlar bilan (Laravel 13)
use Illuminate\Routing\Attributes\Controllers\Authorize;
use Illuminate\Routing\Attributes\Controllers\Middleware;
use Illuminate\Routing\Attributes\Controllers\WithoutMiddleware;

#[Middleware('auth:sanctum')]
class CommentController
{
    #[Middleware('throttle:10,1')]
    public function store(Post $post) { }

    #[Authorize('delete', 'comment')]
    public function destroy(Comment $comment) { }

    #[WithoutMiddleware('auth:sanctum')]
    public function index() { }
}
```

`#[Authorize]` — `can` middleware'ining qisqa ko'rinishi: birinchi argument — ruxsat nomi, ikkinchisi — model klassi yoki route parametri nomi ([22-bob](22-avtorizatsiya.md)).

---

## Yagona javob qaytarish (single action) uslubi

API kontrollerlari ideal holda **yupqa** bo'lishi kerak: validatsiya → amal → javob. Og'ir mantiq alohida klassga chiqadi:

```php
public function store(StoreOrderRequest $request, PlaceOrder $placeOrder)
{
    $order = $placeOrder->handle(
        user: $request->user(),
        items: $request->validated('items'),
    );

    return OrderResource::make($order);
}
```

`PlaceOrder` — oddiy klass (`php artisan make:class Actions/PlaceOrder`). Bu Symfony'dagi "Service" qatlamining aynan o'zi.

---

## Tipik xatolar

| Xato | To'g'ri yechim |
| --- | --- |
| Kontrollerda 200 qator biznes-mantiq | Action/Service klassiga ko'chiring |
| Kontrollerda `Validator::make(...)` | `FormRequest` ishlating ([13-bob](13-validatsiya.md)) |
| Modeldan to'g'ridan-to'g'ri JSON qaytarish | `JsonResource` ishlating ([20-bob](20-api-resurslar.md)) |
| Har bir amalda `if ($user->id !== $post->user_id)` | Policy ishlating ([22-bob](22-avtorizatsiya.md)) |
| Ro'yxatda `Post::all()` | `paginate()` ishlating |

---

## Amaliyot

1. `php artisan make:controller PostController --api --model=Post --requests` ni ishlating va yaratilgan fayllarni ko'ring.
2. `routes/api.php` da `Route::apiResource('posts', PostController::class);` yozing, `php artisan route:list --path=posts` bilan tekshiring.
3. `index()` metodida `Post::paginate(15)` qaytaring va `curl "http://127.0.0.1:8000/api/posts?page=2"` bilan paginatsiya JSON'ini ko'ring.
4. `HasMiddleware` interfeysi orqali `store` va `update` ga `throttle:10,1` qo'ying.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/controllers>
- <https://laravel.com/docs/13.x/routing#route-model-binding>

---

[← Oldingi: Middleware](10-middleware.md) · [Mundarija](README.md) · [Keyingi: So'rov va javob →](12-sorov-va-javob.md)
