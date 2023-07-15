---
title: "Cloudflare WorkersでRustを使って画像と文字列を合成する"
emoji: "🐥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - Rust
  - Cloudflare
published: true
publication_name: "praha"
---

## はじめに

最近仕事で、Cloudflare Workersを使って画像と文字列を合成する処理を書くことがあり、結構おもしろかったので、記事にしてみます。

サンプルリポジトリを[こちら](https://github.com/wooootack/rust-cloudflare-sample)に用意してあるので、コード全体を見たい方はこちらを参照してください。

## 事前準備

wranglerを使うので、[こちら](https://developers.cloudflare.com/workers/wrangler/install-and-update/)の手順に従ってセットアップをお願いします

## やっていく

### 1. プロジェクトの作成

以下のコマンドを実行して、プロジェクトのひな型を作成しましょう。

```bash
npx wrangler generate {任意の名前} https://github.com/cloudflare/workers-sdk/templates/experimental/worker-rust
```

### 2. ローカルで動かす

作成されたディレクトリに移動して、以下のコマンドを実行すれば、ローカルで動かすことができます。

```bash
cd {任意の名前}
npx wrangler dev
```

コンソールに表示されたURL（`http://127.0.0.1:8787/`）にアクセスしてみると、Hello, World!と表示されていることが確認できると思います。

### 3. ルーティングを設定する

今はどんなパスにアクセスしても、Hello, World!と表示されてしまうので、ルーティングを設定してみましょう。

`src/lib.rs`を以下のように書き換えます。

```rust
use worker::*;

#[event(fetch)]
async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let router = Router::new();

    router
        .get("/users", |_, _| Response::ok("get all users!"))
        .post("/users", |_, _| Response::ok("created."))
        .patch("/users", |_, _| Response::ok("patched."))
        .delete("/users/:id", |_, _| Response::ok("deleted."))
        .get_async("/users/async", |_, _| async {
            Response::ok("async get all users!")
        })
        .post_async("/users/async", |_, _| async {
            Response::ok("async created.")
        })
        .patch_async("/users/async", |_, _| async {
            Response::ok("async patched.")
        })
        .delete_async("/users/:id/async", |_, _| async {
            Response::ok("async deleted.")
        })
        .run(req, env)
        .await
}
```

ルーティングを設定するには、`worker`クレートの`Router`を使います。
注意点としては、同期処理と非同期処理でそれぞれのメソッドが用意されているところですね。

今の状態で、色々なパスにリクエストを送って、それぞれのレスポンスを確認してみてください。

### 4. テキストファイルを読み取る

肩慣らし代わりに、テキストファイルを読み取ってみましょう。

`src/lib.rs`を以下のように書き換えます。

```rust
use worker::*;

#[event(fetch)]
async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let router = Router::new();

    router
        .post_async("/upload", |mut req, _| async move {
            match req.form_data().await {
                Ok(form) => match form.get("text_file") {
                    Some(FormEntry::File(file)) => {
                        Response::ok(String::from_utf8(file.bytes().await?).unwrap())
                    }
                    _ => Response::error("不正なリクエストです", 400),
                },
                Err(e) => Response::error(format!("不正なリクエストです: {}", e), 400),
            }
        })
        .run(req, env)
        .await
}
```

テキストファイルを読み取って、中身をそのままレスポンスとして返すだけのシンプルなエンドポイントです。
form-dataからファイルを取り出し、文字列に変換してレスポンスに返しています。

好きなAPIクライアントを使って、リクエストを送ってみましょう。
自分はVSCodeのThunder Clientを使って、以下のようにリクエストを送ってみました。

![image](https://storage.googleapis.com/zenn-user-upload/9e932f217b48-20230717.png)

テキストファイルの中身がレスポンスとして返ってきていますね。
こうなっていればここまでは成功です。

### 5. 画像ファイルを読み取る

次に、リクエストで受け取った画像をそのままレスポンスとして返してみましょう。

`src/lib.rs`を以下のように書き換えます。

```rust
use worker::*;

#[event(fetch)]
async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
    let router = Router::new();

    router
        .post_async("/upload", |mut req, _| async move {
            match req.form_data().await {
                Ok(form) => match form.get("image_file") {
                    Some(FormEntry::File(file)) => match file.type_().as_str() {
                        "image/png" => {
                            let response = Response::from_bytes(file.bytes().await?)?;
                            let mut headers = Headers::new();
                            headers.set("content-type", "image/png")?;
                            Ok(response.with_headers(headers))
                        }
                        n => Response::error(
                            format!("許可されていないファイル種別です： {}", n),
                            400,
                        ),
                    },
                    _ => Response::error("不正なリクエストです", 400),
                },
                Err(e) => Response::error(format!("不正なリクエストです: {}", e), 400),
            }
        })
        .run(req, env)
        .await
}
```

画像ファイルを読み取って、そのままレスポンスとして返しています。
今回は、ファイル種別を見て、`png`以外であればエラーを返すようにしています。
また、画像を返す場合には、`content-type`を適切に設定する必要があるので、注意しましょう。

こちらも好きなAPIクライアントを使って、リクエストを送ってみましょう。
以下のように送信した画像が返って来ていれば成功です。

![image](https://storage.googleapis.com/zenn-user-upload/ff06c7a94187-20230717.png)

### 6. 画像ファイルに文字列を合成する

やっと土台が整いました！
それでは画像に文字列を合成する処理を実装していきましょう。

まずは、今回使用する`crate`を追加します。

```bash
cargo add image
cargo add imageproc
cargo add rusttype
```

次に、フォントファイルを追加します。
今回は、[M+ FONTS](https://mplus-fonts.osdn.jp/)の[M+ 1p](https://mplus-fonts.osdn.jp/about.html)を使います。

```bash
mkdir assets

# ダウンロードしたフォントファイルをassetsディレクトリに配置してください
```

続いて、`src/lib.rs`を以下のように書き換えます。

```rust
use rusttype::{Font, Scale};
use worker::*;

#[event(fetch)]
async fn main(req: Request, env: Env, _ctx: Context) -> Result<Response> {
  let router = Router::new();

  router
    .post_async("/upload", |mut req, _| async move {
      let form_data = match req.form_data().await {
        Ok(form) => form,
        _ => return Response::error("不正なリクエストです", 400),
      };

      let background_image = match form_data.get("background_image") {
        Some(FormEntry::File(file)) => {
          let file = match file.type_().as_str() {
            "image/png" => file,
            "image/jpeg" => file,
            _ => return Response::error("不正なリクエストです", 400),
          };

          match image::load_from_memory(&file.bytes().await?) {
            Ok(image) => image,
            Err(e) => return Response::error(format!("画像の読み込みに失敗しました: {}", e), 500),
          }
        }
        _ => return Response::error("不正なリクエストです", 400),
      };

      let text = match form_data.get("text") {
        Some(FormEntry::Field(text)) => {
          if text.trim().is_empty() {
            return Response::error("不正なリクエストです", 400);
          } else {
            text
          }
        }
        _ => return Response::error("不正なリクエストです", 400),
      };

      let font = Vec::from(include_bytes!("../assets/Mplus1-Black.ttf") as &[u8]);
      let font = match Font::try_from_vec(font) {
        Some(font) => font,
        _ => return Response::error("フォントの読み込みに失敗しました", 500),
      };

      let scale = Scale { x: 100.0, y: 100.0 };

      let h = background_image.height();
      let w = background_image.width();

      let point = rusttype::point(0.0, font.v_metrics(scale).ascent);
      let glyphs = font
        .layout(&text, scale, point)
        .map(|g: rusttype::PositionedGlyph<'_>| g.pixel_bounding_box())
        .filter(|g| g.is_some())
        .collect::<Vec<_>>();

      let first_x = glyphs.first().unwrap().unwrap().min.x;
      let last_x = glyphs.last().unwrap().unwrap().max.x;
      let min_y = glyphs.iter().map(|g| g.unwrap().min.y).min().unwrap();
      let max_y = glyphs.iter().map(|g| g.unwrap().max.y).max().unwrap();

      let total_height = max_y - min_y;
      let total_width = last_x - first_x;

      let center_x = (w / 2) - (total_width / 2) as u32 - first_x as u32;
      let center_y = (h / 2) - (total_height / 2) as u32 - min_y as u32;

      let composite_image = imageproc::drawing::draw_text(
        &background_image,
        image::Rgba([255u8, 255u8, 255u8, 255u8]),
        center_x as i32,
        center_y as i32,
        scale,
        &font,
        &text.replace('_', " "),
      );

      let mut buffer = std::io::Cursor::new(vec![]);
      match composite_image.write_to(&mut buffer, image::ImageOutputFormat::Png) {
        Ok(_) => {}
        Err(e) => return Response::error(format!("画像の書き込みに失敗しました: {}", e), 500),
      }

      let mut headers = Headers::new();
      match headers.set("content-type", "image/png") {
        Ok(_) => {}
        Err(e) => return Response::error(format!("画像の書き込みに失敗しました: {}", e), 500),
      };

      let response = match Response::from_bytes(buffer.into_inner()) {
        Ok(response) => response,
        Err(e) => return Response::error(format!("画像の書き込みに失敗しました: {}", e), 500),
      };
      Ok(response.with_headers(headers))
    })
    .run(req, env)
    .await
}
```

簡単にコードの解説をしておきます。

まずは、受け取ったリクエストやフォントを読み込み、処理の準備を行っています。
Rustの `match`式で早期リターン的な書き方をするのが、個人的にとても読みやすくて好きなので、よく使います。

```rust
let form_data = match req.form_data().await {
    Ok(form) => form,
    _ => return Response::error("不正なリクエストです", 400),
};

let background_image = match form_data.get("background_image") {
    Some(FormEntry::File(file)) => {
        let file = match file.type_().as_str() {
            "image/png" => file,
            "image/jpeg" => file,
            _ => return Response::error("不正なリクエストです", 400),
        };

        match image::load_from_memory(&file.bytes().await?) {
            Ok(image) => image,
            Err(e) => return Response::error(format!("画像の読み込みに失敗しました: {}", e), 500),
        }
    }
    _ => return Response::error("不正なリクエストです", 400),
};

let text = match form_data.get("text") {
    Some(FormEntry::Field(text)) => {
        if text.trim().is_empty() {
            return Response::error("不正なリクエストです", 400);
        } else {
            text
        }
    }
    _ => return Response::error("不正なリクエストです", 400),
};

let font = Vec::from(include_bytes!("../assets/Mplus1-Black.ttf") as &[u8]);
let font = match Font::try_from_vec(font) {
    Some(font) => font,
    _ => return Response::error("フォントの読み込みに失敗しました", 500),
};
```

次に、文字列を描画する座標の計算を行います。
今回は中央に表示できるように座標を計算しています。
[こちらの記事](https://cordea.hatenadiary.com/entry/2019/06/16/153843)を参考に（ほぼパクり）で実装しています。

自分なりに解釈したコードコメントを書いているので、そちらを参考にしてください。

```rust
let scale = Scale { x: 100.0, y: 100.0 };

// 背景画像の横幅と縦幅を取得
let h = background_image.height();
let w = background_image.width();

// 全部の文字を見ていって、左下の座標と右上の座標を取得する。
let point = rusttype::point(0.0, font.v_metrics(scale).ascent);
let glyphs = font
    .layout(&text, scale, point)
    .map(|g: rusttype::PositionedGlyph<'_>| g.pixel_bounding_box())
    //   対応していない文字がきたら、g.pixel_bounding_box() は None を返すので、それを除外する
    .filter(|g| g.is_some())
    .collect::<Vec<_>>();

// 計算に使う座標を取得する。
let first_x = glyphs.first().unwrap().unwrap().min.x;
let last_x = glyphs.last().unwrap().unwrap().max.x;
let min_y = glyphs.iter().map(|g| g.unwrap().min.y).min().unwrap();
let max_y = glyphs.iter().map(|g| g.unwrap().max.y).max().unwrap();

// 一番高い文字の上端と一番低い文字の下端の差分を取る → 文字列全体の高さ
let total_height = max_y - min_y;
// 最初の文字の左端と最後の文字の右端の差分を取る → 文字列全体の幅
let total_width = last_x - first_x;

// 文字列を中央に配置するための座標を計算
let center_x = (w / 2) - (total_width / 2) as u32 - first_x as u32;
let center_y = (h / 2) - (total_height / 2) as u32 - min_y as u32;
```

そして最後に画像と文字列を合成してレスポンスを返しますが、こちらは特に解説するところもないのでスキップします。

それではリクエストを送ってみましょう！
文字列が中央に表示されていれば、バッチリです！

![image](https://storage.googleapis.com/zenn-user-upload/8082babaa988-20230722.png)

今は文字のフォントや大きさや色などは固定になってしまっているのですが、このあたりを自由に変えたりできるようになると更に楽しそうです！

それはまたの機会に...！

### 7. デプロイ

Coming soon...

## まとめ

いかがだったでしょうか

思っていたよりも簡単に画像を合成する処理が実装できました。
自分は最近Cloudflareを使い始めたので、まだ知らないことも多いですが、コストも低いですし、色々と楽しめそうです。

そして何よりRustは読みやすいですね！！！

CloudflareとRustと画像と文字列合成というかなりニッチな記事ではありますが、何かの参考になれば嬉しいです。
