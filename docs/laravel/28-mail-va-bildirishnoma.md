# 28 — Mail va bildirishnomalar

[← Oldingi: Kesh va fayllar](27-kesh-va-fayllar.md) · [Mundarija](README.md) · [Keyingi: Testlash →](29-testlash.md)

---

## Mail sozlash

`.env`:

```ini
MAIL_MAILER=log                    # lokalda: xatlar storage/logs/laravel.log ga yoziladi
# MAIL_MAILER=smtp
# MAIL_HOST=smtp.mailtrap.io
# MAIL_PORT=2525
# MAIL_USERNAME=...
# MAIL_PASSWORD=...
MAIL_FROM_ADDRESS="hello@example.com"
MAIL_FROM_NAME="${APP_NAME}"
```

Drayverlar: `smtp`, `ses`, `postmark`, `resend`, `sendmail`, `log`, `array` (testlar uchun), `failover` (biri ishlamasa ikkinchisi).

> Ichida `symfony/mailer` ishlaydi — Symfony'dan tanish komponent.

---

## Mailable

```shell
php artisan make:mail OrderShipped --markdown=mail.orders.shipped
```

```php
namespace App\Mail;

use App\Models\Order;
use Illuminate\Bus\Queueable;
use Illuminate\Mail\Mailable;
use Illuminate\Mail\Mailables\{Content, Envelope, Attachment};
use Illuminate\Queue\SerializesModels;

class OrderShipped extends Mailable
{
    use Queueable, SerializesModels;

    public function __construct(public Order $order) {}

    public function envelope(): Envelope
    {
        return new Envelope(
            subject: "Buyurtma #{$this->order->id} jo'natildi",
            replyTo: ['support@example.com'],
        );
    }

    public function content(): Content
    {
        return new Content(
            markdown: 'mail.orders.shipped',
            with: ['trackingUrl' => $this->order->trackingUrl()],
        );
    }

    /** @return array<int, Attachment> */
    public function attachments(): array
    {
        return [
            Attachment::fromStorageDisk('public', "invoices/{$this->order->id}.pdf")
                ->as('hisob-faktura.pdf')
                ->withMime('application/pdf'),
        ];
    }
}
```

Shablon (`resources/views/mail/orders/shipped.blade.php`):

```blade
<x-mail::message>
# Buyurtmangiz yo'lda

Salom, {{ $order->user->name }}!

Buyurtma raqami: **#{{ $order->id }}**

<x-mail::button :url="$trackingUrl">
Kuzatish
</x-mail::button>

Rahmat,<br>
{{ config('app.name') }}
</x-mail::message>
```

Markdown shablonlari tayyor, moslashtirilgan dizayn bilan keladi — HTML yozish shart emas.

---

## Yuborish

```php
use Illuminate\Support\Facades\Mail;

Mail::to($user)->send(new OrderShipped($order));
Mail::to($user)->cc($manager)->bcc($archive)->send(new OrderShipped($order));
Mail::to('a@b.uz')->queue(new OrderShipped($order));            // navbatga
Mail::to($user)->later(now()->addMinutes(10), new OrderShipped($order));
Mail::mailer('ses')->to($user)->send(...);
```

Mailable'ni har doim navbatga qo'yish:

```php
class OrderShipped extends Mailable implements ShouldQueue {}
```

Brauzerda ko'rish (yuborilmaydi):

```php
Route::get('/mail-preview', fn () => new OrderShipped(Order::first()));
```

---

## Bildirishnomalar (Notifications)

Mailable — faqat xat. **Notification** esa bitta xabarni bir nechta kanal orqali yuboradi: email, SMS, Slack, bazaga yozish, broadcast.

```shell
php artisan make:notification InvoicePaid
```

```php
namespace App\Notifications;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Notifications\Messages\MailMessage;
use Illuminate\Notifications\Notification;

class InvoicePaid extends Notification implements ShouldQueue
{
    use Queueable;

    public function __construct(public Invoice $invoice) {}

    /** @return array<int, string> */
    public function via(object $notifiable): array
    {
        return ['mail', 'database'];
    }

    public function toMail(object $notifiable): MailMessage
    {
        return (new MailMessage)
            ->subject('To\'lov qabul qilindi')
            ->greeting("Salom, {$notifiable->name}!")
            ->line("Hisob-faktura #{$this->invoice->id} to'landi.")
            ->action('Ko\'rish', route('invoices.show', $this->invoice))
            ->line('Rahmat!');
    }

    /** @return array<string, mixed> */
    public function toArray(object $notifiable): array
    {
        return [
            'invoice_id' => $this->invoice->id,
            'amount' => $this->invoice->amount,
        ];
    }
}
```

Yuborish:

```php
$user->notify(new InvoicePaid($invoice));                       // Notifiable trait orqali

use Illuminate\Support\Facades\Notification;
Notification::send($users, new InvoicePaid($invoice));
Notification::route('mail', 'a@b.uz')->notify(new InvoicePaid($invoice));
```

`User` modelida `Notifiable` trait'i allaqachon bor (skeletonda).

### `database` kanali

```shell
php artisan make:notifications-table && php artisan migrate
```

```php
$user->notifications;                  // hammasi
$user->unreadNotifications;            // o'qilmaganlar
$user->unreadNotifications->markAsRead();
$notification->markAsRead();
```

API'da:

```php
Route::get('/notifications', fn (Request $request) =>
    $request->user()->notifications()->paginate(20)
);
```

### Kanalni sozlash

Foydalanuvchi qaysi manzilga olsin:

```php
class User extends Authenticatable
{
    public function routeNotificationForMail(): string
    {
        return $this->notification_email ?? $this->email;
    }
}
```

O'z kanalingizni yozish (masalan SMS):

```php
namespace App\Notifications\Channels;

class SmsChannel
{
    public function __construct(private SmsSender $sender) {}

    public function send(object $notifiable, Notification $notification): void
    {
        $message = $notification->toSms($notifiable);

        $this->sender->send($notifiable->phone, $message);
    }
}
```

```php
public function via(object $notifiable): array
{
    return [SmsChannel::class];
}
```

---

## Sozlash: qachon Mailable, qachon Notification

| Holat | Yechim |
| --- | --- |
| Faqat email, murakkab shablon/ilova fayllari bilan | **Mailable** |
| Bir xabar, bir nechta kanal (email + baza + SMS) | **Notification** |
| Foydalanuvchi ilova ichida ko'radigan xabar | **Notification** + `database` kanali |

---

## Testlash

```php
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Notification;

Mail::fake();
Notification::fake();

// ... amalni bajaramiz ...

Mail::assertSent(OrderShipped::class);
Mail::assertSent(OrderShipped::class, fn ($mail) => $mail->hasTo($user->email));
Mail::assertNotSent(OrderCancelled::class);
Mail::assertSentCount(1);

Notification::assertSentTo($user, InvoicePaid::class);
Notification::assertNothingSent();
```

`phpunit.xml` da `MAIL_MAILER=array` bo'lgani uchun testlar hech qachon haqiqiy xat yubormaydi.

---

## Amaliyot

1. `php artisan make:mail WelcomeMail --markdown=mail.welcome` yarating va shablonni to'ldiring.
2. `Route::get('/mail-preview', fn () => new WelcomeMail(User::first()))` bilan brauzerda ko'ring.
3. `MAIL_MAILER=log` bilan yuboring va `storage/logs/laravel.log` da xat matnini toping.
4. `InvoicePaid` bildirishnomasini `mail` + `database` kanallari bilan yarating va `$user->unreadNotifications` ni tekshiring.
5. Testda `Notification::fake()` + `assertSentTo` yozing.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/mail>
- <https://laravel.com/docs/13.x/notifications>

---

[← Oldingi: Kesh va fayllar](27-kesh-va-fayllar.md) · [Mundarija](README.md) · [Keyingi: Testlash →](29-testlash.md)
