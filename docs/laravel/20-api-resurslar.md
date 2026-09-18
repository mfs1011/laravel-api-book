# 20 — API Resurslar (Eloquent API Resources)

[← Oldingi: Kolleksiyalar](19-kolleksiyalar.md) · [Mundarija](README.md) · [Keyingi: Autentifikatsiya →](21-autentifikatsiya.md)

---

## Muammo

Modelni to'g'ridan-to'g'ri qaytarsangiz:

```php
return Post::all();
```

JSON'da **bazadagi barcha ustunlar** chiqadi: `user_id`, `internal_note`, `updated_at`... Ustun nomi o'zgarsa — API mijozi buziladi. Ya'ni baza sxemasi API shartnomasiga aylanib qoladi.

**Resource** — model bilan JSON o'rtasidagi transformatsiya qatlami. Symfony'dagi Serializer guruhlari yoki API Platform'ning DTO'lariga o'xshash vazifa bajaradi.

---

## Resurs yaratish

```shell
php artisan make:resource PostResource
php artisan make:resource PostCollection --collection
```

```php
namespace App\Http\Resources;

use Illuminate\Http\Request;
use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    /**
     * @return array<string, mixed>
     */
    public function toArray(Request $request): array
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'excerpt' => str($this->body)->limit(150)->value(),
            'status' => $this->status,
            'published_at' => $this->published_at?->toIso8601String(),
            'author' => UserResource::make($this->whenLoaded('author')),
            'comments_count' => $this->whenCounted('comments'),
        ];
    }
}
```

`$this->id` — resurs ichida model xossalariga to'g'ridan-to'g'ri murojaat qilish mumkin (resource modelga proksi qiladi).

---

## Ishlatish

```php
// bitta model
return PostResource::make($post);
return new PostResource($post);
return $post->toResource();                  // Laravel 13 qulayligi

// kolleksiya
return PostResource::collection($posts);
return $posts->toResourceCollection();

// paginatsiya bilan (links va meta avtomatik qo'shiladi)
return PostResource::collection(Post::with('author')->paginate(15));
```

Model bilan resursni bog'lash (shunda `toResource()` qaysi klassni ishlatishni biladi):

```php
use Illuminate\Database\Eloquent\Attributes\UseResource;

#[UseResource(PostResource::class)]
class Post extends Model {}
```

Status va sarlavha qo'shish:

```php
return PostResource::make($post)
    ->response()
    ->setStatusCode(201)
    ->header('Location', route('posts.show', $post));
```

---

## Shartli maydonlar

```php
return [
    'id' => $this->id,
    'title' => $this->title,

    // faqat admin ko'rsin
    'internal_note' => $this->when($request->user()?->is_admin, $this->internal_note),

    // bir nechta maydonni birga
    $this->mergeWhen($request->user()?->is_admin, [
        'cost' => $this->cost,
        'margin' => $this->margin,
    ]),

    // faqat model'da bo'lsa
    'views' => $this->whenHas('views'),
    'deleted_at' => $this->whenNotNull($this->deleted_at),

    // faqat aloqa YUKLANGAN bo'lsa (N+1 dan himoya)
    'author' => UserResource::make($this->whenLoaded('author')),
    'comments' => CommentResource::collection($this->whenLoaded('comments')),

    // sanoq yuklangan bo'lsa
    'comments_count' => $this->whenCounted('comments'),

    // pivot ma'lumoti
    'sort_order' => $this->whenPivotLoaded('post_tag', fn () => $this->pivot->sort_order),
];
```

> **`whenLoaded` nega muhim?** Agar shunchaki `$this->author` yozsangiz va kontroller `with('author')` qilmagan bo'lsa, har bir element uchun alohida so'rov ketadi — N+1 ([17-bob](17-eloquent-aloqalar.md)). `whenLoaded` esa yuklanmagan bo'lsa kalitni umuman chiqarmaydi.

---

## `data` o'rami (wrapping)

Standart holatda javob `data` kalitiga o'raladi:

```json
{ "data": { "id": 1, "title": "Salom" } }
```

Paginatsiyada esa:

```json
{
  "data": [ ... ],
  "links": { "first": "...", "last": "...", "prev": null, "next": "..." },
  "meta": { "current_page": 1, "per_page": 15, "total": 42, "last_page": 3 }
}
```

O'ramni olib tashlash (butun ilova uchun):

```php
// AppServiceProvider::boot()
use Illuminate\Http\Resources\Json\JsonResource;

JsonResource::withoutWrapping();
```

> **Maslahat:** o'ramni qoldiring. `data` bo'lsa, keyinchalik javobga `meta` qo'shish API'ni buzmaydi. Paginatsiyada esa o'ram baribir majburiy.

---

## Qo'shimcha meta ma'lumot

```php
class PostResource extends JsonResource
{
    public function toArray(Request $request): array
    {
        return ['id' => $this->id, 'title' => $this->title];
    }

    /** @return array<string, mixed> */
    public function with(Request $request): array
    {
        return ['meta' => ['version' => '1.0']];
    }
}
```

Yoki bir martalik:

```php
return PostResource::make($post)->additional(['meta' => ['cached' => false]]);
```

---

## Resource Collection klassi

Butun ro'yxat uchun maxsus mantiq kerak bo'lsa:

```php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class PostCollection extends ResourceCollection
{
    public $collects = PostResource::class;

    public function toArray(Request $request): array
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'published_count' => $this->collection->where('status', 'published')->count(),
            ],
        ];
    }
}
```

```php
return new PostCollection(Post::paginate());
```

---

## JSON:API formati (Laravel 13)

Agar loyihangiz [JSON:API](https://jsonapi.org/) standartiga rioya qilsa, freymvorkda tayyor klass bor:

```shell
php artisan make:resource PostResource --json-api
```

```php
use Illuminate\Http\Resources\JsonApi\JsonApiResource;

class PostResource extends JsonApiResource
{
    public $attributes = ['title', 'body', 'created_at'];

    public $relationships = ['author' => UserResource::class];
}
```

Javob:

```json
{
  "data": {
    "id": "1",
    "type": "posts",
    "attributes": { "title": "Hello World", "body": "..." }
  }
}
```

Bu klass sparse fieldsets (`?fields[posts]=title`) va `include` parametrlarini ham o'zi qo'llab-quvvatlaydi va `Content-Type: application/vnd.api+json` sarlavhasini qo'yadi.

---

## Paginatsiya turlari

```php
Post::paginate(15);         // sahifa raqamlari + umumiy son (COUNT so'rovi bor)
Post::simplePaginate(15);   // faqat "oldingi/keyingi" (COUNT yo'q — tezroq)
Post::cursorPaginate(15);   // kursor asosida (eng tez, katta jadvallar uchun)
```

`cursorPaginate` — cheksiz skroll (infinite scroll) uchun eng to'g'ri tanlov: u `offset` ishlatmaydi, shuning uchun 1 000 000-sahifada ham tez ishlaydi va yangi yozuv qo'shilganda elementlar takrorlanib qolmaydi.

Sahifa hajmini mijoz tanlashi:

```php
$perPage = min($request->integer('per_page', 15), 100);   // yuqori chegara qo'ying

return PostResource::collection(Post::paginate($perPage));
```

---

## API versiyalash

```php
// routes/api.php
Route::prefix('v1')->name('v1.')->group(function () {
    Route::apiResource('posts', \App\Http\Controllers\Api\V1\PostController::class);
});
```

Papkalar: `app/Http/Controllers/Api/V1/`, `app/Http/Resources/V1/`. Yangi versiya chiqqanda eski ishlab turaveradi.

---

## Amaliyot

1. `php artisan make:resource PostResource` yarating va `toArray()` ni yozing.
2. Kontrollerda `PostResource::collection(Post::with('author')->paginate(15))` qaytaring, `curl` bilan `data`, `links`, `meta` tuzilmasini ko'ring.
3. `whenLoaded('author')` ni qo'shing; `with('author')` ni olib tashlab, `author` kaliti yo'qolganini ko'ring.
4. `?per_page=5` parametrini qo'llab-quvvatlang va yuqori chegarani 100 qilib qo'ying.
5. `JsonResource::withoutWrapping()` ni yoqib, javob qanday o'zgarishini ko'ring, keyin qaytaring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/eloquent-resources>
- <https://laravel.com/docs/13.x/pagination>
- <https://laravel.com/docs/13.x/eloquent-serialization>

---

[← Oldingi: Kolleksiyalar](19-kolleksiyalar.md) · [Mundarija](README.md) · [Keyingi: Autentifikatsiya →](21-autentifikatsiya.md)
