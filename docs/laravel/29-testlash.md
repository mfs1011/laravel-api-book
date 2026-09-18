# 29 — Testlash (Pest)

[← Oldingi: Mail va bildirishnomalar](28-mail-va-bildirishnoma.md) · [Mundarija](README.md) · [Keyingi: Blade va frontend →](30-blade-va-frontend.md)

---

## Bu loyihada nima bor

- **Pest 5** (PHPUnit 13 ustida ishlaydi) + `pest-plugin-laravel`
- `tests/Feature/` va `tests/Unit/` papkalari
- `phpunit.xml` da testlar uchun alohida muhit: `DB_CONNECTION=sqlite`, `DB_DATABASE=:memory:`, `CACHE_STORE=array`, `MAIL_MAILER=array`, `QUEUE_CONNECTION=sync`

Ya'ni testlar **xotiradagi SQLite** bilan ishlaydi, haqiqiy xat yubormaydi va haqiqiy navbatga tegmaydi.

```shell
php artisan test
php artisan test --compact
php artisan test tests/Feature/PostApiTest.php
php artisan test --filter="post yaratish"
php artisan test --parallel
vendor/bin/pest
```

---

## Feature va Unit farqi

| Feature test | Unit test |
| --- | --- |
| Butun ilova orqali: HTTP so'rov → route → kontroller → baza | Bitta klass/metod alohida |
| Sekinroq, lekin haqiqatga yaqin | Juda tez |
| **Ko'p testlaringiz shu turda bo'lsin** | Murakkab hisob-kitob mantiqi uchun |

Laravel'da asosiy urg'u **feature testlarga** beriladi: ular haqiqiy qiymatni tekshiradi — endpoint ishlaydimi, ruxsatlar to'g'rimi, baza to'g'ri o'zgardimi.

---

## Birinchi test

```shell
php artisan make:test PostApiTest --pest
php artisan make:test SlugGeneratorTest --pest --unit
```

```php
<?php

use App\Models\Post;
use App\Models\User;

use function Pest\Laravel\{actingAs, getJson, postJson};

it('postlar ro\'yxatini qaytaradi', function () {
    Post::factory()->count(3)->published()->create();

    getJson('/api/posts')
        ->assertOk()
        ->assertJsonCount(3, 'data')
        ->assertJsonStructure([
            'data' => [['id', 'title', 'slug']],
            'links',
            'meta',
        ]);
});

it('mehmon post yarata olmaydi', function () {
    postJson('/api/posts', ['title' => 'Salom'])->assertUnauthorized();   // 401
});

it('autentifikatsiyadan o\'tgan foydalanuvchi post yaratadi', function () {
    $user = User::factory()->create();

    actingAs($user)
        ->postJson('/api/posts', [
            'title' => 'Birinchi post',
            'body' => str_repeat('matn ', 30),
        ])
        ->assertCreated()                                // 201
        ->assertJsonPath('data.title', 'Birinchi post');

    expect(Post::where('user_id', $user->id)->count())->toBe(1);
});

it('validatsiya xatolarini qaytaradi', function () {
    actingAs(User::factory()->create())
        ->postJson('/api/posts', [])
        ->assertStatus(422)
        ->assertJsonValidationErrors(['title', 'body']);
});
```

---

## Bazani tozalash

```php
// tests/Pest.php
use Illuminate\Foundation\Testing\RefreshDatabase;

pest()->extend(Tests\TestCase::class)
    ->use(RefreshDatabase::class)
    ->in('Feature');
```

`RefreshDatabase` har testdan oldin migratsiyalarni bajaradi va testni tranzaksiyada o'rab, oxirida `rollback` qiladi. Natijada testlar bir-biriga ta'sir qilmaydi.

> Bu loyihaning `tests/Pest.php` faylida `->use(RefreshDatabase::class)` qatori **izohga olingan**. Baza bilan ishlaydigan testlar yozishdan oldin uni yoqing.

Alternativalar: `DatabaseMigrations` (har test uchun to'liq migratsiya, sekinroq), `DatabaseTruncation`.

---

## Pest sintaksisi

```php
beforeEach(function () {
    $this->user = User::factory()->create();
});

it('ishlaydi', function () {
    expect(true)->toBeTrue();
});

test('sarlavha bilan ham yozish mumkin', function () { });

describe('PostPolicy', function () {
    it('egasiga ruxsat beradi', function () { });
    it('boshqasiga ruxsat bermaydi', function () { });
});

it('faqat admin uchun', function () { })->skip('keyinroq');
it('sekin test', function () { })->group('slow');
```

### Expectation'lar

```php
expect($value)->toBe(5);
expect($value)->toEqual(['a' => 1]);
expect($user->name)->toBe('Ali');
expect($posts)->toHaveCount(3);
expect($post->published_at)->not->toBeNull();
expect($response->json())->toHaveKey('data.0.id');
expect(fn () => $service->run())->toThrow(InvalidArgumentException::class);
expect($user)->toBeInstanceOf(User::class);
```

### Datasets (bir xil testni bir nechta ma'lumot bilan)

```php
it('email formatini tekshiradi', function (string $email, bool $valid) {
    expect(filter_var($email, FILTER_VALIDATE_EMAIL) !== false)->toBe($valid);
})->with([
    ['ali@example.uz', true],
    ['notanemail', false],
]);
```

---

## HTTP testlari uchun asosiy assertion'lar

```php
$response->assertOk();                 // 200
$response->assertCreated();            // 201
$response->assertNoContent();          // 204
$response->assertUnauthorized();       // 401
$response->assertForbidden();          // 403
$response->assertNotFound();           // 404
$response->assertStatus(422);
$response->assertJson(['ok' => true]);
$response->assertJsonPath('data.title', 'Salom');
$response->assertJsonCount(3, 'data');
$response->assertJsonStructure(['data' => ['id', 'title']]);
$response->assertJsonValidationErrors(['title']);
$response->assertJsonMissing(['password']);
$response->assertHeader('Location');
$response->assertSee('Matn');          // HTML uchun
```

Fluent JSON:

```php
$response->assertJson(fn (AssertableJson $json) =>
    $json->has('data', 3)
        ->has('data.0', fn ($json) =>
            $json->where('title', 'Salom')
                ->missing('internal_note')
                ->etc()
        )
        ->etc()
);
```

Baza tekshiruvlari:

```php
$this->assertDatabaseHas('posts', ['slug' => 'salom']);
$this->assertDatabaseMissing('posts', ['id' => $post->id]);
$this->assertDatabaseCount('posts', 3);
$this->assertSoftDeleted($post);
$this->assertModelExists($post);
```

Autentifikatsiya:

```php
$this->assertAuthenticated();
$this->assertAuthenticatedAs($user);
$this->assertGuest();
```

---

## Tashqi bog'liqliklarni soxtalashtirish

```php
Mail::fake();  Notification::fake();  Queue::fake();  Bus::fake();
Event::fake(); Storage::fake('public'); Http::fake();

// vaqtni "to'xtatish"
$this->travelTo(now()->addDays(3));
$this->travel(5)->days();
$this->freezeTime();

// konteynerda soxta xizmat
$this->mock(SmsSender::class, fn ($mock) => $mock->shouldReceive('send')->once());
$this->instance(SmsSender::class, new FakeSmsSender);
```

> **Qoida:** test hech qachon haqiqiy tashqi tizimga (SMS, to'lov, S3) so'rov yubormasligi kerak. Sekin, ishonchsiz va pulga tushadi.

---

## Nima test qilish kerak

**Albatta:**
- Har bir API endpoint: muvaffaqiyatli holat + validatsiya xatosi + ruxsat yo'qligi
- Policy qoidalari (egasi/begona)
- Muhim biznes mantiq (hisob-kitob, holat o'zgarishi)
- Bug topilganda — avval uni ko'rsatuvchi test yozing, keyin tuzating

**Shart emas:**
- Freymvorkning o'zini (Eloquent `save()` ishlaydimi)
- Getter/setter'lar
- Konfiguratsiya fayllari

---

## Arxitektura testlari (Pest Arch)

```php
arch('modellar faqat Model dan meros oladi')
    ->expect('App\Models')
    ->toExtend('Illuminate\Database\Eloquent\Model');

arch('kontrollerlarda dd qolmasin')
    ->expect(['dd', 'dump', 'ray'])
    ->not->toBeUsed();

arch('preset')->preset()->laravel();
```

Bu testlar kod uslubini avtomatik nazorat qiladi.

---

## Mutatsion testlash

```shell
vendor/bin/pest --mutate --covered-only
```

Pest kodingizni ataylab "buzadi" (masalan `>` ni `>=` ga o'zgartiradi) va testlaringiz buni sezadimi, tekshiradi. Bu — testlarning **haqiqiy sifatini** o'lchash usuli (oddiy coverage foizidan ancha foydali).

---

## Tezlik

```shell
php artisan test --parallel
php artisan test --parallel --recreate-databases
```

Maslahatlar: `phpunit.xml` da `BCRYPT_ROUNDS=4` (allaqachon qo'yilgan), xotiradagi SQLite, `RefreshDatabase` (tranzaksiya — eng tez), keraksiz `sleep()` lardan voz kechish.

---

## Amaliyot

1. `tests/Pest.php` da `RefreshDatabase` ni yoqing.
2. `php artisan make:test PostApiTest --pest` yarating va yuqoridagi 4 ta testni yozing.
3. `php artisan test --filter=PostApiTest` bilan ishga tushiring.
4. Policy testini yozing: begona foydalanuvchi 403 olsin.
5. `Queue::fake()` bilan job navbatga qo'yilganini tekshiring.
6. `vendor/bin/pest --mutate --covered-only` ni ishlatib ko'ring.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/testing>
- <https://laravel.com/docs/13.x/http-tests>
- <https://laravel.com/docs/13.x/database-testing>
- <https://laravel.com/docs/13.x/mocking>
- <https://pestphp.com/docs>

---

[← Oldingi: Mail va bildirishnomalar](28-mail-va-bildirishnoma.md) · [Mundarija](README.md) · [Keyingi: Blade va frontend →](30-blade-va-frontend.md)
