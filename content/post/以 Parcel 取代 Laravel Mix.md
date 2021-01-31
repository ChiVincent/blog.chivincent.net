---
title: "以 Parcel 取代 Laravel Mix"
date: 2020-05-21T10:14:56+08:00
draft: false
categories:
    - PHP
tags:
    - PHP
    - Nodejs
    - Laravel
    - Front End
---

> 求不要更新了，老子學不動了。－－　[denoland/deno#25](https://github.com/denoland/deno/issues/25)

# 前言

眾所周知，前端的世界複雜到不像是人學的。

- 為了標準化 DOM 操作，出現了 jQuery
- 為了分割資料與介面，出現了三大框架（Angular, React 及 Vue）
- 為了套件管理，出現了 bower, npm
- 為了適應不同瀏覽器的標準，出現了 Babel
- 為了整合工作流程，出現了 Gulp

最後，為了整合上面的所有東西，Webpack 誕生了。

原先 Webpack 的定位是「打包器」，在 HTTP/2 尚未問世的時候，（標準上）資源並不會平行下載，所以打包成單一個檔案在大多時候比分割成多個檔案來得有效率。

隨著 JS 社群的成長茁壯，Webpack 的功能越來越五花八門：不僅可以打包，還可以做到壓縮、程式碼分割，甚至還有自動化流程。然而，Webpack 最令人詬病的就是它那個過於複雜的設定，甚至每個不同的 plugin 都還有自己的設定。

為了解決這樣的情況，很多人都嘗試將 Webpack 簡易化，其中一位挑戰者便是 Laravel Mix：它封裝了 webpack 本身並提供一些比較簡單的 API，且在預設上更貼合 Laravel Framework 的使用情境。

在大部份情況，Laravel Mix 已經算是很穩定而且方便，再加上能夠無痛適用 Webpack 的 plugin 等，幾乎可以說是 Laravel 開發者的首選。不過 Laravel Mix 如果不使用預設值，就需要自己去尋找並設定 Webpack Plugin，這對於不熟悉 Webpack 的開發者來說是很麻煩的。

所以，我選用了 [Parcel](https://parceljs.org/) 作為取代 Laravel Mix 的一個替代方案。

## Parcel 簡介

Parcel 主打的是「免設定」的網頁打包工具。

在過去，我們要打包 SASS、Fonts、Typescript、Images 可能要安裝一大堆的 Webpack Plugin。更不用說如果要支援 Vue, React 或 Angular 還需要另外設定一大堆。

在 Parcel，它都幫忙處理好了。

# 整合到 Laravel 中

在 Laravel 的使用方法與 Parcel 官網上介紹的不太一樣，畢竟它有專屬的 `resources/`（來源） 及 `public/`（目的） 資料夾。

以下是我最近比較偏好的流程，當然你可以根據你的需求做變化。

## 移除預設檔案

- 移除 `packages.json` 及 `packages-lock.json`
- 移除 `resources/js` 及 `resources/sass`
- 重新 `npm init`
- 建立 `resources/assets`（這是我們放置所有前端設定的地方）

## 安裝 parcel

- `npm install --save-dev parcel-bundler`

如果要用 React 的話，也要安裝 `react` 及 `react-dom`

## 建立進入點

建立 `resources/assets/index.js` 這將是 parcel 的進入點。

以下是我從 Create React App 中改寫來的範例

```js
import React from 'react';
import ReactDOM from 'react-dom';
import App from './App';

ReactDOM.render(
    <React.StrictMode>
        <App />
    </React.StrictMode>, 
    document.getElementById('root')
);
```

並建立 `resources/assets/App.js` 作為 React 的 Root Component。

```js
import React from 'react';

export default function App() {
    render(
        <h1>Hello React!</h1>
    );
}
```

## 撰寫打包指令

在 `packages.json` 加入 `scripts` 區段。（如果是用 `npm init` 的話，應該預設已經有這個區段了。）

```json
{
    "scripts": {
        "dev": "parcel resources/assets/index.js --out-dir public/assets",
        "prod": "parcel build resources/assets/index.js --out-dir public/assets",
        // test 是由 npm init 自動產生的
        "test": "echo \"Error: no test specified\" && exit 1"
    }
}
```

如此一來，便可以利用 `npm run dev` 或 `npm run prod` 建立打包的資源。

## 建立 Laravel 進入點

建立 `resources/views/entrypoint.blade.php` 

```php
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-" />
    <meta name="viewport" content="width=device-width, initial-scale=1" />
    <link rel="stylesheet" href="{{ asset('assets/index.css') }}">

    <title>React App</title>
  </head>
  <body>
    <noscript>You need to enable JavaScript to run this app.</noscript>
    <div id="root"></div>

  <script src="{{ asset('assets/index.js') }}"></script>
  </body>
</html>
```

在 `routes/web.php` 加入以下路由

```php
use Illuminate\Support\Facades\Route;

Route::view('/', 'entrypoint');
```

# 問題

- Parcel 會預設讀取 `/index.css.map` 及 `/index.js.map`，但是我們的 source map 會放在 `/assets/` 底下
    - 這個可以用 `--public-url` 參數設定，但在載入一些靜態檔案（例如圖片、字型）的時候可能會有問題
- 如果使用 React Router 的話，要記得為所有適配的路由都加入 Laravel 的路由中

```php
use Illuminate\Support\Facades\Route;

$routes = ['/', 'login', 'register'];

foreach ($routes as $route) {
    Route::view($route, 'entrypoint');
}
```