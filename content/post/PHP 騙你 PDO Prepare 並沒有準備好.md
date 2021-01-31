---
title: "PHP 騙你 PDO Prepare 並沒有準備好"
date: 2018-05-26T00:00:00+00:00
draft: false
categories:
    - 語言陷阱
tags:
    - PHP
    - PDO
---

> 被 PHP 騙也不是一天兩天的事了，習慣成自然

大部份的 Modern PHPer 都會告訴你：用 PDO 取代 `mysqli` 相關函式吧，它不僅支援 [Prepared Statement](https://en.wikipedia.org/wiki/Prepared_statement)，而且還有多種 Driver 可以隨時切換不同的 Database。

例如 Laravel 所開發的 [Eloquent ORM](https://github.com/illuminate/database) 就是使用 PDO 作為其底層的實作。

先來看看 PDO 存取資料的標準範例：

```php
<?php

try {
    $dbh = new PDO(
        'mysql:host=127.0.0.1;dbname=test;charset=utf8mb4',
        'root',
        'root'
    );

    $sth = $dbh->prepare('SELECT * FROM `users` WHERE `role` = ?');
    $sth->bindValue(1, $_GET['role']);

    $result = $sth->execute();

    var_dump($result);
} catch (PDOException $exception) {
    die("Something wrong: {$exception->getMessage()}");
}
```

常識告訴我們，如果打算把使用者輸入（包括但不限於 `$_GET`, `$_POST`, `$_COOKIE`, `$_SERVER`，甚至是 `$_SESSION`）放進 SQL Query 中，用 `prepare` 這個 PDO method 會比什麼 `addslashes` 之類的方法來得安全。

> 註：有一些[教學文](https://web.archive.org/web/20171107192133/https://blog.csdn.net/hornedreaper1988/article/details/43520257)也提到，請不要再用 `mysql(i)_real_escape_string` 來防止 SQL Injection，但我個人對此有些懷疑。

而有許多的初學者教學文章或書籍中，也告訴你如果使用 `prepare` 的時，若相同的 SQL Statement 將被執行多次，其效能會提升：因為 SQL Statement 先轉化為樣板（template），之後再對其賦值（assignment）。

> 題外話：另外，如果你手邊的書或文章沒有提到關於 PDO 的內容，請不要猶豫直接燒了它。這種情況通常出現在台灣某些不肖業者或作者，每次號稱「改版」也只是換個封面然後內容原封不動地來汙染數位出版業品質跟讀者智商的垃圾。

然而，這一切其實都是假象。在預設設定下，如果使用的是 MySQL Driver（或 Postgres Driver），PHP 的 `prepare` 其實只是在綁定值之前對該值做 `mysql_real_escape_string`，而不是依照預期的會對 Database 先做 Request Prepare Statement 後再做 Prepare Execute Statement。

這是因為 PHP 為了避免部份 Database 或 Driver 不支援 Prepared Statement，於是在預設情況下採用「模擬」的方式，進而假裝自己是使用 Prepared Statement。

另一方面，據 [ProxySQL](https://github.com/sysown/proxysql) 的 contributor [renecannao](https://github.com/renecannao) 在 [ProxySQL #1118](https://github.com/sysown/proxysql/issues/1118#issuecomment-319585127) 所述，SQL Level 的 Prepared Statement 是相當影響效能的。

順帶一提，如果使用的是 `mysqlnd` 或 `mysql` Driver （也就是不使用 PDO 的情況下），它們底層是屬於 Binary Level 的 Prepared Statement，且它們不會使用模擬的方式進行。

# 實驗

以下我做了一個實驗來驗證。

- PHP 7.2.5
- MySQL 8.0.11 (Running in Docker)

```php
<?php

try {
    $dbh = new PDO(
        'mysql:host=127.0.0.1;dbname=test;charset=utf8mb4',
        'root',
        'root'
    );
} catch (PDOException $exception) {
    die("Something wrong: {$exception->getMessage()}");
}
```

用 wireshark 分析封包，會看到以下結果

![在 PDO::ATTR_EMULATE_PREPARES = true 時（預設）](/2018-05-26/1_88qQ75DSIHslkFd1bOFXhA.png)

<small>在 PDO::ATTR_EMULATE_PREPARES = true 時（預設）</small>

如果在第 9 行加入 `$dbh->setAttribute(PDO::ATTR_EMULATE_PREPARES, false);`

![Request Prepare Statement](/2018-05-26/1_uFCR7TsihoFL30H9R_ibFQ.png)

<small>Request Prepare Statement</small>

![Request Execute Statement](/2018-05-26/1_Jl1fnWIPZuW9t86afveahg.png)

<small>Request Execute Statement</small>

# 結論

開發者應該按照需要決定如何使用 PDO Attributes，如果會在一次的 Request 中重複使用多次相同的 SQL Query，可以考慮把模擬功能關閉。

而 Laravel 中預設是是 `PDO::ATTR_EMULATE_PREPARES = true`，這一點在使用 ProxySQL 或類似應用時要相當小心。