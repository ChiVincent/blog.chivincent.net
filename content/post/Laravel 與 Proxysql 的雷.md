---
title: "Laravel 與 Proxysql 的雷"
date: 2018-07-09T00:00:00+00:00
draft: false
categories:
    - 語言陷阱
tags:
    - PHP
    - PDO
    - ProxySQL
    - Laravel
---

> 雖然 Laravel 的 PDO Prepared，但 ProxySQL 卻沒有。

# 前言

在先前的文章[PHP 騙你 PDO prepare 並沒有準備好](/post/php-%E9%A8%99%E4%BD%A0-pdo-prepare-%E4%B8%A6%E6%B2%92%E6%9C%89%E6%BA%96%E5%82%99%E5%A5%BD/) 中，我們提到了 PHP 預設利用 `PDO::ATTR_EMULATE_PREPARES = true` 的設定，避免有些資料庫不支援 Prepared Statement

在 Laravel Framework 中，這個設定預設已被設為 `false`，也就是說 Laravel 能夠完整享受到 Prepared Statement 的益處（與壞處）。

然而，這在 ProxySQL 的開發者眼中頗[不以為然](https://github.com/sysown/proxysql/issues/1118)，他很訝異 Laravel 居然會使用如此低效率的方式進行 Database Access。

# Prepared Statement 概述

## Prepared Statement 的兩種形式

據 [MySQL 官方文件](https://dev.mysql.com/doc/refman/8.0/en/sql-syntax-prepared-statements.html) 指出，Prepared Statement 支援兩種方式：

- Binary Protocaol: 實際意義上的 Prepared Statement
- SQL Level: 以 SQL 語法實現的 Prepared Statement

C 語言的 libmysqlclient、Java 的 mysql-connector-java 或是 PHP 的 mysqli 及 mysqlnd extension 都是支援 binary protocol 的 prepared statement。

然而在 PDO 的 mysql 實現中，它卻是利用 SQL Level 實現。

順帶一提，PHP 的 Swoole extension 在 MySQL Coroutine 中，因為使用了 mysqlnd 實現 MySQL Connection，所以不僅支援 MySQL 8.0 之後的 `caching_sha2_password` 認證方式，也支援 Binary Protocol 的 Prepared Statement

## 不支援 SQL Level 的 Prepared Statement 原因

據 [ProxySQL #1118](https://github.com/sysown/proxysql/issues/1118) 所述，SQL Level 的 Prepared Statement 是相當影響效能的。

另外，據我個人的猜測，SQL Level 的 Prepared Statement 可能會造成叢集下的 SQL Statement 的錯亂：舉例來說，prepare 語句丟到了 MySQL Server A，但後續的參數綁定卻丟進了 MySQL Server B，因而造成不可預期的錯誤。

## 如何解決 ProxySQL 不支援的問題

1. 在 Laravel 的 `config/database.php` 中，加入 `'options' => [PDO::ATTR_EMULATE_PREPARES => true]`
2. 將 Laravel 的 MySQL 連接方式改為 mysqli driver，利用 Binary protocal 的 prepares
3. （尚未成熟）利用 Swoole 的 MySQL Cororoutine 取代 PDO