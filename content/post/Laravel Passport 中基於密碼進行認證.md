---
title: "Laravel Passport 中基於密碼進行認證"
date: 2019-11-07T12:33:02+08:00
draft: false
categories:
    - Laravel
tags:
    - PHP
    - Laravel
    - Laravel Auth
---
# 前置知識

## Laravel 的認證守衛（Authentication Guard）

Laravel 內建兩種認證守衛 Driver： `session` 及 `token`。

- `session`：
    - 以瀏覽器中的 session cookie 進行用戶認證
    - 預設用於 `routes/web.php` 路由
- `token`：
    - 以存放特定位置的 token 進行用戶認證（依優先順序列出）
        - Query String： 於 URL 加入 `/?api_token={token}` 參數
        - Request Body： 於 Request 中加入 `api_token={token}` 參數，常見於 Form Request
        - Bearer Token： 於 HTTP Header 中加入 `Authorization: Bearer {token}` 的特徵
        - HTTP Basic Auth： 利用 [Basic HTTP Auth 的 Feature](https://www.php.net/manual/en/features.http-auth.php)，取得 `$_SERVER['PHP_AUTH_PW'] = {token}` 值
    - 預設用於 `routes/api.php` 路由

然而，內建的 Token Guard 是相當陽春的：

- 每個用戶只能存放一組 Token
- 無法設定 Token 的時效性
- 無法設定 Token 的可存取範圍

一般而言，[TokenGuard.php](https://github.com/laravel/framework/blob/6.x/src/Illuminate/Auth/TokenGuard.php) 僅適合作為如何擴展 Guard 的官方範例。

> 註：可以在官方文件中的 [Authentication # Add Custom Guards](https://laravel.com/docs/6.x/authentication#adding-custom-guards) 學習如何擴展 Guard

# 實作

## 安裝並設定 Laravel Passport

直接參考官方文件中的 [Passport # Installation](https://laravel.com/docs/6.x/passport#installation) 進行安裝

## 實作「註冊」功能

相較於 Session Guard（預設）的註冊流程，實作 Passport 註冊 API 時會有一些不同

- Password 不需要 `confirm` 驗證
- 註冊完成後，不需要（也不能夠）在註冊後直接登入
    - 「登入」這個功能僅在實現了 `Illuminate\Contracts\Auth\StatefulGuard` 的 Guard 能夠使用，Passport Guard 及 Token Guard 皆未實現
- 註冊完畢後，僅回傳成功與否的訊息

具備上述前提後，我們可以直接 Override 預設的 `App\Http\Controllers\Auth\RegisterController`

```php
<?php

namespace App\Http\Controllers\Auth;

// 這邊省略了一些引入，若有需要請自己添加

class RegisterController extends Controller
{
    use RegistersUsers;

    public function register(Request $request)
    {
        $this->validator($request->all())->validate();

        event(new Registered($user = $this->create($request->all())));

        return $this->registered($request, $user); // 這邊原本是會 redirect 到 home page，在這邊刪掉它
    }

    protected function validator(array $data)
    {
        return Validator::make($data, [
            'name' => ['required', 'string', 'max:255'],
            'email' => ['required', 'string', 'email', 'max:255', 'unique:users'],
            'password' => ['required', 'string', 'min:8'], // 這邊原本存在 confirm 的規則，在這邊刪掉它
        ]);
    }

    protected function registered(Request $request, $user): array
    {
        // 因為 Laravel 會自行序列化（Serialize）array 為 json，所以這邊直接 return array 是可以的
        // 這邊會 return 一個給前端使用的 message，及剛剛註冊 user 的一些資訊
        return [
            'message' => __('User registered.'),
            'data' => $user,
        ];
    }
}
```

## 實作「登入」功能

因為 Passport Guard 並未實現 `Illuminate\Contracts\Auth\StatefulGuard`，所以並不存在 `Auth::guard('passport')->login($user)` 這樣的方法。

我們需要重新思考「登入」對於 Passport 的意義為何：使用 Email 及密碼取得可以使用的 Access Token，此處的 Access Token 是 Password Grant Token。

### 讓 User Model 能夠產生 Password Grant Token

```php
<?php

namespace App;

class User extends Authenticatable
{
    use HasApiTokens; // Passport 的必要設定

    // 利用 Eloquent accessor 這個 Feature 取得 accessToken 值
    // $this->accessToken 是定義於 HasApiTokens 中的 attribute
    public function getAccessTokenAttribute(): ?string
    {
        return $this->accessToken;
    }

    // 利用定義於 HasApiTokens 中的 $this->createToken() 建立 Access Token
    // 利用 Eloquent serialization 的 appends 臨時加入 access_token 這個 attribute
    public function withCreatedToken(string $grantType = 'password_client', array $scopes = []): self
    {
        $this->accessToken = $this->createToken($grantType, $scopes)->accessToken;
        $this->append('access_token');

        return $this;
    }
}
```

### 改寫預設的 LoginController 使其符合邏輯

```php
<?php

namespace App\Http\Controllers\Auth;

class LoginController extends Controller
{
    use AuthenticatesUsers;

    public function attemptLogin(Request $request)
    {
        $user = $this->findUser($credentials = $this->credentials($request));

        // 當使用者不存在時，登入失敗
        if (! $user) {
            return false;
        }

        // When user password is not matched by hash, attempt failed.
        if (! Hash::check($credentials['password'], $user->getAuthPassword())) {
            return false;
        }

        // It will generate access_token and refresh_token when sending login response.
        return true;
    }

    protected function sendLoginResponse(Request $request)
    {
        // 因為在 attemptLogin 已經確認 $this->findUser() 不為 NULL，所以這邊可以確認 $user 就是 User Model
        $user = $this->findUser($this->credentials($request))->withCreatedToken();

        // 清理登入時產生的 Throttle
        $this->clearLoginAttempts($request);

        // 回傳 authenticated 後的一些資訊
        return $this->authenticated($request, $user);
    }

    protected function authenticated(Request $request, $user)
    {
        return [
            'message' => __('User authenticated.'),
            'data' => $user, // 這邊的 user 是有帶入 access_token 的，以利前端使用
        ];
    }

    protected function findUser(array $credentials): ?User
    {
        return User::where($this->username(), $credentials[$this->username()])->first();
    }
}
```

## 實作「登出」功能

「登出」的在 Passport 的意義可以理解為：讓某個特定的 Access Token 失效。

```php
<?php

namespace App\Http\Controllers\Auth;

// 因為 Login 與 Logout 算是類似的功能，Laravel 將其罝都置於 LoginController 中
// 事實上，如果將其命名為 AuthController 可能會比較合乎語意
class LoginController extends Controller
{
    public function logout(Request $request)
    {
        // $request->user()->token() 是來自於 User Model 中的 token()
        // 這個值應該是 Laravel\Passport\Token 或 NULL，但使用者目前可以登入，可以預設它並不是 NULL
        $token = $request->user()->token();

        // 利用 revoke() 使這個 token 失效
        $token->revoke();

        return $this->loggedOut($request);
    }

    public function loggedOut(Request $request)
    {
        return [
            'message' => __('Access token has been revoked.'),
        ];
    }
}
```

## 撰寫測試

### 為 RegisterController 寫 Feature Test

```php
<?php

namespace Tests\Feature\Auth;

class RegisterTest extends TestCase
{
    use RefreshDatabase;

    public function test_register()
    {
        Event::fake(Registered::class);
        $user = factory(User::class)->make();

        $response = $this->postJson(route('register'), [
            'name' => $user->name,
            'email' => $user->email,
            'password' => 'password',
        ]);

        $response->assertSuccessful();
        $response->assertJson([
            'message' => 'User registered.',
            'data' => [
                'name' => $user->name,
                'email' => $user->email,
            ],
        ]);
        $this->assertDatabaseHas('users', [
            'name' => $user->name,
            'email' => $user->email,
        ]);
        Event::assertDispatched(Registered::class);
    }
}
```

## 為 Login 及 Logout 寫 Feature Test

```php
<?php

namespace Tests\Feature\Auth;

class LoginTest extends TestCase
{
    use RefreshDatabase;
     // 因為我們會使用 Passport，所以在測試前需要先建立 Passport 相關的環境
     // 這邊編寫了一個自定義的 InstallPassport trait 去執行這項工作
     // 詳細的實作可以參見 https://blog.chivincent.net/post/如何讓-laravel-test-case-自動執行-trait-function/
    use InstallPassport;

    public function test_login()
    {
        $user = factory(User::class)->create();

        $response = $this->postJson(route('login'), [
            'email' => $user->email,
            'password' => 'password',
        ]);

        $response->assertSuccessful();
        $response->assertJson([
            'message' => 'User authenticated.',
            'data' => [
                'id' => $user->id,
                'name' => $user->name,
                'email' => $user->email,
            ],
        ]);
        $this->assertNotNull($response->json('data.access_token'));
        $this->assertDatabaseHas('oauth_access_tokens', [
            'user_id' => $user->id,
        ]);
    }

    public function test_logout()
    {
        $user = factory(User::class)->create();
        $token = $user->createToken('password_token')->accessToken;

        $response = $this->postJson(route('logout'), [], [
            'Authorization' => "Bearer $token",
        ]);

        $response->assertSuccessful();
        $this->assertDatabaseHas('oauth_access_tokens', [
            'user_id' => $user->id,
            'revoked' => true,
        ]);
    }
}
```