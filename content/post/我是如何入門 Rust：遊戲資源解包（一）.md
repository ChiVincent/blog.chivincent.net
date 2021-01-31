---
title: "我是如何入門 Rust：遊戲資源解包（一）"
date: 2020-01-13T12:35:07+08:00
draft: false
categories:
    - Rust
tags:
    - Rust
    - Crossgate
    - Game Analyze
---

# 前言

從大學本科畢業之後，一直在做 PHP 工程師。

儘管工作上幾乎摸不到 C/C++ 這類相對較基礎的語言，但在刷 Leetcode 時還是比較習慣使用 C/C++。

大概從 Rust 1.2x 版左右就聽聞過這個程式語言，但實際接觸過幾次卻覺得它的編譯器實在有夠囉嗦，各種「常識」覺得應該編譯得過的東西在 Rust 上就是不行。

## 概述

這次會開始使用 Rust 是因為一個 Side Project，大概從國小二年級左右開始接觸一款遊戲叫做 CrossGate（中譯「魔力寶貝」），因為是月費制遊戲導致只玩了一個暑假。

再次接觸到這遊戲已是大學畢業，卻是以比較不同的角度去看待這遊戲：如何解出正確的資源檔案。

因為本人沒有什麼逆向工程的技術，但畢竟這也是個快 20 年的老遊戲，資源格式跟解壓算法也被拆得差不多了，所以我就依照前輩們留下來的資料進行整理與拆解。

> 註：事實上網路上的資料難免存在遺漏、偏誤或解釋不清的情況，所以本次實作大多數的時間都在實驗與處理這些參考資料。

**在此還是提醒一下：本篇內容僅供學術研究用途，請勿用於任何有侵害版權疑慮之處**

# 研究

為了研究 CrossGate 的資料格式，我主要參考了幾個網站：

1. [御劍軒](https://cgsword.com/filesystem_graphicmap.htm)
2. [魔力宝贝高清单机计划（一） 图库提取](https://blog.csdn.net/qq_37543025/article/details/88377553)

整理了一下它們的說明，並且整合進了 [GraphicInfo 的 Wiki]((https://github.com/x-gate/graphic-info/wiki/Data-Structure) 與 [Graphic 的 Wiki](https://github.com/x-gate/graphic/wiki) 之中，順便針對目前大宇資訊代理的港台澳伺服器進行說明上的調整。

> 註：Graphic 的 Wiki 其實還不夠完善，大多還是從御劍軒下來的資料，但經實驗發現在解碼（Decode）的操作上可能還有一些缺失（導致拆出來的資料有時會有色彩上的問題或是破圖）。

# 實作

## GraphicInfo

GraphicInfo 的資料結構相對單純：不分版本，一律都是 40 bytes。

結構的定義如下，它定義於 [src/data_structure/graphic_info.rs](https://github.com/x-gate/graphic-info/blob/master/src/data_structure/graphic_info.rs)

```rust
pub struct GraphicInfo {
    pub id: u32,
    pub address: u32,
    pub length: u32,
    pub offset_x: i32,
    pub offset_y: i32,
    pub width: u32,
    pub height: u32,
    pub tile_east: u8,
    pub tile_south: u8,
    pub access: u8,
    pub unknown: [u8; 5],
    pub map: u32,
}
```

原先我採用 [byteorder](https://crates.io/crates/byteorder) 然後一個個解出 struct fields，其實現類似於

```rust
impl GraphicInfo {
    pub fn new(bytes: &[u8]) -> Self {
	    let cursor = new Cursor(bytes);
		
		let id = cursor.read_u32::<LittleEndian>().unwrap();
		let address = cursor.read_u32::<LittleEndian>().unwrap();
		// 以下省略…
		
		// Self { id, address, …以後省略 }
	}
}
```

但是這樣的做法實在又臭又長，原本想用 Reddit 上討論有人推薦的 [nom](https://github.com/Geal/nom)，但它的用法實在過於複雜所以看了文件之後放棄。

之後某個偶然下，發現了 [bincode](https://github.com/servo/bincode) 與 [serde](https://github.com/serde-rs/serde)

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize)]
struct GraphicInfo {
    // 省略
}

// let bytes: Vec<u8> = vec![0x00, 0x00, 0x00, ....];
// 序列化為 GraphicInfo structure
let graphic_info: GraphicInfo = bincode::serialize(&bytes).unwrap();
// 反序列化為 u8 slice
let data = bincode::deserialize(&graphic_info).unwrap();
```

## 命令列參數處理

依照 [The Rust Book 第十二章](https://doc.rust-lang.org/book/ch12-00-an-io-project.html) 的內容，嘗試建立了一個 Command Line Program。

原本是依照裡面的 example code 去解析命令列參數，實現類似以下範例：

```rust
fn main() {
	let args: Vec<string> = env::args().collect();
	
	if args.len() != 3 {
		panic!("Usage: ./graphic-info [GraphicInfo.bin] [Graphic.bin]");
	}
}
```

這是相當陽春的，其中包含幾個缺點

1. 第二個參數是 `GraphicInfo.bin`、第三個參數 `Graphic.bin`，兩者不能交換
2. 僅用 `panic!` 去寫下說明，不夠友善
3. 在建立比較複雜的規則時難以管理

於是我採用 [clap](https://crates.io/crates/clap) 來處理命令列參數，它具備以下特點

1. 設定參數是否為必須：`required(true)`
2. 設定參數的預設值：`default_value("Hello")`
3. 設定參數的長短：`.short("i").long("input")`
4. 自動產生 `--help`
5. 將參數綁定到自定義結構（底層為 [structopt](https://github.com/TeXitoi/structopt)）

比較大的缺點大概是它在設定時會產生非常多層 `()`，導致有時會覺得混亂，不過可以直接載入 `yaml` 作為參數設定檔，所以這個問題也不算很大。

## SQLite 的使用

因為 GraphicInfo 在編輯上是不方便的，所以我將其匯出至 SQLite 之中，並且能夠由 SQLite 的資料生成 GraphicInfo

在 SQLite 的操作上，我當時選擇 [sqlite](https://crates.io/crates/sqlite)，不過程式寫下去才發現 Rust Cookbook 是使用 [runsqlite](https://github.com/jgallagher/rusqlite)，後者提供更多易於使用 macros。

關於 SQLite 的連線部份我寫在 [src/storage/sqlite.rs](https://github.com/x-gate/graphic-info/blob/master/src/storage/sqlite.rs) 之中。

> TODO: 它應該要被重構為 traits，使其可以支援不止 SQLite

## 建立 GraphicInfoFile

因為 GraphicInfo 本質上是儲存於某個檔案之中，我們當然可以使用 `File::open()` 去取得該檔案，但若是需要在其上實現某些 function 是不被允許的（孤兒規則）

所以我建立了一個 `struct GraphicInfoFile(File)` 作為介面以供接取。

### Iterator 的實作

因為我們知道 GraphicInfo 是以 40 bytes 為單位，所以我們其實可以為 GraphicInfoFile 實作一個 Iterator，每次讀取 40 bytes。

```rust
impl Iterator for GraphicInfoFile {
    type Item = GraphicInfo; 
	
	fn next(&mut self) -> Option<Self::Item> {
		let mut buf = [0; 40];
		
		match self.0.read_exact(&mut buf) {
			Ok(_) => {
				let graphic_info: GraphicInfo = bincode::deserialize(&buf).unwrap();
				Some(graphic_info)
			}, 
			Err(_) => None,
		}
	}
}
```

### 匯出為 SQLite

在 [src/data_structure/graphic_info_file.rs](https://github.com/x-gate/graphic-info/blob/master/src/data_structure/graphic_info_file.rs#L36) 的 36 行，定義了 `pub fn dump_into(&mut self, database: &Sqlite)` 

這個函式可以幫助我把當前的 GraphicInfo 檔案轉到 SQLite，但我認為那個型別轉換是真的囉嗦……

### 自 SQLite 生成 GraphicInfoFile

在 [src/data_structure/graphic_info_file.rs](https://github.com/x-gate/graphic-info/blob/master/src/data_structure/graphic_info_file.rs#L70) 的 70 行，定義了 `pub fn build_from(&mut self, database: &Sqlite)`

這個函式可以把 SQLite 中的資料序列化為 `struct GraphicInfo`，然後再把它寫入特定檔案（這得益於 bincode 的強大功能）

# 後記

原本這個 Side Project 僅僅是作為學習 Rust 的實作，過程中也順道發現了不少易於使用的 crates。

不過我在實作時刻意避開了 Lifetime、Multithread、Smart Pointer 這些相對比較進階的 Rust 內容，或許在下一篇文章解說 Graphic 時可以去擁抱它們。

## 對於 Rust 的一些想法

Rust 寫起來是真的開心，雖然編譯器真的囉嗦，但至少給出來的資訊大部份是足以解決問題的。

不過 Rust 的錯誤處理算是從來沒見過的型式，類似於 Golang 中的 `if _, err := maybe_error(); err != nil { }`，但更清晰表達「錯誤」的這件事。

只不過大部份情況下在閉包之中無法使用（需要另外指定 `|| -> Result<(), std::error::Error>`），而無法享有自動剖析閉包回傳值的特性。

另外就是對於 Lifetime 的觀念也是對初學者而言難以上手的，一下是 `<'a>` 一下是 `move` 有時又有 `&`，結果通常搞成「如果沒辦法編譯過，那就加個 `&` 試試」的思維（註：這不正確
