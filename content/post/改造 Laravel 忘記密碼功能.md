---
title: "改造 Laravel 忘記密碼功能"
date: 2019-07-03T15:17:11+08:00
draft: false
categories:
    - Laravel
tags:
    - PHP
    - Laravel
    - Laravel Auth
---

> Laravel 使用上很容易，但改造它卻常有難度。

# 前言

Laravel 的定位是「全端框架」，它包山包海，具備前端與後端的各式功能。

然而，隨著前後端分離的思想興起、SPA 開始成為顯學，Laravel 的幾個預設功能常常會需要更動：事實上，Laravel 比較適合「使用」而非「更動」。

舉例而言，在前後端分離的情況下，主站的 URL 可能是 `http://my-home-site.test/`，但 API 的 URL 是 `http://api.my-home-site.test`，而且兩個分屬不同的專案、有不同的 code repository。

這個時候如果依照預設的 Auth 功能建構「忘記密碼」流程的話，就會發生使用了 `http://api.my-home-site.test/password/resets` 但期望卻是 `http://my-home-site.test/password/resets` 的狀況。

# 目標

- 替換 Laravel Auth Password Forgot 功能中的 URL 生成方式
    - 理由：Laravel 預設使用 `route('password.reset')` 生成 URL，但在純 API Server 中是不需要這個路由與這個 View 的（應該由前端實現）
 
# 分析

對於原理沒有興趣的人可以直接跳到 [實作](#實作)

## 如何寄發「重設密碼」的 Email？

在 `App\Auth\Http\Controllers\Auth\ForgotPasswordController` 中，使用了 `Illuminate\Foundate\Auth\SendsPasswordResetEmails` trait

其中， `sendResetLinkEmail()` 實作如下

```php
<?php

    /**
     * Send a reset link to the given user.
     *
     * @param  \Illuminate\Http\Request  $request
     * @return \Illuminate\Http\RedirectResponse|\Illuminate\Http\JsonResponse
     */
    public function sendResetLinkEmail(Request $request)
    {
        $this->validateEmail($request);

        // We will send the password reset link to this user. Once we have attempted
        // to send the link, we will examine the response then see the message we
        // need to show to the user. Finally, we'll send out a proper response.
        $response = $this->broker()->sendResetLink(
            $this->credentials($request)
        );

        return $response == Password::RESET_LINK_SENT
                    ? $this->sendResetLinkResponse($request, $response)
                    : $this->sendResetLinkFailedResponse($request, $response);
    }
```

我們不難從命名上得知，真正參與「寄發電子郵件」行為的是在 `$this->broker()->sendResetLink()` 這個位置。

`$this->broker()` 會嘗試取得目前密碼重設的 broker（譯為「中間商」較妥當），這個 broker 是一個實現（implement） `Illuminate\Contracts\Auth\PasswordBroker` 的 Class，其中存在 `sendResetLink()` 這個 method。

這個 Class 是 `Illuminate\Auth\Passwords\PasswordBroker`，其中對於 `sendResetLink()` 的實現如下

```php
<?php
    /**
     * Send a password reset link to a user.
     *
     * @param  array  $credentials
     * @return string
     */
    public function sendResetLink(array $credentials)
    {
        // First we will check to see if we found a user at the given credentials and
        // if we did not we will redirect back to this current URI with a piece of
        // "flash" data in the session to indicate to the developers the errors.
        $user = $this->getUser($credentials);

        if (is_null($user)) {
            return static::INVALID_USER;
        }

        // Once we have the reset token, we are ready to send the message out to this
        // user with a link to reset their password. We will then redirect back to
        // the current URI having nothing set in the session to indicate errors.
        $user->sendPasswordResetNotification(
            $this->tokens->create($user)
        );

        return static::RESET_LINK_SENT;
    }
```

從 `$user->sendPasswordResetNotification()` 中可以得知，寄發密碼重設通知的功能是寫在我們常用的 User Model 裡的。

## 研究 User Model 中的 `sendPasswordResetNotification()`

在 User Model 中的 `sendPasswordResetNotification()` 是實現於 `Illuminate\Auth\Passwords\CanResetPassword` trait

其中對於 `sendPasswordResetNotification()` 的實現如下

```php
<?php
    /**
     * Send the password reset notification.
     *
     * @param  string  $token
     * @return void
     */
    public function sendPasswordResetNotification($token)
    {
        $this->notify(new ResetPasswordNotification($token));
    }
```

而這個 `ResetPasswordNotification` Class 指的是 `Illuminate\Auth\Notifications\ResetPassword` Class

其中對於 `toMail()` 的實現如下

```php
<?php
    /**
     * Build the mail representation of the notification.
     *
     * @param  mixed  $notifiable
     * @return \Illuminate\Notifications\Messages\MailMessage
     */
    public function toMail($notifiable)
    {
        if (static::$toMailCallback) {
            return call_user_func(static::$toMailCallback, $notifiable, $this->token);
        }

        return (new MailMessage)
            ->subject(Lang::getFromJson('Reset Password Notification'))
            ->line(Lang::getFromJson('You are receiving this email because we received a password reset request for your account.'))
            ->action(Lang::getFromJson('Reset Password'), url(config('app.url').route('password.reset', ['token' => $this->token, 'email' => $notifiable->getEmailForPasswordReset()], false)))
            ->line(Lang::getFromJson('This password reset link will expire in :count minutes.', ['count' => config('auth.passwords.users.expire')]))
            ->line(Lang::getFromJson('If you did not request a password reset, no further action is required.'));
    }
```

看到這邊，我們已經瞭解到「重設密碼」的 Mail 以及其 URL 的生成方式。

關鍵在於這一行 `->action(Lang::getFromJson('Reset Password'), url(config('app.url').route('password.reset', ['token' => $this->token, 'email' => $notifiable->getEmailForPasswordReset()], false)))`

# 實作

## 定義 URL 及其格式

在 `config/auth.php` 中的 `password` 裡新增一個 key 名為 `reset_url`

```php
<?php

    'passwords' => [
        'users' => [
            'provider' => 'users',
            'table' => 'password_resets',
            'expire' => 60,
        ],
        // 新增以下內容
        'reset_url' => env('PASSSWORD_RESET_URL', 'https://your-application.test/password/resets/')
    ]
```

把上面的 `your-application.test` 改成你自己的網址即可，如果不知道或不確定的話也可以用 `config('app.url')`

## 改寫 User Model 中的 `sendPasswordResetNotification()`

在 `Illuminate\Auth\Notifications\ResetPassword` 其實預留了一個 public static method `toMailUsing()` 讓我們不需要重寫一個 Reset Password Notification。

> 註：其實重寫一個 `App\Auth\Notification\ResetPassword` 也是可以，只是我覺得沒啥必要。

```php
<?php

public function sendPasswordResetNotification($token)
{
    $notification = new Illuminate\Auth\Notifications\ResetPassword($token);
    $notification::toMailUsing(function (User $notifiable, string $token) {
        // 建立「重設密碼」的 URL
        $passwordResetUrl = url(
            sprintf(config('auth.passwords.reset_url') . '%s?email=%s', $token, $notifiable->getEmailForPasswordReset())
        );

        // 重建 MailMessage
        return (new MailMessage())
                ->subject(Lang::getFromJson('Reset Password Notification'))
                ->line(Lang::getFromJson('You are receiving this email because we received a password reset request for your account.'))
                ->action(Lang::getFromJson('Reset Password'), $passwordResetUrl)
                ->line(Lang::getFromJson('This password reset link will expire in :count minutes.', ['count' => config('auth.passwords.users.expire')]))
                ->line(Lang::getFromJson('If you did not request a password reset, no further action is required.'));

    });

    // 依照原本的方式執行 notify
    $this->notify($notification);
}
```

# 後記

在以往（Laravel 5.6 以前），如果要自訂 Password Reset Notification 的話，需要自己建構一個獨立的 Notification 來取代原本的。

自從有了 `toMailUsing()` 之後就可以直接使用原生的 Notification Class，算是方便許多。