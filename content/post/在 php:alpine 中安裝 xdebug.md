---
title: "在 php:alpine 中安裝 xdebug"
date: 2019-08-05T11:38:14+08:00
draft: false
categories:
    - PHP
tags:
    - PHP
    - Docker
    - Xdebug
---

> 朕不給的，你就自己裝

# 前言

Docker 官方提供許多 Official Images，尤其是針對比較熱門的幾個程式語言或執行環境，PHP 也是其中之一。

一般來說，可以利用 `docker-php-ext-install` 安裝大部份的 PHP extensions（這是一個官方提供的 shell script，它會自動幫忙處理安裝、啟動等流程）

然而不幸的是，`docker-php-ext-install` 並不提供 xdebug 的安裝，這部份就需要手動進行。

# 實作流程

## 使用 php:alpine image

```bash
$ docker pull php:alpine
```

通常會建議盡量使用官方提供的 Image，不僅能夠在發現安全漏洞時盡速拿到更新，也能夠避免惡意人士包裹奇怪的東西進 image 中。

## 撰寫 Dockerfile

```Dockerfile
FROM php:alpine

RUN apk add $PHPIZE_DEPS && pecl install xdebug && \
    echo 'zend_extension=xdebug.so' > /usr/local/etc/php/conf.d/docker-php-xdebug.ini
```

- `apk add $PHPIZE_DEPS` 安裝 PHPIZE 時所需要的軟體（autoconf, make, g++ 等）
- `pecl install xdebug` 安裝 xdebug
    - 如果有需要指定 xdebug 版本的話，可以寫為 `pecl install xdebug-2.8.0beta1` 或是 `pecl install xdebug-2.5.0`
- `echo 'zend_extension=xdebug.so' > /usr/local/etc/php/conf.d/docker-php-xdebug.ini` 在預設資料夾中加入啟用 xdebug 的設定

## 構建 docker image

```bash
$ docker build -t php:xdebug .
```

# 參考資料

1. [How do I install XDebug on docker's official php-fpm-alpine image?](https://stackoverflow.com/questions/46825502/how-do-i-install-xdebug-on-dockers-official-php-fpm-alpine-image)