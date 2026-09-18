# 31 — Amaliy loyiha: to'liq REST API

[← Oldingi: Blade va frontend](30-blade-va-frontend.md) · [Mundarija](README.md) · [Keyingi: Optimallashtirish va deploy →](32-optimallashtirish-va-deploy.md)

---

## Nima quramiz

Blog API: foydalanuvchilar ro'yxatdan o'tadi, token oladi, post yozadi, tahrirlaydi va o'chiradi. Postlarni hamma ko'radi, lekin faqat egasi o'zgartira oladi.

Ishlatiladigan mavzular: routing, Form Request, Eloquent, aloqalar, policy, Sanctum, API resource, navbat, testlar — ya'ni 1–29 boblarning barchasi.

Endpointlar:

| Metod | URI | Kim | Tavsif |
| --- | --- | --- | --- |
| POST | `/api/register` | hamma | Ro'yxatdan o'tish |
| POST | `/api/login` | hamma | Token olish |
| POST | `/api/logout` | token | Tokenni bekor qilish |
| GET | `/api/posts` | hamma | Chop etilgan postlar (paginatsiya, qidiruv) |
| POST | `/api/posts` | token | Post yaratish |
| GET | `/api/posts/{post}` | hamma | Bitta post |
| PUT | `/api/posts/{post}` | egasi | Tahrirlash |
| DELETE | `/api/posts/{post}` | egasi | O'chirish |

---

## 0-qadam: tayyorgarlik

```shell
php artisan install:api
php artisan migrate
```

`app/Models/User.php` ga trait qo'shing:

```php
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens, HasFactory, Notifiable;

    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}
```

`AppServiceProvider::boot()` ga qat'iy rejimni qo'shing:

```php
use Illuminate\Database\Eloquent\Model;

public function boot(): void
{
    Model::shouldBeStrict(! $this->app->isProduction());
}
```

---

## 1-qadam: Model va migratsiya

```shell
php artisan make:model Post -mf
```

```php
// database/migrations/xxxx_create_posts_table.php
public function up(): void
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->foreignId('user_id')->constrained()->cascadeOnDelete();
        $table->string('title');
        $table->string('slug')->unique();
        $table->text('body');
        $table->string('status')->default('draft')->index();
        $table->timestamp('published_at')->nullable();
        $table->timestamps();
    });
}
```

```php
// app/Models/Post.php
namespace App\Models;

use Illuminate\Database\Eloquent\Attributes\{Fillable, Scope, UsePolicy, UseResource};
use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;

#[Fillable(['title', 'slug', 'body', 'status', 'published_at'])]
#[UsePolicy(\App\Policies\PostPolicy::class)]
#[UseResource(\App\Http\Resources\PostResource::class)]
class Post extends Model
{
    use HasFactory;

    protected function casts(): array
    {
        return ['published_at' => 'datetime'];
    }

    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }

    #[Scope]
    protected function published(Builder $query): void
    {
        $query->where('status', 'published')->whereNotNull('published_at');
    }

    #[Scope]
    protected function search(Builder $query, ?string $term): void
    {
        $query->when($term, fn ($q) => $q->where('title', 'like', "%{$term}%"));
    }

    public function getRouteKeyName(): string
    {
        return 'slug';
    }
}
```

```shell
php artisan migrate
```

---

## 2-qadam: Factory

```php
// database/factories/PostFactory.php
public function definition(): array
{
    $title = fake()->sentence();

    return [
        'user_id' => User::factory(),
        'title' => $title,
        'slug' => str($title)->slug()->value(),
        'body' => fake()->paragraphs(3, true),
        'status' => 'draft',
        'published_at' => null,
    ];
}

public function published(): static
{
    return $this->state(['status' => 'published', 'published_at' => now()]);
}
```

---

## 3-qadam: Form Request'lar

```shell
php artisan make:request StorePostRequest
php artisan make:request UpdatePostRequest
```

```php
// app/Http/Requests/StorePostRequest.php
namespace App\Http\Requests;

use App\Models\Post;
use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Post::class);
    }

    /** @return array<string, array<int, mixed>> */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'min:5', 'max:255'],
            'body' => ['required', 'string', 'min:20'],
            'status' => ['sometimes', Rule::in(['draft', 'published'])],
        ];
    }

    protected function prepareForValidation(): void
    {
        if ($this->filled('title')) {
            $this->merge(['slug' => str($this->title)->slug()->value()]);
        }
    }
}
```

```php
// app/Http/Requests/UpdatePostRequest.php
class UpdatePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('update', $this->route('post'));
    }

    /** @return array<string, array<int, mixed>> */
    public function rules(): array
    {
        return [
            'title' => ['sometimes', 'required', 'string', 'min:5', 'max:255'],
            'body' => ['sometimes', 'required', 'string', 'min:20'],
            'status' => ['sometimes', Rule::in(['draft', 'published'])],
        ];
    }
}
```

---

## 4-qadam: Policy

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
        return true;
    }

    public function view(?User $user, Post $post): bool
    {
        return $post->status === 'published' || $user?->id === $post->user_id;
    }

    public function create(User $user): bool
    {
        return true;
    }

    public function update(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post): bool
    {
        return $user->id === $post->user_id;
    }
}
```

---

## 5-qadam: Resource'lar

```shell
php artisan make:resource PostResource
php artisan make:resource UserResource
```

```php
// app/Http/Resources/PostResource.php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    /** @return array<string, mixed> */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'excerpt' => str($this->body)->limit(150)->value(),
            'body' => $this->when($request->routeIs('posts.show'), $this->body),
            'status' => $this->status,
            'published_at' => $this->published_at?->toIso8601String(),
            'author' => UserResource::make($this->whenLoaded('author')),
            'links' => [
                'self' => route('posts.show', $this->resource),
            ],
        ];
    }
}
```

```php
// app/Http/Resources/UserResource.php
public function toArray(Request $request): array
{
    return [
        'id' => $this->id,
        'name' => $this->name,
        'email' => $this->when($request->user()?->id === $this->id, $this->email),
    ];
}
```

---

## 6-qadam: Kontrollerlar

```shell
php artisan make:controller Api/PostController --api --model=Post
php artisan make:controller Api/AuthController
```

```php
// app/Http/Controllers/Api/PostController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use App\Http\Resources\PostResource;
use App\Models\Post;
use Illuminate\Http\Request;
use Illuminate\Routing\Attributes\Controllers\Authorize;

class PostController extends Controller
{
    public function index(Request $request)
    {
        $perPage = min($request->integer('per_page', 15), 100);

        $posts = Post::query()
            ->published()
            ->search($request->string('q')->toString() ?: null)
            ->with('author')
            ->latest('published_at')
            ->paginate($perPage)
            ->withQueryString();

        return PostResource::collection($posts);
    }

    public function store(StorePostRequest $request)
    {
        $post = $request->user()->posts()->create([
            ...$request->validated(),
            'slug' => $request->input('slug'),
            'published_at' => $request->input('status') === 'published' ? now() : null,
        ]);

        return PostResource::make($post->load('author'))
            ->response()
            ->setStatusCode(201)
            ->header('Location', route('posts.show', $post));
    }

    #[Authorize('view', 'post')]
    public function show(Post $post)
    {
        return PostResource::make($post->load('author'));
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        $data = $request->validated();

        if (($data['status'] ?? null) === 'published' && $post->published_at === null) {
            $data['published_at'] = now();
        }

        $post->update($data);

        return PostResource::make($post->load('author'));
    }

    #[Authorize('delete', 'post')]
    public function destroy(Post $post)
    {
        $post->delete();

        return response()->noContent();
    }
}
```

```php
// app/Http/Controllers/Api/AuthController.php
namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Http\Resources\UserResource;
use App\Models\User;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use Illuminate\Validation\Rules\Password;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function register(Request $request)
    {
        $data = $request->validate([
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'email', 'max:255', 'unique:users,email'],
            'password' => ['required', 'confirmed', Password::min(8)],
        ]);

        $user = User::create($data);

        return response()->json([
            'user' => UserResource::make($user),
            'token' => $user->createToken('api')->plainTextToken,
        ], 201);
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => ['required', 'email'],
            'password' => ['required', 'string'],
            'device_name' => ['sometimes', 'string', 'max:255'],
        ]);

        $user = User::where('email', $credentials['email'])->first();

        if (! $user || ! Hash::check($credentials['password'], $user->password)) {
            throw ValidationException::withMessages([
                'email' => ['Kiritilgan ma\'lumotlar mos kelmadi.'],
            ]);
        }

        return response()->json([
            'user' => UserResource::make($user),
            'token' => $user->createToken($credentials['device_name'] ?? 'api')->plainTextToken,
        ]);
    }

    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->noContent();
    }
}
```

---

## 7-qadam: Route'lar

```php
// routes/api.php
use App\Http\Controllers\Api\AuthController;
use App\Http\Controllers\Api\PostController;
use Illuminate\Support\Facades\Route;

Route::post('/register', [AuthController::class, 'register'])->middleware('throttle:6,1');
Route::post('/login', [AuthController::class, 'login'])->middleware('throttle:login');

Route::get('/posts', [PostController::class, 'index'])->name('posts.index');
Route::get('/posts/{post}', [PostController::class, 'show'])->name('posts.show');

Route::middleware('auth:sanctum')->group(function () {
    Route::post('/logout', [AuthController::class, 'logout']);
    Route::get('/me', fn (Request $request) => UserResource::make($request->user()));

    Route::post('/posts', [PostController::class, 'store'])->name('posts.store');
    Route::put('/posts/{post}', [PostController::class, 'update'])->name('posts.update');
    Route::delete('/posts/{post}', [PostController::class, 'destroy'])->name('posts.destroy');
});
```

`AppServiceProvider::boot()` da `login` uchun limiter:

```php
RateLimiter::for('login', fn (Request $request) => [
    Limit::perMinute(30)->by($request->ip()),
    Limit::perMinute(5)->by('email:'.$request->input('email')),
]);
```

Tekshirish:

```shell
php artisan route:list --path=api
```

---

## 8-qadam: Sinab ko'rish

```shell
php artisan dev
```

```shell
# ro'yxatdan o'tish
curl -s -X POST http://127.0.0.1:8000/api/register \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -d '{"name":"Ali","email":"ali@example.uz","password":"parol12345","password_confirmation":"parol12345"}'

# token bilan post yaratish
TOKEN=...
curl -s -X POST http://127.0.0.1:8000/api/posts \
  -H "Accept: application/json" -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title":"Birinchi post","body":"Bu postning matni, kamida yigirma belgidan iborat.","status":"published"}'

# ro'yxat
curl -s "http://127.0.0.1:8000/api/posts?q=birinchi&per_page=5" -H "Accept: application/json"
```

---

## 9-qadam: Testlar

`tests/Pest.php` da `RefreshDatabase` ni yoqing, keyin:

```shell
php artisan make:test PostApiTest --pest
php artisan make:test AuthApiTest --pest
```

```php
<?php

use App\Models\Post;
use App\Models\User;

use function Pest\Laravel\{actingAs, deleteJson, getJson, postJson, putJson};

it('chop etilgan postlar ro\'yxatini qaytaradi', function () {
    Post::factory()->count(3)->published()->create();
    Post::factory()->count(2)->create();               // draft

    getJson('/api/posts')
        ->assertOk()
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure(['data' => [['id', 'title', 'slug', 'author']], 'links', 'meta']);
});

it('qidiruv ishlaydi', function () {
    Post::factory()->published()->create(['title' => 'Laravel haqida']);
    Post::factory()->published()->create(['title' => 'Symfony haqida']);

    getJson('/api/posts?q=Laravel')
        ->assertOk()
        ->assertJsonCount(1, 'data')
        ->assertJsonPath('data.0.title', 'Laravel haqida');
});

it('mehmon post yarata olmaydi', function () {
    postJson('/api/posts', ['title' => 'Salom', 'body' => str_repeat('a', 30)])
        ->assertUnauthorized();
});

it('foydalanuvchi post yaratadi', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->postJson('/api/posts', [
            'title' => 'Birinchi post',
            'body' => str_repeat('matn ', 10),
            'status' => 'published',
        ])
        ->assertCreated()
        ->assertJsonPath('data.title', 'Birinchi post');

    $this->assertDatabaseHas('posts', ['user_id' => $user->id, 'slug' => 'birinchi-post']);
});

it('validatsiya xatolarini qaytaradi', function () {
    actingAs(User::factory()->create())
        ->postJson('/api/posts', [])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['title', 'body']);
});

it('begona foydalanuvchi postni tahrirlay olmaydi', function () {
    $post = Post::factory()->published()->create();

    actingAs(User::factory()->create())
        ->putJson("/api/posts/{$post->slug}", ['title' => 'Yangi sarlavha'])
        ->assertForbidden();
});

it('egasi postni tahrirlaydi va o\'chiradi', function () {
    $user = User::factory()->create();
    $post = Post::factory()->published()->for($user, 'author')->create();

    actingAs($user)
        ->putJson("/api/posts/{$post->slug}", ['title' => 'Yangilangan sarlavha'])
        ->assertOk()
        ->assertJsonPath('data.title', 'Yangilangan sarlavha');

    actingAs($user)
        ->deleteJson("/api/posts/{$post->slug}")
        ->assertNoContent();

    $this->assertDatabaseMissing('posts', ['id' => $post->id]);
});
```

```shell
php artisan test --compact
```

---

## 10-qadam: Yakuniy tozalash

```shell
vendor/bin/pint            # kod uslubini tekshirish
php artisan route:list --path=api
php artisan test
```

---

## Keyingi qadamlar (mashq sifatida)

1. **Izohlar**: `Comment` modeli, `/api/posts/{post}/comments` nested resource, policy bilan.
2. **Teglar**: `belongsToMany` aloqasi, `sync()` bilan yangilash.
3. **Rasm yuklash**: `store('covers', 'public')` va `Storage::url()`.
4. **Bildirishnoma**: post chop etilganda muallifga xat (queued notification).
5. **Kesh**: `index` natijasini 60 soniyaga keshlash va post o'zgarganda `Cache::forget`.
6. **Versiyalash**: `/api/v1/` prefiksi va `Api\V1\` namespace.
7. **Hujjat**: Postman kolleksiyasi yoki OpenAPI spetsifikatsiyasi.

---

[← Oldingi: Blade va frontend](30-blade-va-frontend.md) · [Mundarija](README.md) · [Keyingi: Optimallashtirish va deploy →](32-optimallashtirish-va-deploy.md)
