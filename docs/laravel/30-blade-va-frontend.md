# 30 — Blade va frontend

[← Oldingi: Testlash](29-testlash.md) · [Mundarija](README.md) · [Keyingi: Amaliy loyiha →](31-amaliy-loyiha-api.md)

---

> Bu bob API qurish uchun majburiy emas. Lekin admin panel, xat shablonlari yoki oddiy sahifalar kerak bo'lganda Blade'ni bilish foydali.

---

## Blade — Twig'ning Laravel'dagi muqobili

Shablonlar `resources/views/` da, kengaytmasi `.blade.php`. Blade kompilyatsiya qilinadi (`storage/framework/views/`) va oddiy PHP tezligida ishlaydi.

```blade
{{-- resources/views/posts/index.blade.php --}}
@extends('layouts.app')

@section('title', 'Postlar')

@section('content')
    <h1>Postlar</h1>

    @forelse ($posts as $post)
        <article>
            <h2>{{ $post->title }}</h2>
            <p>{!! $post->html_body !!}</p>       {{-- himoyalanmagan chiqish --}}
        </article>
    @empty
        <p>Post yo'q.</p>
    @endforelse

    {{ $posts->links() }}
@endsection
```

| Blade | Twig |
| --- | --- |
| `{{ $var }}` | `{{ var }}` (ikkalasi ham HTML'dan himoyalaydi) |
| `{!! $var !!}` | `{{ var\|raw }}` |
| `@extends` / `@section` / `@yield` | `{% extends %}` / `{% block %}` |
| `@include('partial')` | `{% include %}` |
| `@foreach` / `@if` | `{% for %}` / `{% if %}` |
| `@php ... @endphp` | — (Twig'da PHP yozilmaydi) |

Asosiy direktivalar:

```blade
@if ($user->is_admin) ... @elseif (...) ... @else ... @endif
@unless ($user) ... @endunless
@isset($name) ... @endisset
@empty($items) ... @endempty
@auth ... @endauth
@guest ... @endguest
@can('update', $post) ... @endcan
@foreach ($items as $item)
    {{ $loop->index }} {{ $loop->first }} {{ $loop->last }} {{ $loop->count }}
@endforeach
@switch($status) @case('draft') ... @break @default ... @endswitch
@csrf
@method('PUT')
@vite(['resources/css/app.css', 'resources/js/app.js'])
@json($data)
```

---

## Komponentlar

```shell
php artisan make:component Alert
```

```blade
{{-- resources/views/components/alert.blade.php --}}
@props(['type' => 'info', 'title'])

<div {{ $attributes->merge(['class' => "alert alert-{$type}"]) }}>
    @isset($title)
        <strong>{{ $title }}</strong>
    @endisset

    {{ $slot }}
</div>
```

```blade
<x-alert type="danger" title="Xato">
    Ma'lumot saqlanmadi.
</x-alert>
```

Klass qismisiz, faqat shablonli komponent ham bo'ladi (anonymous component) — shunchaki `resources/views/components/` ga fayl qo'ying.

---

## Ma'lumot uzatish

```php
return view('posts.index', ['posts' => $posts]);
return view('posts.index')->with('posts', $posts);
return view('posts.index', compact('posts'));
```

Barcha sahifalarga umumiy ma'lumot:

```php
// AppServiceProvider::boot()
View::share('appVersion', '1.0');

View::composer('partials.sidebar', function ($view) {
    $view->with('categories', Category::all());
});
```

---

## Vite, Tailwind va assetlar

Bu loyihada: **Vite 8** + **Tailwind CSS 4**.

```js
// vite.config.js
import { defineConfig } from 'vite';
import laravel from 'laravel-vite-plugin';
import tailwindcss from '@tailwindcss/vite';

export default defineConfig({
    plugins: [
        laravel(['resources/css/app.css', 'resources/js/app.js']),
        tailwindcss(),
    ],
});
```

```css
/* resources/css/app.css — Tailwind 4 uslubi */
@import "tailwindcss";
```

Buyruqlar:

```shell
npm run dev       # HMR bilan ishlab chiqish serveri
npm run build     # productionga yig'ish
composer run dev  # yoki: php artisan dev (hammasi birga)
```

Shablonda:

```blade
@vite(['resources/css/app.css', 'resources/js/app.js'])
<img src="{{ Vite::asset('resources/images/logo.png') }}">
```

> **"Unable to locate file in Vite manifest" xatosi** — `npm run build` bajarilmagan yoki `npm run dev` ishlamayapti degani.

---

## SPA yoki mobil frontend bilan ishlash

Laravel'ni API sifatida ishlatganda Blade umuman kerak bo'lmasligi mumkin:

| Yondashuv | Qanday |
| --- | --- |
| **Alohida SPA** (Vue/React/Next.js) | Laravel faqat JSON qaytaradi; auth — Sanctum ([21-bob](21-autentifikatsiya.md)) |
| **Inertia.js** | Vue/React komponentlari, lekin API yozmaysiz — kontroller to'g'ridan-to'g'ri props uzatadi |
| **Livewire** | Frontend'ni PHP'da yozasiz, JS deyarli yozilmaydi |
| **Blade + Alpine.js** | Oddiy sahifalar uchun yetarli |

Starter kitlar (React, Vue, Livewire) `laravel new` paytida tanlanadi va tayyor auth bilan keladi.

---

## Amaliyot

1. `resources/views/posts/index.blade.php` yarating, `@forelse` bilan ro'yxat chiqaring.
2. `x-alert` komponentini yarating va ikki xil `type` bilan ishlating.
3. `npm run dev` ni ishga tushiring, `resources/css/app.css` ga Tailwind klassi qo'shib, brauzerda o'zgarish darhol ko'rinishini tekshiring.
4. `@can('update', $post)` bilan tugmani faqat egasiga ko'rsating.

---

## Rasmiy hujjat

- <https://laravel.com/docs/13.x/blade>
- <https://laravel.com/docs/13.x/vite>
- <https://laravel.com/docs/13.x/starter-kits>

---

[← Oldingi: Testlash](29-testlash.md) · [Mundarija](README.md) · [Keyingi: Amaliy loyiha →](31-amaliy-loyiha-api.md)
