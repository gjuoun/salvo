<div align="center">
<p><img alt="Savlo" width="132" style="max-width:40%;min-width:60px;" src="https://salvo.rs/images/logo-text.svg" /></p>
<p>
    <a href="https://github.com/salvo-rs/salvo/blob/main/README.md">English</a>&nbsp;&nbsp;
    <a href="https://github.com/salvo-rs/salvo/blob/main/README.zh-hans.md">简体中文</a>&nbsp;&nbsp;
    <a href="https://github.com/salvo-rs/salvo/blob/main/README.zh-hant.md">繁體中文</a>
</p>
<p>
<a href="https://github.com/salvo-rs/salvo/actions">
    <img alt="build status" src="https://github.com/salvo-rs/salvo/workflows/ci-linux/badge.svg?branch=main&event=push" />
</a>
<a href="https://github.com/salvo-rs/salvo/actions">
    <img alt="build status" src="https://github.com/salvo-rs/salvo/workflows/ci-macos/badge.svg?branch=main&event=push" />
</a>
<a href="https://github.com/salvo-rs/salvo/actions">
    <img alt="build status" src="https://github.com/salvo-rs/salvo/workflows/ci-windows/badge.svg?branch=main&event=push" />
</a>
<br>
<a href="https://crates.io/crates/salvo"><img alt="crates.io" src="https://img.shields.io/crates/v/salvo" /></a>
<a href="https://docs.rs/salvo"><img alt="Documentation" src="https://docs.rs/salvo/badge.svg" /></a>
<a href="https://github.com/rust-secure-code/safety-dance/"><img alt="unsafe forbidden" src="https://img.shields.io/badge/unsafe-forbidden-success.svg" /></a>
<a href="https://blog.rust-lang.org/2022/09/22/Rust-1.64.0.html"><img alt="Rust Version" src="https://img.shields.io/badge/rust-1.64%2B-blue" /></a>
<br>
<a href="https://salvo.rs">
    <img alt="Website" src="https://img.shields.io/badge/https-salvo.rs-%23f00" />
</a>
<a href="https://codecov.io/gh/salvo-rs/salvo"><img alt="codecov" src="https://codecov.io/gh/salvo-rs/salvo/branch/main/graph/badge.svg" /></a>
<a href="https://crates.io/crates/salvo"><img alt="Download" src="https://img.shields.io/crates/d/salvo.svg" /></a>
<img alt="License" src="https://img.shields.io/crates/l/salvo.svg" />
</p>
</div>

Salvo 是一個極其簡單且功能強大的 Rust Web 後端框架. 僅僅需要基礎 Rust 知識即可開發後端服務.

> **Note**: salvo's [main](https://github.com/salvo-rs/salvo) branch is
> currently preparing breaking changes. For the most recently *released* code,
> look to the [0.37.x branch](https://github.com/salvo-rs/salvo/tree/v0.37.x).
> 
> 中国用户可以添加我微信(chrislearn), 拉微信讨论群.

## 🎯 功能特色

  - 基於 [Hyper](https://crates.io/crates/hyper), [Tokio](https://crates.io/crates/tokio) 開發;
  - 統一的中間件和句柄接口;
  - 支持 Http1, Http2 和 **Http3**;
  - 路由支持無限層次嵌套;
  - 每一個路由都可以擁有一個或者多個中間件;
  - 集成 Multipart 錶單處理;
  - 支持 WebSocket;
  - 支持 Acme, 自動從 [let's encrypt](https://letsencrypt.org/) 獲取 TLS 證書.

## ⚡️ 快速開始

你可以查看[實例代碼](https://github.com/salvo-rs/salvo/tree/main/examples),  或者訪問[官網](https://salvo.rs).


創建一個全新的項目:

```bash
cargo new hello_salvo --bin
```

添加依賴項到 `Cargo.toml`

```toml
[dependencies]
salvo = "*"
tokio = { version = "1", features = ["macros"] }
```
在 `main.rs` 中創建一個簡單的函數句柄, 命名為`hello`, 這個函數隻是簡單地打印文本 ```"Hello World"```.

```rust
use salvo::prelude::*;

#[handler]
async fn hello(_req: &mut Request, _depot: &mut Depot, res: &mut Response) {
    res.render(Text::Plain("Hello World"));
}
```

### 中間件

Salvo 中的中間件其實就是 Handler, 冇有其他任何特別之處. **所以書寫中間件並不需要像其他某些框架需要掌握泛型關聯類型等知識. 隻要你會寫函數就會寫中間件, 就是這麼簡單!!!**

```rust
use salvo::http::header::{self, HeaderValue};
use salvo::prelude::*;

#[handler]
async fn add_header(res: &mut Response) {
    res.headers_mut()
        .insert(header::SERVER, HeaderValue::from_static("Salvo"));
}
```

然後將它添加到路由中:

```rust
Router::new().hoop(add_header).get(hello)
```

這就是一個簡單的中間件, 它嚮 ```Response``` 的頭部添加了 ```Header```, 查看[完整源碼](https://github.com/salvo-rs/salvo/blob/main/examples/middleware-add-header/src/main.rs).


### 可鏈式書寫的樹狀路由係統

正常情況下我們是這樣寫路由的：

```rust
Router::with_path("articles").get(list_articles).post(create_article);
Router::with_path("articles/<id>")
    .get(show_article)
    .patch(edit_article)
    .delete(delete_article);
```

往往查看文章和文章列錶是不需要用戶登錄的, 但是創建, 編輯, 刪除文章等需要用戶登錄認證權限才可以. Salvo 中支持嵌套的路由係統可以很好地滿足這種需求. 我們可以把不需要用戶登錄的路由寫到一起：

```rust
Router::with_path("articles")
    .get(list_articles)
    .push(Router::with_path("<id>").get(show_article));
```

然後把需要用戶登錄的路由寫到一起， 並且使用相應的中間件驗證用戶是否登錄：

```rust
Router::with_path("articles")
    .hoop(auth_check)
    .post(list_articles)
    .push(Router::with_path("<id>").patch(edit_article).delete(delete_article));
```

雖然這兩個路由都有這同樣的 ```path("articles")```, 然而它們依然可以被同時添加到同一個父路由, 所以最後的路由長成了這個樣子:

```rust
Router::new()
    .push(
        Router::with_path("articles")
            .get(list_articles)
            .push(Router::with_path("<id>").get(show_article)),
    )
    .push(
        Router::with_path("articles")
            .hoop(auth_check)
            .post(list_articles)
            .push(Router::with_path("<id>").patch(edit_article).delete(delete_article)),
    );
```

```<id>```匹配了路徑中的一個片段, 正常情況下文章的 ```id``` 隻是一個數字, 這是我們可以使用正則錶達式限製 ```id``` 的匹配規則, ```r"<id:/\d+/>"```. 

還可以通過 ```<*>``` 或者 ```<**>``` 匹配所有剩餘的路徑片段. 為了代碼易讀性性強些, 也可以添加適合的名字, 讓路徑語義更清晰, 比如: ```<**file_path>```.

有些用於匹配路徑的正則錶達式需要經常被使用, 可以將它事先註冊, 比如 GUID:

```rust
PathFilter::register_wisp_regex(
    "guid",
    Regex::new("[0-9a-fA-F]{8}-([0-9a-fA-F]{4}-){3}[0-9a-fA-F]{12}").unwrap(),
);
```

這樣在需要路徑匹配時就變得更簡潔:

```rust
Router::with_path("<id:guid>").get(index)
```

查看[完整源碼](https://github.com/salvo-rs/salvo/blob/main/examples/routing-guid/src/main.rs)

### 文件上傳

可以通過 ```Request``` 中的 ```file``` 異步獲取上傳的文件:

```rust
#[handler]
async fn upload(req: &mut Request, res: &mut Response) {
    let file = req.file("file").await;
    if let Some(file) = file {
        let dest = format!("temp/{}", file.name().unwrap_or_else(|| "file".into()));
        if let Err(e) = std::fs::copy(&file.path, Path::new(&dest)) {
            res.set_status_code(StatusCode::INTERNAL_SERVER_ERROR);
        } else {
            res.render("Ok");
        }
    } else {
        res.set_status_code(StatusCode::BAD_REQUEST);
    }
}
```

### 提取請求數據

可以輕鬆地從多個不同數據源獲取數據, 並且組裝為你想要的類型. 可以先定義一個自定義的類型, 比如: 

```rust
#[derive(Serialize, Deserialize, Extractible, Debug)]
/// 默認從 body 中獲取數據字段值
#[extract(default_source(from = "body"))]
struct GoodMan<'a> {
    /// 其中, id 號從請求路徑參數中獲取, 並且自動解析數據為 i64 類型.
    #[extract(source(from = "param"))]
    id: i64,
    /// 可以使用引用類型, 避免內存複製.
    username: &'a str,
    first_name: String,
    last_name: String,
}
```

然後在 ```Handler``` 中可以這樣獲取數據:

```rust
#[handler]
async fn edit(req: &mut Request) {
    let good_man: GoodMan<'_> = req.extract().await.unwrap();
}
```

甚至於可以直接把類型作為參數傳入函數, 像這樣:


```rust
#[handler]
async fn edit<'a>(good_man: GoodMan<'a>) {
    res.render(Json(good_man));
}
```

數據類型的定義有相當大的靈活性, 甚至可以根據需要解析為嵌套的結構:

```rust
#[derive(Serialize, Deserialize, Extractible, Debug)]
#[extract(default_source(from = "body", format = "json"))]
struct GoodMan<'a> {
    #[extract(source(from = "param"))]
    id: i64,
    #[extract(source(from = "query"))]
    username: &'a str,
    first_name: String,
    last_name: String,
    lovers: Vec<String>,
    /// 這個 nested 字段完全是從 Request 重新解析.
    #[extract(source(from = "request"))]
    nested: Nested<'a>,
}

#[derive(Serialize, Deserialize, Extractible, Debug)]
#[extract(default_source(from = "body", format = "json"))]
struct Nested<'a> {
    #[extract(source(from = "param"))]
    id: i64,
    #[extract(source(from = "query"))]
    username: &'a str,
    first_name: String,
    last_name: String,
    #[extract(rename = "lovers")]
    #[serde(default)]
    pets: Vec<String>,
}
```

查看[完整源碼](https://github.com/salvo-rs/salvo/blob/main/examples/extract-nested/src/main.rs)

### 更多示例

您可以從 [examples](./examples/) 文件夾下查看更多示例代碼, 您可以通過以下命令運行這些示例：

```
cargo run --bin example-basic-auth
```

您可以使用任何你想運行的示例名稱替代這裏的 ```basic-auth```.

## 🚀 性能

Benchmark 測試結果可以從這裏查看:

[https://web-frameworks-benchmark.netlify.app/result?l=rust](https://web-frameworks-benchmark.netlify.app/result?l=rust)

[https://www.techempower.com/benchmarks/#section=data-r21](https://www.techempower.com/benchmarks/#section=data-r21)
![techempower](assets/tp.jpg)

## 🎇 部署

你可以通過 [shuttle.rs](https://www.shuttle.rs/) 部署你的 Salvo 項目, 這非常簡單, 具體參見 shuttle's [官方文檔)(https://docs.shuttle.rs/guide/salvo-examples.html).

## 🩸 貢獻

非常歡迎大家為項目貢獻力量，可以通過以下方法為項目作出貢獻:

  - 在 issue 中提交功能需求和 bug report;
  - 在 issues 或者 require feedback 下留下自己的意見;
  - 通過 pull requests 提交代碼;
  - 在博客或者技術平臺發錶 Salvo 相關的技術文章。

All pull requests are code reviewed and tested by the CI. Note that unless you explicitly state otherwise, any contribution intentionally submitted for inclusion in Salvo by you shall be dual licensed under the MIT License, without any additional terms or conditions.

## ☕ 支持

`Salvo`是一個開源項目, 如果想支持本項目, 可以 ☕ [**在這裏買一杯咖啡**](https://ko-fi.com/chrislearn). 
<p style="text-align: center;">
<img src="https://salvo.rs/images/alipay.png" alt="Alipay" width="180"/>&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;<img src="https://salvo.rs/images/weixin.png" alt="Weixin" width="180"/>
</p>


## ⚠️ 開源協議

Salvo 項目採用以下開源協議:
* Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or [http://www.apache.org/licenses/LICENSE-2.0](http://www.apache.org/licenses/LICENSE-2.0))
* MIT license ([LICENSE-MIT](LICENSE-MIT) or [http://opensource.org/licenses/MIT](http://opensource.org/licenses/MIT))

