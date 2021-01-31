---
title: "Laravel Validation Sql Injection"
date: 2019-03-22T00:00:00+00:00
draft: false
categories:
    - Laravel
tags:
    - Web Security
    - PHP
    - Laravel
---

> 標題 hen 聳動，其實保持良好習慣的話是沒啥問題的。

# 前言

Laravel 是擁有眾多用戶的 PHP Framework，雖然只是將其它語言/框架中用到爛到的幾個 Design Pattern 移植進 PHP，卻意外掀起 PHP 的革命。

> 地圖砲：就知道以前的有不少 PHPer 對於架構有多不在乎……

在 [Laravel 5.8.6 的 Release Note](https://github.com/laravel/framework/releases/tag/v5.8.6) 中，多了一筆 [commit#34759cc](https://github.com/laravel/framework/commit/34759cc0e0e63c952d7f8b7580f48144a063c684)，其內容是將 Unique Validation 的某些部份加入 `addslashes` 及 `stripslashes`。

於此同時，Laravel 的作者 Tylor 也發表了一篇 [Laravel Blog](https://blog.laravel.com/unique-rule-sql-injection-warning)，指出有安全專家認為在 Unique Validation 中有 SQL Injection 的疑慮。

這在 [GitHub Issue](https://github.com/laravel/framework/pull/27940/commits/5015a79f4e27ed80623c7d7919ed49250f67932e) 及 [Reddit](https://www.reddit.com/r/laravel/comments/b2ygav/unique_rule_sql_injection_warning/) 中引入論戰：人們普遍認為 `addslashes` 不是一個解決 SQL Injection 的方法。

# 前導概念與注意事項

## 注意事項

1. `addslashes` **絕對不是** 解決 SQL Injection 的方法
2. 雖然講過很多次，但要注意的是 **絕對不要** 信任來自客戶端的資料

## 前導概念

Laravel 的 Validation 是自創的方法，目的是優雅地驗證資料是否合法。

```php
<?php

$v = Validation::make(
    ['email' => 'song374561@chivincent.net'], 
    ['email' => 'string|email']
);

if ($v->fails()) { 
    abort(422); 
}
```

就上述範例而言，我們可以驗證 `song374561@chivincent.net` 是否為合法的 `string` 及 `email`。

同樣地，我們也可以依賴 `Validator` 去驗證特定資料庫中是否存在/不存在。

```php
<?php

$v = Validator::make(
    ['email' => 'song374561@chivincent.net'],
    ['email' => 'string|email|unique:users,email'],
);

$v = Validator::make(
    ['email' => 'song374561@chivincent.net'],
    ['email' => 
        ['string', 'email', Rule::unique('users', 'email')]
    ]
)
```

上下兩個敘述式是等價的，差別僅在於使用 `Rule::unique('users', 'email')` 的語法糖產生 `unique:users,email` 這個規則敘述式。

`Rule::unique()` 也有眾多方便好用的 methods，可以針對 Query 做操作。例如用 `ignore()` 去忽略某些 id，或是用 `where()`, `whereIn()` 之類的 Query Builder 去建立具備良好可讀性的規則。

# 問題發生原因

```php
<?php

$v = Validator::make($request->only('email'), [
    'email' => Rule::unique('users')->ignore($request->input('id')),
]);
```

以上程式常見於我可能是用戶管理員（User Manager），當我正在手動修改用戶資料時，我需要事件進行以下確認：

1. 修改後的 Email 不與其它用戶重複，使用 `Rule::unique`
2. 但是修改後的 Email 可以與當前用戶重複（表示不修改），使用 `ignore`

我傳遞了這位 User 的 Email 及 ID，以便我更改它，一切看起來合情合理。

這個 `Rule::unique('users')->ignore($request->input('id'))` 將被轉換成 `unique:users,NULL,"1",id`，其代表的意義為「此值應該在 `users` 表格中唯一，但是當 `id` 為 `1` 時忽略之」。

為了保持 Rule 的彈性，在 unique 規則中還可以另外指定要忽略的欄位名與其值，如果使用 `unique:users,NULL,"test","name"` 的話，表示要忽略的是 `name` 為 `test` 的欄位。

---

或許你已經發現了，`$request->input('id')` 是來自於使用者輸入，也就是不可信任的。

如果我今天 `$request->input('id')` 的內容為 `1","name","` 就可以將 Rule 改成 `unique:users,NULL,"test","name","",id`，執行時期其 SQL Statement 會變成：

```sql
SELECT count(*) as aggregate FROM `users` WHERE `email` = ? AND `name` <> ? AND `` = id
```

這起因於 `Rule:unique` 的設計方法，`Rule::unique` 原本就是設計為輸入 `unique:xxxx` 而出現的，這導致如果我們隨意去操作 `Rule::unique` 裡面的參數或值，會導致輸出的 `unique` 條件式與預期不符。

再加上，雖然 Value 中有 使用 Prepared Statement 去防止 SQL Injection，但是 name 被視為欄位名，所以不會套用 Prepared Statement。

# 解決思路

## 官方解決思路

為了解決這個問題，Taylor 的解決思路如下：

1. 確定 `Rule::unique` 所產生出來的內容不應該被隨意跳脫產生非預期的 `unique` 條件式，所以使用 `addslashes`
2. 使用 DB 執行規則前，避免 `addslashes` 造成問題，所以用 `stripslashes` 將加入的 slashes 移除

所以爭議最大的 `addslashes` 與 `stripslashes` 完全是用在與 Database 無關的地方，它僅僅是在剖析 `unique` 條件式中的 CSV 時避免非預期的狀況出現。

然而需要注意的是，這僅僅能夠防禦來自 `Rule::unique` 所組出來的 `unique` 條件式。如果開發者自行組裝 `unique` 條件式的話就會無法使用，例如 `unique:users,email,"{$request->input('id')}","id"` 這樣的話是無法防禦的。

同時，Laravel 官方文件中也加入提醒：在使用 `ignore()` 時，應該確定其參數來自 Eloquent Model，而非客戶端請求。

## 我的想法

我認為 Laravel 應該除除 Validation 條件式的 Feature，使用 ValidationRuleClass（之類的東西）進行外部輸入驗證，因為 Validation 條件式由 string 組成，其不確定性很高。

目前官方應該是沒有計畫廢棄 Validation 條件式，所以在將內容丟入 `Rule::unique` 之前，應該要先確定其資源存在：

```php
<?php

$u = User::findOrFail($request->input('id'));

$v = Validator::make($request->only('email'), [
    'email' => Rule::unique('users')->ingore($u->id),
]);
```

雖然這會多一次 Database Query，但至少能夠確定傳入的值是合法的，而這也是官方文件中所警告的。