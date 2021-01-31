---
title: "如何讓 Laravel Test Case 自動執行 Trait Function"
date: 2019-07-03T16:19:04+08:00
draft: false
categories:
    - Laravel
tags:
    - PHP
    - Laravel
    - Laravel Passport
    - Laravel Testing
---

# 前言

在測試 Laravel Passport 時，如果在 phpunit test case 的 `setUp()` 中加入 `$this->artisan('passport:install')` 是非常影響測試效率的。

這是因為 `passport:install` 會去產生 RSA Key Pair，而公私鑰對的產生是需要不少時間運算。

不使用 `$this->artisan('passport:install')` 的話又會讓 `setUp()` 稍嫌複雜，大概會寫成以下內容

```php
<?php

class PassportTestCase extends Tests\TestCase
{
    public function setUp() 
    {
        parent::setUp();

        if (!$this->keyExists()) {
            $this->makeKeyPair();
        }

        $this->makeClients();
    }
}
```

其中 `$this->keyFileExists()`, `$this->makeKeyPair()` 及 `$this->makeClients()` 都需要另外實現。

# 目標

- 在 TestCase 中 use 指定的 trait 時，執行指定的 trait function。

# 分析

對於 Laravel 底層運作沒有興趣的人，可以直接跳到 [實作](#實作)

## 是否已經有類似的功能？

事實上有的，如果在 TestCase 中 `use Illuminate\Foundation\Testing\RefreshDatabase` 的話，就會自動執行 `refreshDatabase()` 這個 function。

## 它是如何被實現的？

在 `Tests\TestCase` 中所使用的 `Illuminate\Foundation\Testing\TestCase` 中有一個 protected function `setUpTraits()`，實作如下

```php
<?php
    /**
     * Boot the testing helper traits.
     *
     * @return array
     */
    protected function setUpTraits()
    {
        $uses = array_flip(class_uses_recursive(static::class));

        if (isset($uses[RefreshDatabase::class])) {
            $this->refreshDatabase();
        }

        if (isset($uses[DatabaseMigrations::class])) {
            $this->runDatabaseMigrations();
        }

        if (isset($uses[DatabaseTransactions::class])) {
            $this->beginDatabaseTransaction();
        }

        if (isset($uses[WithoutMiddleware::class])) {
            $this->disableMiddlewareForAllTests();
        }

        if (isset($uses[WithoutEvents::class])) {
            $this->disableEventsForAllTests();
        }

        if (isset($uses[WithFaker::class])) {
            $this->setUpFaker();
        }

        return $uses;
    }
```

其中，`class_uses_recursive()` 是 Laravel 實現的 Helper Function。

它可以將指定的 class 及其 parent classes 所使用的 traits 都列舉出來。

此時再根據 `$uses` 是否存在指定的 traits 以決定是否執行某些 methods。

例如，如果存在 `RefreshDatabase` 這個 Class，就執行 `refreshDatabase()` 這個 method。

而且這個 method 還很好心，把 `$uses` 也 return 出來，讓我們不必重複再去取一次。

# 實作

## 建構 `InstallPassport` trait

今天的目的是希望可以在 TestCases 中 `use InstallPassport` 時，執行以下動作
    - 檢查 Key Pair 是否存在，如果不存在則建立 Key Pair
    - 建立 Personal Access Clients 及 Password Clients（因為每次 RefreshDatabase 都會被洗掉，所以都要重建）

建立 `app/Foundations/Testing/InstallPassport.php` 內容如下

```php
<?php

namespace App\Foundation\Testing;

trait InstallPassport
{
    public function installPassport()
    {
        if (!$this->keyExists()) {
            $this->makeKeys();
        }

        $this->makeClients();
    }

    protected function keyExists()
    {
        // 如果有修改 passport keys 的儲存位置，請記得在這邊修改
        $path = storage_path();

        if (!(file_exists("$path/oauth-private.key") && file_exists("$path/oauth-public.key"))) {
            return false;
        }

        // 可以在這邊校驗 Key Pair 的合法性，但理論上是不需要的。

        return true;
    }

    protected function makeKeys()
    {
        $this->artisan('passport:keys', ['--force' => true, '--length' => 4096]);
    }

    protected function makeClients()
    {
        $this->artisan('passport:client', ['--personal' => true, '--name' => 'Testing Personal Access Client']);
        $this->artisan('passport:client', ['--password' => true, '--name' => 'Testing Password Grant Client']); 
    }
}
```

## 覆寫 `setUpTraits()`

在 `tests/TestCase.php` 中覆寫（Override）Parent Class 的 `setUpTraits()`

```php
<?php

namespace Tests;

use App\Foundation\Testing\InstallPassport;
use Illuminate\Foundation\Testing\TestCase as BaseTestCase;

abstract class TestCase extends BaseTestCase
{
    protected function setUpTraits()
    {
        $uses = parent::setUpTraits();

        if (isset($uses[InstallPassport::class])) {
            $this->installPassport();
        }

        return $uses;
    }
}
```

# 後記

這個小技巧算是在寫 Laravel Test Case 可以應用的小訣竅，如此一來就可以在使用 trait 的同時還不用特別去初始化。

不過還是要提醒一下

- **除非你非常瞭解 trait 的用途與用法，否則不應該在 test case 中使用過多的 trait**
- **絕對不要用 trait 下去簡化 test cases，那會讓測試變得難以維護**