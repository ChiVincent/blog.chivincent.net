---
title: "Docker 與 Laravel 的實踐（一）： Local"
date: 2019-06-27T13:58:39+08:00
draft: false
categories:
    - Deployment Envrionment
tags:
    - PHP
    - Laravel
    - Docker
    - Docker Compose
---

> 「Container 的技術有利於分工。」，ㄏㄏ。

# 前言

## 系列文

- Docker 與 Laravel 的實踐（一）：Local（本文）
- [Docker 與 Laravel 的實踐（二）：Development](/post/docker-與-laravel-的實踐二development/)
- [Docker 與 Laravel 的實踐（三）：Testing](/post/docker-與-laravel-的實踐三testing/)

## 文章適用對象

- 使用 Laravel 開發的後端工程師
- 使用 Laravel 開發前端（如內建的 Vue 跟 Blade 之類的功能）的工程師

# 實作

## 環境設計

- 以 Docker 啟動所需服務
    - 以 Port Binding 或 Volume 的方式共享服務
    - 可以利用既有工具存取服務（例如用 Navicat 存取資料庫）
- PHP Runtime 依賴開發者本身的環境
    - 開發者可以自行維護環境

## 需求

- Docker
- Docker Compose
- PHP Runtime
    - PHP 7.3
    - Composer
    - Laravel Requirements（bcmath, pdo, pdo_*, tokenizer, curl, openssl, exif, gd, json, pcntl, xml）

## 步驟

統一將環境放在 `.docker/local` 資料夾下（如果沒有這個資料夾請自行建立）

### Step 1. 建立基礎建設資料夾

建立 `.docker/local/docker-compose.yml`

```yaml
version: '3'

services:
    database:
        image: mysql
    redis:
        image: redis
```

目前設計兩個服務：`mysql` 及 `redis`

如果有需要其它服務（如 elasticsearch）就在此處新增即可

### Step 2. 設定 Port Binding

預設 MySQL 會綁定到 Port 3306；Redis 綁定到 Port 6379，在 `docker-compose.yml` 加入這些設定。

```yaml
version: '3'

services:
    database:
        image: mysql
        ports:
            - 3306:3306
    redis:
        image: redis
        ports:
            - 6379:6379
```

順帶一提，如果希望使用 unix socket 而非 port 的方式，可以用 volume 的方式搬出來。

然而受限於 Docker 的 IO Performance（尤其是在非 Linux 上），這種方式比 port binding 相比其實沒差多少（甚至效率更低），所以通常不建議這麼做。

### Step 3. 設定環境變數

MySQL 具有多個可設定的環境變數，例如 root 密碼、預設資料庫、預設使用者及密碼等。

```yaml
version: '3'

services:
    database:
        image: mysql
        ports:
            - 3306:3306
        environments:
            - MYSQL_ROOT_PASSWORD: root
            - MYSQL_DATABASE: laravel
            - MYSQL_USERNAME: homestead
            - MYSQL_PASSWORD: secret
    redis:
        image: redis
        ports:
            - 6379:6379
```

### Step 4. 設定 mysql.cnf

因為 PHP 至今還是不支援 MySQL 8 的 caching_sha2_password 認證方式，還有預設 MySQL 8 是採用 utf8mb3 的編碼方式。

此時我們需要另外加入 MySQL 的設定檔。

建立 `.docker/local/database/my.cnf`：

```ini
[mysqld]
default-authentication-plugin=mysql_native_password
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
```

然後以 volume 的方式引入

```yaml
version: '3'

services:
    database:
        image: mysql
        ports:
            - 3306:3306
        environments:
            - MYSQL_ROOT_PASSWORD: root
            - MYSQL_DATABASE: homestead
            - MYSQL_USERNAME: homestead
            - MYSQL_PASSWORD: secret
        volumes:
            - ./database/my.cnf:/etc/mysql/conf.d/custom.cnf
    redis:
        image: redis
        ports:
            - 6379:6379
```

※ 不建議去覆蓋既有的 `/etc/mysql/my.cnf`，裡面可能有些重要設定
※ 其實可以用 command 的方式加入這些參數，但這會讓執行指令變得很長且不易管理

### Step 5. 啟動服務

現在，可以在專案根目錄下執行指令啟動相應服務

```bash
docker-compose --file .docker/local/docker-compose.yml up
```

接著在 `.env` 中設定相應的環境變數（如資料庫使用者及密碼等）

```bash
php artisan serve
```

如此一來，就會啟動一個 PHP Built-in Web Server 以供開發使用