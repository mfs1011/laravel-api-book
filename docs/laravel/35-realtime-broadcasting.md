# 35 — Real-time: Broadcasting va WebSocket

[← Oldingi: Symfony → Laravel lug'ati](34-symfony-laravel-lugat.md) · [Mundarija](README.md) · [Keyingi: Ko'p tillilik →](36-koptillilik-va-mintaqa.md)

---

## Muammo

Yangi xabar kelganda foydalanuvchi sahifani yangilamasdan ko'rishi kerak. "Har 5 soniyada so'rov yuborish" (polling) — server uchun qimmat va sekin. To'g'ri yechim — **WebSocket**: server o'zi mijozga xabar yuboradi.

Laravel'da bu **broadcasting** deb ataladi: siz odatdagi hodisa (event) yozasiz, uni `ShouldBroadcast` bilan belgilaysiz — va u WebSocket orqali brauzerga uchadi.

```
Controller → Event (ShouldBroadcast) → Queue → WebSocket server → Echo (brauzer)
```

---

## Drayverni tanlash

| Drayver | Qachon |
| --- | --- |
| **Reverb** | Laravel'ning o'z WebSocket serveri, o'z serveringizda ishlaydi — standart tanlov |
| **Pusher** | Boshqariladigan bulut xizmati (server sozlash shart emas, pullik) |
| **Ably** | Pusher muqobili |
| `log` / `null` | Lokal ishlab chiqish va testlar |

O'rnatish:

```shell
php artisan install:broadcasting --reverb
```

Bu buyruq: `laravel/reverb` va `laravel-echo` + `pusher-js` ni o'rnatadi, `config/broadcasting.php` va `routes/channels.php` yaratadi, `.env` ga kalitlarni qo'shadi, `resources/js/echo.js` ni sozlaydi.

```shell
php artisan reverb:start          # WebSocket server
php artisan queue:work            # broadcast joblarini ishlovchi worker
npm run dev
```

> **Muhim:** broadcast hodisalari **navbat orqali** yuboriladi. Worker ishlamasa, hech narsa brauzerga yetib bormaydi. Bu eng ko'p uchraydigan "nega ishlamayapti" sababi.

---

## Hodisani broadcast qilish

```shell
php artisan make:event MessageSent
```

```php
namespace App\Events;

use App\Models\Message;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class MessageSent implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public function __construct(public Message $message) {}

    /** @return array<int, \Illuminate\Broadcasting\Channel> */
    public function broadcastOn(): array
    {
        return [new PrivateChannel("chat.{$this->message->chat_id}")];
    }

    /** Brauzerda eshitiladigan nom */
    public function broadcastAs(): string
    {
        return 'message.sent';
    }

    /** Aynan nima yuboriladi (butun modelni yubormang!) */
    public function broadcastWith(): array
    {
        return [
            'id' => $this->message->id,
            'body' => $this->message->body,
            'user' => ['id' => $this->message->user_id, 'name' => $this->message->user->name],
            'sent_at' => $this->message->created_at->toIso8601String(),
        ];
    }

    /** Shartli broadcast */
    public function broadcastWhen(): bool
    {
        return $this->message->chat->is_active;
    }
}
```

```php
MessageSent::dispatch($message);
broadcast(new MessageSent($message));
broadcast(new MessageSent($message))->toOthers();   // yuborgan odamga qaytarmaydi
```

> **`broadcastWith` nega kerak?** Standart holatda hodisaning barcha `public` xossalari yuboriladi — ya'ni butun model, shu jumladan maxfiy ustunlar ham. Har doim aniq ro'yxat yozing.

> **Tranzaksiya bilan:** `ShouldDispatchAfterCommit` interfeysini qo'shing, aks holda broadcast `commit` dan oldin ketib, mijoz bazada hali yo'q yozuvni so'rashi mumkin ([25-bob](25-navbatlar.md)).

---

## Kanal turlari va ruxsat

| Kanal | Kim eshitadi | Klass |
| --- | --- | --- |
| Public | Hamma | `Channel` |
| Private | Ruxsati borlar | `PrivateChannel` |
| Presence | Ruxsati borlar + kim onlayn ekani ko'rinadi | `PresenceChannel` |

Ruxsat `routes/channels.php` da:

```php
use App\Models\Chat;
use App\Models\User;
use Illuminate\Support\Facades\Broadcast;

Broadcast::channel('chat.{chatId}', function (User $user, int $chatId) {
    return Chat::find($chatId)?->hasParticipant($user) ?? false;
});

// presence kanal: massiv qaytarsangiz — o'sha ma'lumot boshqa ishtirokchilarga ko'rinadi
Broadcast::channel('room.{roomId}', function (User $user, int $roomId) {
    return $user->canJoinRoom($roomId)
        ? ['id' => $user->id, 'name' => $user->name]
        : false;
});
```

Bu — WebSocket dunyosidagi **avtorizatsiya** ([22-bob](22-avtorizatsiya.md) bilan bir xil mantiq). Mijoz kanalga ulanmoqchi bo'lganda Laravel shu closure'ni chaqiradi.

> **Xavfsizlik qoidasi:** chat, bildirishnoma, buyurtma holati — har doim `PrivateChannel`. Public kanalga faqat hammaga ochiq ma'lumotni yuboring (masalan umumiy onlayn foydalanuvchilar soni).

---

## Brauzer tomoni (Echo)

```js
// resources/js/echo.js — install:broadcasting yaratadi
import Echo from 'laravel-echo';
import Pusher from 'pusher-js';

window.Pusher = Pusher;

window.Echo = new Echo({
    broadcaster: 'reverb',
    key: import.meta.env.VITE_REVERB_APP_KEY,
    wsHost: import.meta.env.VITE_REVERB_HOST,
    wsPort: import.meta.env.VITE_REVERB_PORT,
    forceTLS: (import.meta.env.VITE_REVERB_SCHEME ?? 'https') === 'https',
    enabledTransports: ['ws', 'wss'],
});
```

```js
// private kanal
Echo.private(`chat.${chatId}`)
    .listen('.message.sent', (e) => {
        console.log(e.body, e.user.name);
    });

// presence kanal
Echo.join(`room.${roomId}`)
    .here((users) => console.log('Hozir onlayn:', users))
    .joining((user) => console.log(user.name, 'qo\'shildi'))
    .leaving((user) => console.log(user.name, 'chiqdi'))
    .listen('.message.sent', (e) => { /* ... */ });

// bildirishnomalar ([28-bob])
Echo.private(`App.Models.User.${userId}`)
    .notification((n) => console.log(n.type));
```

`broadcastAs()` bilan nom bergan bo'lsangiz, `listen()` da nom oldiga **nuqta** qo'yiladi (`.message.sent`) — bu "namespace'siz nom" degani.

React/Vue uchun rasmiy hook'lar ham bor:

```js
import { useEcho, useEchoNotification } from '@laravel/echo-react';
```

---

## Model broadcasting (tez yo'l)

Har bir o'zgarish uchun alohida hodisa yozmasdan:

```php
use Illuminate\Database\Eloquent\BroadcastsEvents;

class Post extends Model
{
    use BroadcastsEvents;

    /** @return array<int, mixed> */
    public function broadcastOn(string $event): array
    {
        return match ($event) {
            'deleted' => [],
            default => [$this, $this->user],
        };
    }
}
```

Endi `created`, `updated`, `deleted`, `trashed`, `restored` avtomatik broadcast bo'ladi:

```js
Echo.private(`App.Models.Post.${postId}`)
    .listen('.PostUpdated', (e) => console.log(e.model));
```

> Qulay, lekin **nazorat kam**: nima yuborilishini aniq boshqarish kerak bo'lsa, oddiy hodisa yozing.

---

## Serverdan mijozga xabar yuborishning boshqa yo'llari

| Usul | Qachon |
| --- | --- |
| **WebSocket (Reverb)** | Ikki tomonlama, chat, jonli panel |
| **SSE** (`response()->eventStream()`) | Faqat serverdan mijozga oqim: AI javobi, progress bar |
| **Polling** (har N soniyada so'rov) | Juda kam yangilanadigan ma'lumot; eng oddiy |

SSE misoli:

```php
Route::get('/stream', function () {
    return response()->eventStream(function () {
        foreach ($chunks as $chunk) {
            yield $chunk;
        }
    });
});
```

---

## Productionda

- Reverb alohida jarayon sifatida ishlaydi (supervisor/systemd): `php artisan reverb:start --host=0.0.0.0 --port=8080`
- Nginx orqali `wss://` uchun proksi va TLS sozlanadi (`Upgrade`/`Connection` sarlavhalari).
- Bir nechta server bo'lsa — Reverb'ni Redis bilan gorizontal masshtablash (`REVERB_SCALING_ENABLED=true`).
- Broadcast navbati uchun alohida queue ajrating: `broadcast` nomli navbat va o'z worker'i.
- Monitoring: `php artisan reverb:restart` deploy'dan keyin.

---

## Testlash

```php
use Illuminate\Support\Facades\Event;

Event::fake();

// ...

Event::assertDispatched(MessageSent::class);
```

Kanal avtorizatsiyasini tekshirish:

```php
actingAs($user)
    ->postJson('/broadcasting/auth', ['channel_name' => "private-chat.{$chat->id}", 'socket_id' => '123.456'])
    ->assertOk();
```

---

## Amaliyot

1. `php artisan install:broadcasting --reverb` ni bajaring va `reverb:start` + `queue:work` + `npm run dev` ni birga ishga tushiring.
2. `MessageSent` hodisasini yarating, `broadcastWith()` bilan aniq maydonlarni yuboring.
3. `routes/channels.php` da `chat.{chatId}` uchun ruxsatni yozing va begona foydalanuvchi ulana olmasligini tekshiring.
4. Brauzerda `Echo.private(...).listen('.message.sent', ...)` bilan xabarni qabul qiling.
5. `queue:work` ni to'xtatib, xabar yetib bormasligini ko'ring — bog'liqlikni his qilish uchun.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/broadcasting>
- <https://laravel.com/docs/13.x/reverb>
- <https://laravel.com/docs/13.x/notifications#broadcast-notifications>

---

[← Oldingi: Symfony → Laravel lug'ati](34-symfony-laravel-lugat.md) · [Mundarija](README.md) · [Keyingi: Ko'p tillilik →](36-koptillilik-va-mintaqa.md)
