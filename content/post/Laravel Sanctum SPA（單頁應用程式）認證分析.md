---
title: "Laravel Sanctum SPA（單頁應用程式）認證分析"
date: 2020-10-24T16:15:52+08:00
draft: false
categories:
    - Laravel
tags:
    - PHP
    - Laravel
    - Sanctum
    - SPA
---

# 前置知識

Sanctum 是 Laravel 提供的輕量化 API 服務認證（Authenticate）解決方案。

以下我們用一個表格來揭示各種不同的 Session Guard Driver 的功能差異：

| Driver   | 套件 | Bearer Token | Session Cookie | Token Scope |
| :--:     | :-- | :--: | :--: | :--: |
| Session  | 內建 | ❌ | ✅ | ❌ | 
| Token    | 內建 | ✅ | ❌ | ❌ | 
| Sanctum  | `laravel/sanctum`  | ✅ | ✅ | ✅ | 
| Passport | `laravel/passport` | ✅ | ❌ | ✅ |

## Session Guard Driver 的應用情境

- Session
    - Laravel 預設的認證方式
    - 常用於 Server Side Rendering 應用程式
- Token
    - Laravel API 預設的認證方式，罕用
    - 每個用戶僅能有一組 Token
    - 無法設定 Token 過期時間
    - 無法設定 Token 應用範圍（Scope）
- Sanctum
    - Laravel 輕量化的認證方式
    - 每個用戶能有多組 Token
    - 可以設定 Token 過期時間與應用範圍
    - **可用於 SPA 的服務，且不需要 Token 作為溝通媒介**
    - **可核發單獨的 Token 用於其它服務（例如手機 APP）**
        - 這部份類似於 GitHub 提供的 Personal Access Token，也可以做成開發者服務
- Passport
    - Laravel 官方提供的 OAuth2 服務
    - 每個用戶能有多組 Token
    - 可以設定 Token 過期時間與應用範圍
    - [可以為 Token 做詳細的分類，以應用在各種不同的情境](https://oauth2.thephpleague.com/authorization-server/which-grant/)

# 緣起

會想到要寫這篇文章，主要是被 Sanctum 的 **不需要 Token 就可以讓 SPA 認證** 的功能所吸引。

實務上，將 Laravel 作為一個 API Service，並且用 React 製作一個 SPA，它們分屬於兩個不同的 Repository，這種做法一般稱為前後端分離。

然而我很疑惑，究竟 React SPA 是如何用 Sanctum 與 Laravel API 進行認證的。

# 實驗

## 環境建立

### 新建 Laravel 專案

```
$ laravel new test-sanctum
```

### 設定 Database

- `.env` 中 `DB_CONNECTION=sqlite`
- 移除 `.env` 中 `DB_HOST`, `DB_PORT`, `DB_DATABASE`, `DB_USERNAME` 及 `DB_PASSWORD`
- 建立 `database/database.sqlite` 的空檔案

### 安裝 Sanctum

- `composer require laravel/sacntum`
- `php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"`
- 將 `config/auth.php` 中 `guards.api.driver` 改為 `sanctum`
- 將 `config/cors.php` 中 `paths` 加入 `sanctum/csrf-cookie`
- 在 `.env` 中加入 `SANCTUM_STATEFUL_DOMAINS=127.0.0.1:8000,localhost:8000`
- 在 `app/Http/Kernel.php` 中 `$middlewareGroups` 的 `api` 中加入 `Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class`

### 執行 Migration 與建立用戶

```
$ php artisan migrate
$ php artisan tinker
(tinker) $ User::factory()->create(['email' => 'test@sanctum.test'])
```

## 基礎功能

### 登入

```php
<?php
// routes/web.php

Route::post('login', function (Request $request) {
    return Auth::attempt($request->only('email', 'password'))
        ? 'auth.success' : 'auth.failed';
});
```

### 顯示使用者資訊

```php
<?php
// routes/api.php

Route::middleware('api:sanctum')
    ->get('user', fn(Request $request) => $request->user());
```

## 測試：用內建的 axios 測試認證功能

### 安裝所需套件與編譯

```
$ npm install
```

### 設定 axios

```js
// resources/js/bootstrap.js

window.axios = require('axios');

window.axios.defaults.headers.common['X-Requested-With'] = 'XMLHttpRequest';
window.axios.defaults.withCredentials = true; // 這一行是新增的
```

- 記得程式更改後要執行 `npm run dev`

### 建立一個空白頁面

將預設的 `resources/views/welcom.blade.js` 改成以下內容

```html
<html>
    <head></head>
    <body>
    <script src="{{ url('js/app.js') }}"></script>
    </body>
</html>
```

### 測試登入與獲取使用者資訊

執行 `php artisan serve` 之後，開啟 `http://localhost:8000`

使用開發者工具執行以下程式並觀察結果

```js
await axios.get('sanctum/csrf-cookie');
await axios.post('login', {email: 'test@sanctum.test', password: 'password'});
await axios.get('api/user');
```

理論上應該要能正常看到執行結果

## 測試：用 curl 進行測試

### 請求 CSRF Cookie

```
$ curl -i -X GET http://localhost:8000/sanctum/csrf-cookie
```

註：此處應會取得 `XSRF-TOKEN` 及 `laravel_session` 這兩個 Cookie，請記錄下來

### Login

```
$ curl -i -X POST http://localhost:8000/login
```

- 註：此處建議帶上 `-H 'Accept: application/json` 以顯示 JSON Type 的 Error Message
- 註2：此處要加上 `-H 'X-XSRF-TOKEN: {token}'`，其中 `{token}` 是上一步的 `XSRF-TOKEN`
    - `XSRF-TOKEN` 的結尾處可能會是 `%3D`，這邊要改成 `=`
- 註3：此處要加上 `--cookie "{XSRF-TOKEN}&{laravel_session}"`

### 取得用戶

```
$ curl -i -X GET http://localhost:8000/api/user
```

- 註：此處要帶上 `-H 'Accept: application/json` 以顯示 JSON Type 的 Error
- 註2：要加上 `--cookie "{XSRF-TOKEN}&{laravel_session}"`
- **註3：要加上 `-H 'Origin: http://localhost:8000` 或 `-H 'Referer: http://localhost:8000/` 才能正常取得資料**

# 實驗結果

從 curl 的實驗結論我們得知：**要加上 Origin 或 Referer Header 才能正常取得資料**

這是因為 Laravel 在判斷是否為 SPA 是以這兩個 Header 作為依據，我們可以在 `Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class` 查閱相關的程式碼

```php
<?php
    /**
     * Middleware 的進入點
     */ 
    public function handle($request, $next)
    {
        $this->configureSecureCookieSessions();

        return (new Pipeline(app()))->send($request)->through(static::fromFrontend($request) ? [ // 確認是否為 SPA 打過來的請求
            function ($request, $next) {
                $request->attributes->set('sanctum', true);

                return $next($request);
            },
            // 啟動 Session 相關的 Middlewares
            config('sanctum.middleware.encrypt_cookies', \Illuminate\Cookie\Middleware\EncryptCookies::class),
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            config('sanctum.middleware.verify_csrf_token', \Illuminate\Foundation\Http\Middleware\VerifyCsrfToken::class),
        ] : [])->then(function ($request) use ($next) {
            return $next($request);
        });
    }

    public static function fromFrontend($request)
    {
        // $domain 的來源是 Referer Header 或 Origin Header
        $domain = $request->headers->get('referer') ?: $request->headers->get('origin');

        // 過濾 $domain，把前方的 Protocol 去掉並且確保一定會有 / 結尾
        $domain = Str::replaceFirst('https://', '', $domain);
        $domain = Str::replaceFirst('http://', '', $domain);
        $domain = Str::endsWith($domain, '/') ? $domain : "{$domain}/";

        $stateful = array_filter(config('sanctum.stateful', []));

        // 確認 $domain 是否符合 config/sacntum.php stateful 中設定的 Pattern
        return Str::is(Collection::make($stateful)->map(function ($uri) {
            return trim($uri).'/*';
        })->all(), $domain);
    }
```