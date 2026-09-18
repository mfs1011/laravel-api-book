# 39 — Arxitektura: loyiha o'sganda

[← Oldingi: API pro darajasi](38-api-pro.md) · [Mundarija](README.md) · [Keyingi: CI/CD va Docker →](40-cicd-va-production.md)

---

## Muammo

Laravel'ning standart tuzilmasi (`Controllers`, `Models`) 10-20 endpointgacha juda yaxshi ishlaydi. 100 endpoint va 5 yillik loyihada esa: 800 qatorli kontrollerlar, 1500 qatorli modellar, "bu logika qayerda?" degan savol.

Bu bobda **qachon** va **qanday** qatlam qo'shish kerakligini ko'ramiz. Asosiy qoida: **erta abstraktsiya — zarar**. Muammo paydo bo'lgandan keyin qatlam qo'shing.

---

## 1. Action / Service klasslar (birinchi va eng foydali qadam)

Kontroller uchta ish qilishi kerak: so'rovni qabul qilish, amalni chaqirish, javob qaytarish. Qolgani — alohida klassda.

```shell
php artisan make:class Actions/PlaceOrder
```

```php
namespace App\Actions;

use App\Models\Order;
use App\Models\User;
use Illuminate\Support\Facades\DB;

class PlaceOrder
{
    public function __construct(
        private InventoryService $inventory,
        private PaymentGateway $payments,
    ) {}

    /** @param array<int, array{product_id: int, qty: int}> $items */
    public function handle(User $user, array $items, string $paymentToken): Order
    {
        return DB::transaction(function () use ($user, $items, $paymentToken) {
            $this->inventory->reserve($items);

            $order = $user->orders()->create(['status' => 'pending']);
            $order->items()->createMany($items);

            $this->payments->charge($paymentToken, $order->total());

            $order->update(['status' => 'paid']);

            OrderPlaced::dispatch($order);

            return $order;
        });
    }
}
```

```php
public function store(StoreOrderRequest $request, PlaceOrder $placeOrder)
{
    $order = $placeOrder->handle(
        user: $request->user(),
        items: $request->validated('items'),
        paymentToken: $request->validated('payment_token'),
    );

    return OrderResource::make($order)->response()->setStatusCode(201);
}
```

**Foydasi:**

- Bitta amal — bitta fayl; qidirish oson.
- Kontroller, Artisan buyruq, queue job va test — hammasi shu klassni chaqiradi.
- Unit test yozish oson (HTTP qatlamisiz).

> **Symfony bilan solishtirish:** bu — sizga tanish "Service" qatlami. Laravel'da u `app/Actions/` yoki `app/Services/` da yashaydi; farq faqat nomlashda.

**Nomlash:** `PlaceOrder`, `PublishPost`, `CancelSubscription` — **fe'l + ot**. Klass bitta narsa qilsin. Agar `OrderService` ichida 15 ta metod paydo bo'lsa — uni bo'lish vaqti kelgan.

---

## 2. DTO (Data Transfer Object)

Assotsiativ massivlar (`array $items`) o'sib ketganda tipli ob'ektga o'ting:

```php
namespace App\Data;

final readonly class OrderItemData
{
    public function __construct(
        public int $productId,
        public int $quantity,
    ) {}

    /** @param array{product_id: int, qty: int} $row */
    public static function fromArray(array $row): self
    {
        return new self($row['product_id'], $row['qty']);
    }
}
```

```php
$items = array_map(OrderItemData::fromArray(...), $request->validated('items'));
$placeOrder->handle($user, $items, $token);
```

➕ IDE avtoto'ldirish, statik tahlil, typo yo'q. ➖ ko'proq kod.

**Qachon:** ma'lumot 3+ qatlamdan o'tsa yoki tuzilmasi murakkab bo'lsa. Oddiy CRUD'da `validated()` massivi yetarli.

Paket: `spatie/laravel-data` (validatsiya + DTO + resource'ni birlashtiradi).

---

## 3. Repository kerakmi?

**Ko'p hollarda — yo'q.** Eloquent allaqachon repository naqshining o'zi: `Post::published()->paginate()`. Ustiga yana bir qatlam qo'shish — kod ko'payadi, foyda kam.

Repository **haqiqatan foydali** bo'lgan holatlar:

1. Ma'lumot bir nechta manbadan keladi (baza + tashqi API + kesh) va chaqiruvchi buni bilmasligi kerak.
2. Ma'lumot manbasini almashtirish rejasi bor (masalan Eloquent → Elasticsearch).
3. Domen qatlami freymvorkdan mustaqil bo'lishi kerak (qat'iy DDD).

Buning o'rniga oddiyroq vositalar:

```php
// 1) Scope — takrorlanuvchi shartlar modelning o'zida ([16-bob])
class Post extends Model
{
    #[Scope]
    protected function published(Builder $query): void { /* ... */ }
}
```

```php
// 2) Query klass — bitta murakkab so'rov uchun alohida klass
final class PopularPostsQuery
{
    public function __invoke(int $days = 7): Collection
    {
        return Post::published()
            ->where('published_at', '>=', now()->subDays($days))
            ->withCount('comments')
            ->orderByDesc('comments_count')
            ->limit(10)
            ->get();
    }
}
```

---

## 4. Modelni yengil ushlash

Modelda bo'lishi kerak: aloqalar, cast'lar, scope'lar, accessor'lar, kichik yordamchi metodlar (`isPublished()`).

Modeldan chiqarish kerak: xat yuborish, to'lov, hisobot, tashqi API — bular Action/Service ishi.

Shishgan modelni bo'lish:

```php
// trait bilan guruhlash
class Post extends Model
{
    use HasComments, Publishable, Searchable;
}
```

> Trait — tashkiliy vosita, sehr emas. Agar `Publishable` trait'i 200 qator bo'lsa, u alohida klass bo'lishi kerak edi.

---

## 5. Enum va qiymat ob'ektlari

Satr holatlar (`'draft'`, `'published'`) o'rniga enum:

```php
namespace App\Enums;

enum OrderStatus: string
{
    case Pending = 'pending';
    case Paid = 'paid';
    case Shipped = 'shipped';
    case Cancelled = 'cancelled';

    public function label(): string
    {
        return match ($this) {
            self::Pending => 'Kutilmoqda',
            self::Paid => 'To\'langan',
            self::Shipped => 'Jo\'natilgan',
            self::Cancelled => 'Bekor qilingan',
        };
    }

    /** @return array<int, self> */
    public function allowedTransitions(): array
    {
        return match ($this) {
            self::Pending => [self::Paid, self::Cancelled],
            self::Paid => [self::Shipped, self::Cancelled],
            default => [],
        };
    }

    public function canTransitionTo(self $next): bool
    {
        return in_array($next, $this->allowedTransitions(), true);
    }
}
```

```php
protected function casts(): array
{
    return ['status' => OrderStatus::class];
}
```

```php
abort_unless($order->status->canTransitionTo(OrderStatus::Shipped), 409);
```

**Nega bu kuchli?** Holat mashinasi (state machine) qoidalari bitta joyda turadi va butun ilova bo'ylab bir xil ishlaydi. Validatsiyada ham ishlatiladi: `Rule::enum(OrderStatus::class)`.

---

## 6. Modulli tuzilma (katta loyihalar)

100+ fayl bo'lganda "tur bo'yicha" (`Controllers/`, `Models/`) emas, **"domen bo'yicha"** guruhlash qulayroq:

```
app/
├── Domains/
│   ├── Orders/
│   │   ├── Actions/        PlaceOrder, CancelOrder
│   │   ├── Models/         Order, OrderItem
│   │   ├── Http/           Controllers, Requests, Resources
│   │   ├── Events/         OrderPlaced
│   │   ├── Listeners/
│   │   └── Policies/
│   ├── Catalog/
│   └── Billing/
└── Support/                umumiy yordamchilar
```

Sozlash kerak bo'lgan joylar:

```php
// bootstrap/app.php
->withEvents(discover: [__DIR__.'/../app/Domains/*/Listeners'])
->withCommands([__DIR__.'/../app/Domains/*/Console'])
```

```php
// route fayllarini domen bo'yicha bo'lish
->withRouting(
    api: __DIR__.'/../routes/api.php',
    then: function () {
        foreach (glob(base_path('app/Domains/*/routes.php')) as $file) {
            Route::middleware('api')->prefix('api')->group($file);
        }
    },
)
```

Policy avtomatik topilishi konvensiyaga tayanadi, shuning uchun modelda aniq ko'rsating:

```php
#[UsePolicy(OrderPolicy::class)]
class Order extends Model {}
```

> **Ogohlantirish:** modulli tuzilma — **katta** loyihalar uchun. 30 ta fayl bor loyihada u faqat murakkablik qo'shadi. Laravel konvensiyalaridan chiqqaningizda bir nechta narsani qo'lda sozlash kerak bo'ladi.

Chegaralarni test bilan qotirish (Pest Arch):

```php
arch("Orders domeni Catalog ichki qismiga bog'lanmasin")
    ->expect('App\Domains\Orders')
    ->not->toUse('App\Domains\Catalog\Actions');
```

---

## 7. Domen hodisalari va bog'liqlikni kamaytirish

`PlaceOrder` ichida xat yuborish, omborni yangilash, statistika — hammasini yozsangiz, u har hafta o'sadi. Yechim — hodisa e'lon qilish va tinglovchilarni alohida yozish ([26-bob](26-hodisalar-va-observerlar.md)):

```php
OrderPlaced::dispatch($order);
```

```
App\Domains\Orders\Listeners\SendOrderConfirmation
App\Domains\Billing\Listeners\CreateInvoice
App\Domains\Catalog\Listeners\DecrementStock
```

Har bir tinglovchi mustaqil, alohida test qilinadi, navbatga qo'yiladi.

> **Chegara:** agar amal **muvaffaqiyatli bo'lishi shart** bo'lsa (masalan to'lov), uni hodisaga chiqarmang — u asosiy oqimda va tranzaksiyada qolsin. Hodisa — "yon ta'sir" uchun.

---

## 8. O'z paketingizni yozish

Bir nechta loyihada takrorlanadigan kod bo'lsa, uni paketga chiqaring:

```
packages/mycompany/sms/
├── composer.json
├── config/sms.php
├── src/
│   ├── SmsServiceProvider.php
│   ├── SmsManager.php
│   └── Facades/Sms.php
└── tests/
```

```json
{
  "name": "mycompany/sms",
  "autoload": { "psr-4": { "MyCompany\\Sms\\": "src/" } },
  "extra": { "laravel": { "providers": ["MyCompany\\Sms\\SmsServiceProvider"] } }
}
```

```php
public function register(): void
{
    $this->mergeConfigFrom(__DIR__.'/../config/sms.php', 'sms');
    $this->app->singleton(SmsManager::class);
}

public function boot(): void
{
    $this->publishes([__DIR__.'/../config/sms.php' => config_path('sms.php')], 'sms-config');
    $this->loadMigrationsFrom(__DIR__.'/../database/migrations');
}
```

Lokal ishlab chiqishda loyiha `composer.json` iga:

```json
"repositories": [{ "type": "path", "url": "packages/mycompany/sms" }],
"require": { "mycompany/sms": "*" }
```

Paketni sinash uchun `orchestra/testbench` ishlatiladi ([06-bob](06-service-provider.md)).

---

## 9. Amaliy qarorlar jadvali

| Belgi | Qadam |
| --- | --- |
| Kontroller metodi 30 qatordan oshdi | Action klass ajrating |
| Bitta mantiq 3 joyda takrorlandi | Umumiy klass/scope ga chiqaring |
| Model 300 qatordan oshdi | Trait/Action ga bo'ling |
| Massivda 5+ kalit qatlamlar orasida yuribdi | DTO qiling |
| Satr holatlar `if` larda ko'paydi | Enum + holat mashinasi |
| Jamoada 3+ dasturchi, domenlar aniq | Modulli tuzilma |
| Bir necha loyihada bir xil kod | Paket |

Va eng muhim qoida: **har qadamda test bo'lsin** ([29-bob](29-testlash.md)). Refaktoring testsiz — qimor.

---

## Nima qilmaslik kerak

- **Hamma narsaga interfeys.** Bitta implementatsiya bo'lsa interfeys kerak emas.
- **Repository "shunchaki to'g'ri bo'lgani uchun".** Foydasini ayta olmasangiz — kerak emas.
- **Hexagonal/DDD'ni to'liq ko'chirish.** Laravel'da bu ko'p ishlaydigan naqsh emas; jamoa tushunmasa zarar keltiradi.
- **Erta mikroservislar.** Monolit + navbatlar 95% loyihaga yetadi.
- **Freymvorkka qarshi kurashish.** Eloquent'ni yashirish, fasadlarni taqiqlash — ko'p mehnat, kam foyda.

---

## Amaliyot

1. 31-bobdagi loyihada `PostController::store()` mantig'ini `App\Actions\PublishPost` ga ko'chiring va test yozing.
2. `status` ni `OrderStatus` enum'iga aylantiring va `canTransitionTo()` bilan noto'g'ri o'tishda 409 qaytaring.
3. `spatie/laravel-data` ni sinab ko'ring yoki oddiy DTO yozing.
4. Pest Arch testi yozing: `App\Http\Controllers` ichida `DB::` ishlatilmasin.
5. Kichik paket yarating (`packages/mycompany/sms`) va uni loyihaga `path` repository orqali ulang.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/packages>
- <https://laravel.com/docs/13.x/container>
- <https://pestphp.com/docs/arch-testing>

---

[← Oldingi: API pro darajasi](38-api-pro.md) · [Mundarija](README.md) · [Keyingi: CI/CD va Docker →](40-cicd-va-production.md)
