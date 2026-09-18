# 26 — Hodisalar, listener'lar va observerlar

[← Oldingi: Navbatlar](25-navbatlar.md) · [Mundarija](README.md) · [Keyingi: Kesh va fayllar →](27-kesh-va-fayllar.md)

---

## Nima uchun kerak

Buyurtma yaratilganda: mijozga xat, adminga bildirishnoma, omborga signal, statistikani yangilash... Bularning hammasini kontrollerga yozsangiz, u shishib ketadi va har yangi talab kontrollerni o'zgartirishni talab qiladi.

Hodisa (event) yondashuvi: kontroller faqat **"buyurtma yaratildi"** deb e'lon qiladi; kim eshitishi va nima qilishi — alohida klasslarda.

> **Symfony bilan solishtirish:** EventDispatcher bilan bir xil. Farqi: Laravel'da listener'lar avtomatik topiladi va ularni navbatga qo'yish bitta interfeys bilan hal bo'ladi.

---

## Event va Listener yaratish

```shell
php artisan make:event OrderPlaced
php artisan make:listener SendOrderConfirmation --event=OrderPlaced
```

```php
namespace App\Events;

use App\Models\Order;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderPlaced
{
    use Dispatchable, SerializesModels;

    public function __construct(public Order $order) {}
}
```

```php
namespace App\Listeners;

use App\Events\OrderPlaced;

class SendOrderConfirmation
{
    public function handle(OrderPlaced $event): void
    {
        $event->order->user->notify(new OrderConfirmation($event->order));
    }
}
```

---

## Ro'yxatdan o'tkazish: avtomatik topish

Laravel `app/Listeners` papkasini skanerlaydi va `handle` (yoki `__invoke`) metodidagi **tip ko'rsatilgan argument**ga qarab listener'ni tegishli hodisaga bog'laydi. Ya'ni hech qanday ro'yxat yozish shart emas.

Bir nechta hodisani eshitish:

```php
public function handle(OrderPlaced|OrderUpdated $event): void {}
```

Boshqa papkalarni skanerlash:

```php
// bootstrap/app.php
->withEvents(discover: [
    __DIR__.'/../app/Domain/*/Listeners',
])
```

Qo'lda bog'lash ham mumkin:

```php
// AppServiceProvider::boot()
Event::listen(OrderPlaced::class, SendOrderConfirmation::class);

Event::listen(function (OrderPlaced $event) {
    // closure listener
});
```

Tekshirish:

```shell
php artisan event:list
```

---

## Hodisani e'lon qilish

```php
OrderPlaced::dispatch($order);
event(new OrderPlaced($order));

OrderPlaced::dispatchIf($order->isPaid(), $order);
OrderPlaced::dispatchUnless($order->isTest(), $order);
```

Tranzaksiyadan keyin yuborish:

```php
use Illuminate\Contracts\Events\ShouldDispatchAfterCommit;

class OrderPlaced implements ShouldDispatchAfterCommit {}
```

---

## Listener'ni navbatga qo'yish

Bu — hodisalarning eng katta amaliy foydasi:

```php
use Illuminate\Contracts\Queue\ShouldQueue;

class SendOrderConfirmation implements ShouldQueue
{
    public string $queue = 'notifications';
    public int $tries = 3;
    public int $backoff = 10;

    public function handle(OrderPlaced $event): void { /* ... */ }

    public function failed(OrderPlaced $event, \Throwable $e): void { /* ... */ }
}
```

Endi `OrderPlaced::dispatch($order)` chaqirilganda foydalanuvchi kutmaydi — xat fonda yuboriladi ([25-bob](25-navbatlar.md)).

Shartli navbat:

```php
public function shouldQueue(OrderPlaced $event): bool
{
    return $event->order->total > 100_000;
}
```

---

## Freymvork hodisalari

Laravel o'zi ham ko'p hodisa tarqatadi — ularga obuna bo'lish mumkin:

```php
Illuminate\Auth\Events\Login;
Illuminate\Auth\Events\Failed;
Illuminate\Auth\Events\Registered;
Illuminate\Queue\Events\JobFailed;
Illuminate\Database\Events\QueryExecuted;
Illuminate\Mail\Events\MessageSent;
```

Masalan, muvaffaqiyatsiz login urinishlarini loglash:

```php
Event::listen(function (\Illuminate\Auth\Events\Failed $event) {
    logger()->warning('Login urinishi muvaffaqiyatsiz', ['email' => $event->credentials['email'] ?? null]);
});
```

---

## Model observerlari

Model hodisalari (`created`, `updated`, `deleted`...) uchun alohida klass:

```shell
php artisan make:observer PostObserver --model=Post
```

```php
namespace App\Observers;

use App\Models\Post;

class PostObserver
{
    public function creating(Post $post): void
    {
        $post->slug ??= str($post->title)->slug()->value();
    }

    public function created(Post $post): void
    {
        SearchIndex::queue($post);
    }

    public function updated(Post $post): void
    {
        if ($post->wasChanged('status')) {
            PostStatusChanged::dispatch($post);
        }
    }

    public function deleted(Post $post): void
    {
        $post->comments()->delete();
    }
}
```

Ro'yxatdan o'tkazish — atribut bilan:

```php
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy([PostObserver::class])]
class Post extends Model {}
```

Tranzaksiyadan keyin ishlashi uchun:

```php
use Illuminate\Contracts\Events\ShouldHandleEventsAfterCommit;

class PostObserver implements ShouldHandleEventsAfterCommit {}
```

### ⚠️ Observer tuzoqlari

1. **Ommaviy amallar hodisa tarqatmaydi.** `Post::where(...)->update([...])` da `updated` ishlamaydi — chunki modellar umuman yuklanmaydi. Kerak bo'lsa `each()` bilan aylanib chiqing yoki buni bilib ish tuting.
2. **Seed va import paytida** observerlar keraksiz ishga tushadi — `WithoutModelEvents` trait'i yoki `Model::withoutEvents()` ishlating.
3. **Yashirin mantiq.** Observer kodni "sehrli" qiladi: kontrollerga qarab nima sodir bo'lishini bilib bo'lmaydi. Shuning uchun observerga faqat **model bilan chambarchas bog'liq** ish (slug yaratish, indeks yangilash) yozing; biznes jarayonlar uchun aniq `Event` yaxshiroq.

Hodisasiz saqlash:

```php
$post->saveQuietly();
$post->deleteQuietly();
Post::withoutEvents(fn () => $post->update([...]));
```

---

## Testlash

```php
use Illuminate\Support\Facades\Event;

Event::fake();

// ... amalni bajaramiz ...

Event::assertDispatched(OrderPlaced::class);
Event::assertDispatched(OrderPlaced::class, fn ($e) => $e->order->id === $order->id);
Event::assertNotDispatched(OrderCancelled::class);
Event::assertDispatchedTimes(OrderPlaced::class, 1);

// faqat ba'zi hodisalarni fake qilish
Event::fake([OrderPlaced::class]);
```

Listener'ni alohida sinash:

```php
(new SendOrderConfirmation)->handle(new OrderPlaced($order));
```

---

## Qachon nima ishlatish kerak

| Vaziyat | Yechim |
| --- | --- |
| Modelning o'zi bilan bog'liq kichik ish (slug, indeks) | **Observer** |
| Biznes jarayoni ("buyurtma joylandi") | **Event + Listener** |
| Uzoq davom etadigan ish | **Job** (yoki queued listener) |
| Bitta joydan chaqiriladigan oddiy amal | Oddiy **Service/Action klassi** — hodisa shart emas |

> Hodisalarni haddan tashqari ko'p ishlatish kodni kuzatishni qiyinlashtiradi. "Bu ishni kim qiladi?" degan savolga javob topish qiyin bo'lsa — demak, hodisa ortiqcha.

---

## Amaliyot

1. `OrderPlaced` hodisasi va `SendOrderConfirmation` listener'ini yarating; `php artisan event:list` da bog'lanish paydo bo'lganini ko'ring.
2. Listener'ga `ShouldQueue` qo'shing va `queue:work` bilan fonda bajarilishini tekshiring.
3. `PostObserver` yozing: `creating` da slug avtomatik to'lsin.
4. `Post::where(...)->update(...)` qilib, observer ishlamasligini o'z ko'zingiz bilan ko'ring.
5. Testda `Event::fake()` + `assertDispatched` yozing.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/events>
- <https://laravel.com/docs/13.x/eloquent#observers>

---

[← Oldingi: Navbatlar](25-navbatlar.md) · [Mundarija](README.md) · [Keyingi: Kesh va fayllar →](27-kesh-va-fayllar.md)
