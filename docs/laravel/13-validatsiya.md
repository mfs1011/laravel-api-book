# 13 — Validatsiya

[← Oldingi: So'rov va javob](12-sorov-va-javob.md) · [Mundarija](README.md) · [Keyingi: Ma'lumotlar bazasi →](14-malumotlar-bazasi.md)

---

## Eng oddiy usul: `$request->validate()`

```php
public function store(Request $request)
{
    $validated = $request->validate([
        'title' => ['required', 'string', 'max:255'],
        'body' => ['required', 'string'],
        'published_at' => ['nullable', 'date'],
        'tags' => ['array', 'max:5'],
        'tags.*' => ['string', 'max:30'],
    ]);

    return Post::create($validated);
}
```

Nima bo'ladi:

- Qoidalar bajarilsa — `$validated` faqat **tekshirilgan** maydonlarni qaytaradi.
- Bajarilmasa — `ValidationException` tashlanadi va ilova **avtomatik** javob qaytaradi:
  - brauzer so'rovi bo'lsa — oldingi sahifaga redirect + xatolar sessiyada;
  - JSON so'rov bo'lsa — **422** status va JSON xatolar.

Kontroller kodida `if ($validator->fails())` yozish shart emas — bu Laravel'ning asosiy yengilligi.

### 422 javob formati

```json
{
  "message": "The title field is required. (and 1 more error)",
  "errors": {
    "title": ["The title field is required."],
    "body": ["The body field is required."]
  }
}
```

Frontend shu formatga tayanadi, shuning uchun uni o'zgartirmaslik ma'qul.

> **Muhim:** `$validated` — bu "oq ro'yxat". `Post::create($request->all())` yozmang: foydalanuvchi so'roviga `is_admin=1` qo'shib yuborishi mumkin. Har doim `validated()` natijasini ishlating ([16-bob](16-eloquent-asoslari.md), mass assignment).

---

## Form Request — tavsiya etiladigan yo'l

```shell
php artisan make:request StorePostRequest
```

```php
namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        return $this->user()->can('create', Post::class);
    }

    /**
     * @return array<string, \Illuminate\Contracts\Validation\ValidationRule|array<mixed>|string>
     */
    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'max:255'],
            'slug' => ['required', 'string', Rule::unique('posts', 'slug')],
            'body' => ['required', 'string', 'min:20'],
            'status' => ['required', Rule::enum(PostStatus::class)],
        ];
    }

    /** @return array<string, string> */
    public function messages(): array
    {
        return [
            'title.required' => 'Sarlavha majburiy.',
            'body.min' => 'Matn kamida :min belgidan iborat bo\'lsin.',
        ];
    }

    /** @return array<string, string> */
    public function attributes(): array
    {
        return ['body' => 'maqola matni'];
    }

    protected function prepareForValidation(): void
    {
        $this->merge(['slug' => str($this->title)->slug()->value()]);
    }
}
```

Kontrollerda faqat tipni ko'rsatasiz:

```php
public function store(StorePostRequest $request)
{
    $post = Post::create($request->validated());

    return PostResource::make($post);
}
```

Validatsiya **kontroller metodi chaqirilishidan oldin** bajariladi. `authorize()` `false` qaytarsa — **403**.

> **Symfony bilan solishtirish:** Form Request ≈ Symfony Form + Validator constraint'lari, lekin ancha yengil: HTML render qismi yo'q, faqat qoidalar va ruxsat.

Yangilash uchun alohida Request yozing (`UpdatePostRequest`) — chunki qoidalar farq qiladi (`sometimes`, `unique` da o'zini istisno qilish).

---

## Ko'p ishlatiladigan qoidalar

```php
'required', 'nullable', 'sometimes', 'filled', 'present',
'string', 'integer', 'numeric', 'boolean', 'array', 'json',
'min:3', 'max:255', 'between:1,10', 'size:5', 'digits:9',
'email', 'url', 'uuid', 'ulid', 'ip', 'timezone', 'regex:/^\+998\d{9}$/',
'date', 'date_format:Y-m-d', 'after:today', 'before_or_equal:2030-01-01',
'in:draft,published', 'not_in:banned',
'confirmed',                 // password + password_confirmation
'same:password', 'different:old_password',
'exists:posts,id', 'unique:users,email',
'image', 'mimes:jpg,png,pdf', 'max:2048',   // KB
'accepted',                  // shartlarga rozilik
```

### `sometimes` — PATCH so'rovlar uchun

```php
'title' => ['sometimes', 'required', 'string', 'max:255'],
```

Ma'nosi: "kalit **kelgan bo'lsa**, u bo'sh bo'lmasin". Kalit kelmasa qoida tekshirilmaydi. PATCH/qisman yangilash uchun aynan shu kerak.

### `Rule` klassi — moslashuvchan qoidalar

```php
use Illuminate\Validation\Rule;

'email' => ['required', 'email', Rule::unique('users')->ignore($user->id)],
'status' => [Rule::in(['draft', 'published'])],
'role' => [Rule::enum(UserRole::class)],
'start_date' => [Rule::date()->format('Y-m-d')],
'slug' => [Rule::unique('posts')->withoutTrashed()],
'author_id' => [Rule::exists('users', 'id')->where('is_active', true)],
```

### Shartli qoidalar

```php
'reason' => ['required_if:status,rejected'],
'company' => ['required_with:tax_id'],
'card' => ['exclude_unless:payment,card', 'required'],
```

Yoki dinamik:

```php
use Illuminate\Validation\Validator;

public function withValidator(Validator $validator): void
{
    $validator->sometimes('discount', 'required|numeric', fn ($input) => $input->total > 1_000_000);
}
```

### Parol qoidalari

```php
use Illuminate\Validation\Rules\Password;

'password' => ['required', 'confirmed', Password::min(8)
    ->letters()
    ->mixedCase()
    ->numbers()
    ->symbols()
    ->uncompromised(),   // ma'lum bo'lgan sizib chiqqan parollar bazasiga qarshi tekshiradi
],
```

---

## Massivlar va ichma-ich ma'lumot

```php
'items' => ['required', 'array', 'min:1'],
'items.*.id' => ['required', 'integer', 'exists:products,id'],
'items.*.qty' => ['required', 'integer', 'min:1'],
'user.profile.phone' => ['required', 'regex:/^\+998\d{9}$/'],
```

---

## Maxsus qoida yozish

```shell
php artisan make:rule UzbekPhone
```

```php
namespace App\Rules;

use Closure;
use Illuminate\Contracts\Validation\ValidationRule;

class UzbekPhone implements ValidationRule
{
    public function validate(string $attribute, mixed $value, Closure $fail): void
    {
        if (! preg_match('/^\+998\d{9}$/', (string) $value)) {
            $fail('The :attribute maydoni +998XXXXXXXXX formatida bo\'lishi kerak.');
        }
    }
}
```

```php
'phone' => ['required', new UzbekPhone],
```

Oddiy holat uchun closure ham yetarli:

```php
'title' => ['required', function (string $attribute, mixed $value, Closure $fail) {
    if (str_contains(strtolower($value), 'reklama')) {
        $fail('Sarlavhada reklama bo\'lmasin.');
    }
}],
```

---

## Qo'lda validator yaratish

```php
use Illuminate\Support\Facades\Validator;

$validator = Validator::make($data, [
    'title' => 'required',
]);

if ($validator->fails()) {
    return response()->json(['errors' => $validator->errors()], 422);
}

$validated = $validator->validated();
```

Qo'shimcha tekshiruv:

```php
$validator->after(function ($validator) use ($order) {
    if ($order->isClosed()) {
        $validator->errors()->add('order', 'Buyurtma yopilgan.');
    }
});
```

---

## Xato xabarlarini tarjima qilish

```shell
php artisan lang:publish
```

`lang/uz/validation.php` yarating va xabarlarni tarjima qiling. Tilni `.env` da tanlaysiz:

```ini
APP_LOCALE=uz
APP_FALLBACK_LOCALE=en
```

---

## Amaliyot

1. `php artisan make:request StorePostRequest` yarating, qoidalarni yozing va kontrollerga ulang.
2. `curl -X POST http://127.0.0.1:8000/api/posts -H "Accept: application/json" -d '{}'` yuboring va 422 javob tuzilmasini ko'ring.
3. `prepareForValidation()` da `slug` ni sarlavhadan avtomatik yarating.
4. `UzbekPhone` qoidasini yozing va uni `phone` maydoniga qo'llang.
5. `UpdatePostRequest` yarating: `title` da `sometimes`, `slug` da `Rule::unique('posts')->ignore($this->post)` ishlating.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/validation>
- <https://laravel.com/docs/13.x/validation#available-validation-rules> — qoidalarning to'liq ro'yxati

---

[← Oldingi: So'rov va javob](12-sorov-va-javob.md) · [Mundarija](README.md) · [Keyingi: Ma'lumotlar bazasi →](14-malumotlar-bazasi.md)
