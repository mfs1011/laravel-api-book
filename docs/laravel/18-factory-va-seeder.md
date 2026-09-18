# 18 — Factory va Seeder

[← Oldingi: Eloquent aloqalari](17-eloquent-aloqalar.md) · [Mundarija](README.md) · [Keyingi: Kolleksiyalar →](19-kolleksiyalar.md)

---

## Factory nima uchun

Factory — model uchun **soxta ma'lumot generatori**. Testlarda va lokal bazani to'ldirishda ishlatiladi. Symfony'dagi `zenstruck/foundry` yoki Doctrine fixtures'ga o'xshaydi, lekin freymvork ichida keladi.

```shell
php artisan make:factory PostFactory --model=Post
```

```php
namespace Database\Factories;

use App\Models\Post;
use App\Models\User;
use Illuminate\Database\Eloquent\Factories\Factory;

/**
 * @extends Factory<Post>
 */
class PostFactory extends Factory
{
    /**
     * @return array<string, mixed>
     */
    public function definition(): array
    {
        $title = fake()->sentence();

        return [
            'user_id' => User::factory(),          // bog'langan model ham yaratiladi
            'title' => $title,
            'slug' => str($title)->slug()->value(),
            'body' => fake()->paragraphs(3, true),
            'status' => 'draft',
            'published_at' => null,
        ];
    }

    public function published(): static
    {
        return $this->state(fn (array $attributes) => [
            'status' => 'published',
            'published_at' => now()->subDays(fake()->numberBetween(1, 30)),
        ]);
    }

    public function forAuthor(User $user): static
    {
        return $this->state(['user_id' => $user->id]);
    }
}
```

Model bilan bog'lash `HasFactory` trait'i orqali avtomatik:

```php
class Post extends Model
{
    use HasFactory;
}
```

---

## Ishlatish

```php
Post::factory()->create();                       // 1 ta, bazaga saqlanadi
Post::factory()->count(10)->create();
Post::factory(10)->create();                     // qisqa shakl
Post::factory()->make();                         // saqlanmaydi (xotirada)
Post::factory()->published()->create();          // holat (state)
Post::factory()->create(['title' => 'Aniq sarlavha']);

// aloqalar bilan
User::factory()
    ->has(Post::factory()->count(3))
    ->create();

Post::factory()
    ->for(User::factory()->create(['name' => 'Ali']), 'author')
    ->count(5)
    ->create();

// ko'p-ga-ko'p
Post::factory()->hasAttached(Tag::factory()->count(3))->create();

// ketma-ket qiymatlar
Post::factory()->count(6)->sequence(
    ['status' => 'draft'],
    ['status' => 'published'],
)->create();

// har bir yaratilgandan keyin qo'shimcha ish
Post::factory()->count(3)->afterCreating(fn (Post $post) => $post->tags()->attach(1))->create();
```

> **Nega `create()` va `make()` ikkalasi bor?** `make()` bazaga tegmaydi — unit testlarda tez ishlaydi. `create()` esa haqiqiy yozuv yaratadi — feature testlarda va seed'da kerak.

---

## Faker

`fake()` helperi `fakerphp/faker` ni qaytaradi:

```php
fake()->name();
fake()->unique()->safeEmail();
fake()->sentence();
fake()->paragraphs(3, true);
fake()->numberBetween(1, 100);
fake()->randomElement(['draft', 'published']);
fake()->boolean(70);            // 70% ehtimol bilan true
fake()->dateTimeBetween('-1 year', 'now');
fake()->uuid();
fake()->imageUrl();
fake()->optional()->text();     // ba'zan null
```

Tilni `.env` da tanlash mumkin:

```ini
APP_FAKER_LOCALE=uz_UZ
```

---

## Seeder

Seeder — bazaga **boshlang'ich yoki demo ma'lumot** joylash.

```shell
php artisan make:seeder PostSeeder
```

```php
namespace Database\Seeders;

use App\Models\Post;
use App\Models\User;
use Illuminate\Database\Seeder;

class PostSeeder extends Seeder
{
    public function run(): void
    {
        $author = User::factory()->create(['email' => 'author@example.com']);

        Post::factory()
            ->count(20)
            ->published()
            ->forAuthor($author)
            ->create();
    }
}
```

`DatabaseSeeder` — kirish nuqtasi:

```php
class DatabaseSeeder extends Seeder
{
    use WithoutModelEvents;      // seed paytida model hodisalari ishlamasin

    public function run(): void
    {
        $this->call([
            UserSeeder::class,
            PostSeeder::class,
        ]);
    }
}
```

```shell
php artisan db:seed
php artisan db:seed --class=PostSeeder
php artisan migrate:fresh --seed
```

> **`WithoutModelEvents` nega kerak?** Seed paytida `created` hodisasi ishga tushsa, u bildirishnoma yuborishi yoki job navbatga qo'yishi mumkin. Seed'da bu keraksiz (va sekin).

Katta hajmda `insert` tezroq:

```php
DB::table('regions')->insert([
    ['name' => 'Toshkent'],
    ['name' => 'Samarqand'],
]);
```

---

## Testlarda factory

```php
use function Pest\Laravel\actingAs;

it('foydalanuvchi o\'z postini yangilay oladi', function () {
    $user = User::factory()->create();
    $post = Post::factory()->forAuthor($user)->create();

    actingAs($user)
        ->putJson("/api/posts/{$post->id}", ['title' => 'Yangilandi'])
        ->assertOk()
        ->assertJsonPath('data.title', 'Yangilandi');
});
```

Batafsil [29-bob](29-testlash.md).

---

## Yaxshi amaliyotlar

1. **Testda modelni qo'lda yaratmang** — factory ishlating. Migratsiyaga ustun qo'shilsa faqat factory'ni yangilaysiz.
2. Takrorlanuvchi holatlarni **state** ga chiqaring (`published()`, `withComments()`).
3. Seeder — demo ma'lumot uchun; **productionga kerakli ma'lumot** (masalan viloyatlar ro'yxati) uchun alohida, idempotent seeder yozing (`updateOrCreate` bilan).
4. Faktoriylar `database/factories/` da, seederlar `database/seeders/` da turadi va **git'da bo'ladi**.

---

## Amaliyot

1. `php artisan make:factory PostFactory --model=Post` yarating va `definition()` ni to'ldiring.
2. `published()` state'ini qo'shing.
3. Tinker'da: `Post::factory()->count(5)->published()->create();`
4. `PostSeeder` yozing, `DatabaseSeeder` ga ulang va `php artisan migrate:fresh --seed` ni bajaring.
5. `User::factory()->has(Post::factory()->count(3))->create()` ni sinang va aloqa to'g'ri bog'langanini tekshiring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/eloquent-factories>
- <https://laravel.com/docs/13.x/seeding>

---

[← Oldingi: Eloquent aloqalari](17-eloquent-aloqalar.md) · [Mundarija](README.md) · [Keyingi: Kolleksiyalar →](19-kolleksiyalar.md)
