# 22 — Avtorizatsiya (Gate va Policy)

[← Oldingi: Autentifikatsiya](21-autentifikatsiya.md) · [Mundarija](README.md) · [Keyingi: Xatoliklar va loglar →](23-xatoliklar-va-loglar.md)

---

## Autentifikatsiya va avtorizatsiya farqi

- **Autentifikatsiya** — "sen kimsan?" (401 Unauthorized)
- **Avtorizatsiya** — "senga bu amalga ruxsat bormi?" (403 Forbidden)

Laravel'da avtorizatsiyaning ikki vositasi bor:

| Vosita | Qachon |
| --- | --- |
| **Gate** | Modelga bog'liq bo'lmagan oddiy tekshiruv ("admin panelga kirsinmi?") |
| **Policy** | Aniq bir model atrofidagi qoidalar to'plami ("bu postni kim tahrirlaydi?") |

> **Symfony bilan solishtirish:** Gate ≈ oddiy `Security::isGranted('ROLE_ADMIN')`, Policy ≈ Voter klassi.

---

## Gate

`AppServiceProvider::boot()` da ta'riflanadi:

```php
use App\Models\User;
use Illuminate\Support\Facades\Gate;

Gate::define('admin-panel', fn (User $user) => $user->is_admin);

Gate::define('update-post', fn (User $user, Post $post) => $user->id === $post->user_id);
```

Tekshirish:

```php
if (Gate::allows('admin-panel')) { }
if (Gate::denies('update-post', $post)) { }

Gate::authorize('update-post', $post);      // ruxsat yo'q bo'lsa 403 tashlaydi

$user->can('update-post', $post);
$request->user()->cannot('update-post', $post);
```

Route'da:

```php
Route::get('/admin', ...)->middleware('can:admin-panel');
```

Barcha tekshiruvlardan oldin ishlaydigan "super admin" qoidasi:

```php
Gate::before(fn (User $user, string $ability) => $user->is_super_admin ? true : null);
```

`null` qaytarish "qaror qabul qilmadim, keyingisiga o'tsin" degani.

---

## Policy — asosiy vosita

```shell
php artisan make:policy PostPolicy --model=Post
```

```php
namespace App\Policies;

use App\Models\Post;
use App\Models\User;

class PostPolicy
{
    public function viewAny(?User $user): bool
    {
        return true;                       // ro'yxatni hamma ko'ra oladi
    }

    public function view(?User $user, Post $post): bool
    {
        return $post->status === 'published' || $user?->id === $post->user_id;
    }

    public function create(User $user): bool
    {
        return $user->hasVerifiedEmail();
    }

    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id || $user->is_admin;
    }
}
```

**Ro'yxatdan o'tkazish shart emas:** Laravel `App\Models\Post` uchun `App\Policies\PostPolicy` ni konvensiya bo'yicha o'zi topadi. Boshqacha bo'lsa:

```php
use Illuminate\Database\Eloquent\Attributes\UsePolicy;

#[UsePolicy(CustomPostPolicy::class)]
class Post extends Model {}
```

### Mehmonlar (guest)

Argument `?User $user` bo'lsa, kirmagan foydalanuvchi ham policy'ga yetib keladi (aks holda avtomatik `false`).

### Policy filtri

```php
public function before(User $user, string $ability): ?bool
{
    return $user->is_admin ? true : null;
}
```

---

## Policy'ni chaqirishning 5 usuli

```php
// 1) Kontrollerda
$this->authorize('update', $post);                  // Controller bazasidan
Gate::authorize('update', $post);

// 2) Foydalanuvchi orqali
if ($request->user()->can('update', $post)) { }

// 3) Route middleware
Route::put('/posts/{post}', ...)->middleware('can:update,post');

// 4) Kontroller atributi (Laravel 13)
use Illuminate\Routing\Attributes\Controllers\Authorize;

#[Authorize('update', 'post')]
public function update(UpdatePostRequest $request, Post $post) { }

// 5) Form Request ichida
public function authorize(): bool
{
    return $this->user()->can('update', $this->post);
}
```

`create` kabi modelsiz amallarda klass nomi uzatiladi:

```php
$this->authorize('create', Post::class);

#[Authorize('create', Post::class)]
public function store(StorePostRequest $request) { }
```

Resource kontrollerning hamma metodini bir yo'la bog'lash:

```php
// kontroller konstruktorida yoki route ta'rifida
Route::apiResource('posts', PostController::class)->middleware('can:...');
// yoki kontrollerda:
$this->authorizeResource(Post::class, 'post');
```

---

## Javob va xabar bilan qaytarish

```php
use Illuminate\Auth\Access\Response;

public function update(User $user, Post $post): Response
{
    return $user->id === $post->user_id
        ? Response::allow()
        : Response::deny('Siz faqat o\'z postingizni tahrirlay olasiz.', 403);
}
```

API'da bu xabar JSON'da ko'rinadi:

```json
{ "message": "Siz faqat o'z postingizni tahrirlay olasiz." }
```

---

## Rollar va ruxsatlar

Laravel'da tayyor "rol/permission" tizimi yo'q — chunki har loyihada talab har xil. Ikki yo'l bor:

1. **Oddiy** — `users.role` ustuni (`enum`) va Gate/Policy ichida tekshirish:

```php
enum UserRole: string
{
    case Admin = 'admin';
    case Editor = 'editor';
    case Viewer = 'viewer';
}

Gate::define('manage-users', fn (User $user) => $user->role === UserRole::Admin);
```

2. **Murakkab** (ko'p rol, dinamik ruxsatlar) — `spatie/laravel-permission` paketi.

---

## API'da avtorizatsiya oqimi

Tipik tartib:

1. `auth:sanctum` — token to'g'rimi (yo'q bo'lsa **401**).
2. `ability:post:create` — token shu amalga ruxsat berganmi (yo'q bo'lsa **403**).
3. Form Request `authorize()` yoki `#[Authorize]` — foydalanuvchi shu **aniq resursga** ruxsatga egami (yo'q bo'lsa **403**).
4. `rules()` — ma'lumot to'g'rimi (yo'q bo'lsa **422**).

Shu tartib API'da xatoliklarning to'g'ri kod bilan qaytishini ta'minlaydi.

---

## Ro'yxatlarni filtrlash

Policy bitta obyekt uchun javob beradi. Ro'yxat uchun **so'rovni** cheklang:

```php
public function index(Request $request)
{
    $posts = Post::query()
        ->when(! $request->user()?->is_admin, fn ($q) => $q->where('status', 'published'))
        ->paginate(15);

    return PostResource::collection($posts);
}
```

Yoki global scope ishlating ([16-bob](16-eloquent-asoslari.md)).

---

## Amaliyot

1. `php artisan make:policy PostPolicy --model=Post` yarating va `update`/`delete` qoidalarini yozing.
2. `PostController::update()` ga `#[Authorize('update', 'post')]` qo'ying.
3. Ikkita foydalanuvchi yarating; birining posti'ni ikkinchisi tahrirlashga urinib, 403 olganini tekshiring.
4. `Response::deny('...')` bilan maxsus xabar qaytaring va JSON'da ko'ring.
5. `Gate::before()` bilan admin uchun hamma narsani ochib qo'ying.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/authorization>
- <https://laravel.com/docs/13.x/controllers#authorization-attributes>

---

[← Oldingi: Autentifikatsiya](21-autentifikatsiya.md) · [Mundarija](README.md) · [Keyingi: Xatoliklar va loglar →](23-xatoliklar-va-loglar.md)
