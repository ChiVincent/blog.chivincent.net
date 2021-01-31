---
title: "PHP 的複合類型"
date: 2018-09-19T00:00:00+00:00
draft: false
categories:
    - 程式語言實作
tags:
    - PHP
    - Data Types
---

> 我們時常熟練於使用某些東西，但事實上對它卻一無所知。

# 前言

根據 PHP 官方文件指出，PHP 的資料型態分為純類型（scalar types）、複合類型（compound types）及特殊類型（special types）

- 純類型：`boolean`, `integer`, `fload/double` 及 `string`
- 複合類型：`array`, `object`, `callable` 及 `iterable`
- 特殊類型：`resource` 及 `NULL`

其中，複合類型是個令人困惑的存在，它可能會被任意改變，也可能一成不變。

# 實驗

```php
<?php

$arr = ['a' => 1];
$obj = (object)['a' => 1];
$callable = function () { return 1; };

function changeArray(array $a) {
    $a['a'] = 2;
}

function changeObject(object $o) {
    $o->a = 2;
}

function changeCallable(callable $c) {
    $c = function () { return 2; };
}
```

- 定義三個 compound types: `$arr`, `$obj` 及 `$callable`
- 定義三個 functions，它會將傳入的值變更

考量以下程式，試問執行結束後 `$arr`, `$obj` 及 `$callable` 的值分別為何？

```php
changeArray($arr);
changeObject($obj);
changeCallable($callable);
```

![Result for changes](/2018-09-19/1_KY50hEVgQo0FwFaWcWK9fQ.png)

我們可以見到，僅有 `$obj` 的值被改變了。

# 說明

## 淺拷貝（Shallow Copy）與深拷貝（Deep Copy）

在計算機科學中，如果要「複製」（Copy）有兩種常見的方式：

- 淺拷貝：申請一個記憶體位置，並儲存被拷貝的值所在的記憶體位置。
- 深拷貝：申請一個記憶體位置，將被拷貝的值重新在這個記憶體複寫一次。

![在 Single Object 下的 Shallow Copy 及 Deep Copy](/2018-09-19/1_y81qOaNk5Uy3w0A8aWTwLw.jpeg)

<small>在 Single Object 下的 Shallow Copy 及 Deep Copy</small>

![在 Linked List 下的 Shallow Copy 及 Deep Copy](/2018-09-19/1_RNDeqUnJkGX5lGjsOqWt9w.jpeg)

<small>在 Linked List 下的 Shallow Copy 及 Deep Copy</small>

我們可以看出來，在 Shallow Copy 時若改變 Original 的資料，將會連帶影響到 Clone 出來的資料；Deep Copy 的資料則是 Original 與 Cloned 互相獨立。

至此，我們可以推斷在 PHP 中，`array` 及 `callable` 在 function call 時的值傳遞方式是 Deep Copy，而 `object` 則是 Shallow Copy

## 設計理念？

通常在程式語言的設計中，「已知且固定記憶體大小」的資料會採用 Deep Copy，「未知固定記憶體大小，或所佔用記憶體較高」的資料會採用 Shallow Copy。

因為在 PHP 中，`object` 可能不是固定大小的資料型態（它可能是一個 Linked List 串接大量的 Node 組合而成，甚至是一個 Tree 或 Graph），故在設計上採用 Shallow Coopy。

而 `array` 或 `callable` 在建立時就已知其記憶體大小（更新時也會一併更新該值），故設計上採用 Deep Copy。

> 註：事實上在 PHP 底層的實作中，`array`, `object` 及 `callable` 都是從 `zval` （它是一個自定義的 HashMap Structure）衍生出來的。

## 如何繞過

在大部份情況下，預設是足夠使用的，然而寫程式最有趣的莫過於總會出現例外情況。

我們可以利用 `&` 運算子將原本是 Deep Copy 轉為 Shallow Copy：

```php
<?php

function changeArray(array &$arr) {
    $arr['a'] = 2;
}

function changeCallable(callable &$callable) {
    $callable = function ()  { return 2; };
}
```

同時也可以利用 `clone` 將原本是 Shallow Copy 的 `object` 轉為 Deep Copy

```php
<?php

function changeObject(object $object) {
    $newObj = clone $object;
    $newObje->a = 2;
}
```

值得注意的是，Deep Copy 會使 `object` 耗費過多資源（尤其在 GC 上特別明顯），請謹慎使用。

話雖如此，大多數的 PHP 開發者並不會去在乎這些底層的開銷，而 PHP 的設計理念也不希望開發者太過於在意這些細節。

# 參考資料

1. [Understanding PHP's internal array implementation](https://nikic.github.io/2012/03/28/Understanding-PHPs-internal-array-implementation.html)
2. [PHP 7 Arrays: HashTable](http://blog.jpauli.tech/2016/04/08/hashtables.html)
3. [[Javascript]關於 JS 中的淺拷貝和深拷貝](http://larry850806.github.io/2016/09/20/shallow-vs-deep-copy/)