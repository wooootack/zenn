---
title: "RustでAPIを開発してみたら結構辛かった話"
emoji: "😺"
type: "tech" # tech: 技術記事 / idea: アイデア
topics:
  - Rust
  - DDD
  - ドメイン駆動設計
  - モジュラモノリス
  - API
published: true
publication_name: "praha"
---

## はじめに

皆様こんにちは、[株式会社プラハ](https://www.praha-inc.com/)のAwataです。
今日は、以前書いた[リーダーの振り返り記事](https://zenn.dev/praha/articles/6ca14058507cb1)で軽く触れていた、RustでのAPI開発についての記事を書いていこうと思います。

結論RustでWebは辛い！という話なんですが、約5か月くらいRustでWeb開発をしたので、今後の参考になるようなことを書いていこうと思います。
ぜひ最後までお付き合いください。

## TL;DR

- RustでWeb開発はまだ早いかもしれない。
- RustでDDDはやりやすい。ただしDIがやりにくい場合があるので、そこは要注意。
- Rustはモジュールの仕組みが協力なので、モジュラモノリスはやりやすい。
- [サンプルリポジトリ](https://github.com/wooootack/rust-ddd-sample)はこちら
- Rustはやっぱり難しいけど人気の理由も少し分かった気がする

## 工夫したところ

辛かった話なんですが、辛い！辛い！と書いても誰も嬉しくないと思うので、その辛さを乗り越えるために工夫したところを紹介しようと思います。
これで辛さが2割くらい削減されたような気がします。

### 統合テストと単体テストを別々に動かせるようにした

自分は統合テストは実行コストが高いため、単体テストとは別々で動かしたいと考えています。

そして、Rustには[簡単に単体テストや統合テストを書ける仕組み](https://doc.rust-jp.rs/book-ja/ch11-00-testing.html)が用意されています。
単体テストは同じファイルに気軽に書けて、統合テストも特殊な書き方を覚える必要もなく簡単に書けるため、とても良い仕組みだと思っています。
しかし、**統合テストだけを動かすコマンドは自分の調べた限りでは用意されていません**でした。

そこで、[フィーチャーフラグ](https://doc.rust-lang.org/cargo/reference/features.html)を使って、統合テストだけを動かせるようにしました。

手順は以下の通りです。

#### 1. `Cargo.toml` に以下のようにフィーチャーフラグを定義する

```toml
[package]
# 省略

[dependencies]
# 省略

[features]
integration_test = []
```

#### 2. testsディレクトリに配置している統合テストに、以下のようにフィーチャーフラグを指定する

```rust
#[cfg(feature = "integration_test")]
mod test {
  // 省略
}
```

#### 3. [テスト実行時にフィーチャーフラグを指定](https://doc.rust-lang.org/cargo/commands/cargo-test.html#feature-selection)する

```bash
cargo test --features integration_test
```

### Fromトレイトを積極的に使って変換処理を楽にした

今回の設計では、doman/usecase/presentation/infra/scenario と言った具合にレイヤーが多く存在しています。
そして、レイヤー間でのデータのやり取りをする際は、DTO的な構造体を定義して使っています。
そしれ、それぞれの構造体は同じようなプロパティを持っていることが多く、詰め替え作業が割と面倒です。

※ Rustは公称型のため、例え全てのプロパティが同じでも必ず詰め替え作業が必要です

これまでの自分なら、以下のような`from_xxx`といったメソッドを定義していたと思います。

```rust
// イメージを共有するためだけのコードなのでコンパイルできません

let usecase_result = usecase::execute();
PresentationResult::from_usecase_result(usecase_result);
```

しかし、このコードはあまりRustらしくないですね。
Rustでは、[Fromトレイト](https://doc.rust-jp.rs/rust-by-example-ja/conversion/from_into.html)が用意されているため、それを使うと以下のように書けます。

```rust
// イメージを共有するためだけのコードなのでコンパイルできません

let usecase_result = usecase::execute();
PresentationResult::from(usecase_result);
```

これだけ見ると、**なんかちょっと短くなっただけじゃん**と思うかもしれませんが、Fromトレイトを他の構造体にも実装してあげれば、他の構造体も同じように`from`メソッドを通じて変換できるようになります。

### `?` 演算子を積極的に使ってエラーハンドリングを楽にした

Rustには例外は存在せず、基本的には`Result`型を使ってエラーハンドリングを行います。

※[パニック](https://doc.rust-jp.rs/book-ja/ch09-01-unrecoverable-errors-with-panic.html)も存在しますが、これは原則回復不能なエラーに対して使うため、ハンドリングすることはあまりありません。

サンプルリポジトリの[こちら](https://github.com/wooootack/rust-ddd-sample/blob/53e03f779fc88dbafb60b01d371c08d3cba2ea59/src/scenarios/find_one_user_scenario.rs#L39)に以下のようなコードがありますが、ここで`?`演算子を使わずに書くと以下のようになります。

```rust
pub async fn execute(
  context: web::Data<RequestContext>,
  user_id: web::Path<String>,
) -> Result<HttpResponse, ApiError> {
  let request = FindOneUserRequest {
    id: user_id.to_string(),
  };

  let response: Result<Result<Option<UserResponse>, ApiError>, BlockingError> = web::block(move || {
    let conn = context.get_connection();

    find_one_user::execute(conn, request)
  })
  .await;

  match response {
    Ok(response_2) => match response_2 {
      Ok(response_3) => match response_3 {
        Some(response_3) => Ok(HttpResponse::Ok().json(UserScenarioResponse::from(response_3))),
        None => Ok(HttpResponse::Ok().body("User not found")),
      },
      Err(_) => Ok(HttpResponse::Ok().body("User not found")),
    },
    Err(_) => Ok(HttpResponse::Ok().body("User not found")),
  }
}
```

地獄のように`match`式がネストしていますね。
`web::block`の戻り値が、`Result<T, BlockingError>`で、`find_one_user::execute`の戻り値が`Result<Option<UserResponse>, ApiError>`で、これが先ほどの`T`に入るため、このような型になってしまいます。

これを簡略化するために`?`演算子を使う必要があり、`?`演算子を使えるようにするには（このコードにおいては）`From`トレイトを通って、ApiErrorに変換されるように定義しておく必要があります。
そしてその変換処理の[こちら](https://github.com/wooootack/rust-ddd-sample/blob/develop/src/modules/common/api_error.rs)にあるため、上記のコードは以下のようになります。

```rust
pub async fn execute(
  context: web::Data<RequestContext>,
  user_id: web::Path<String>,
) -> Result<HttpResponse, ApiError> {
  let request = FindOneUserRequest {
    id: user_id.to_string(),
  };

  let response = web::block(move || {
    let conn = context.get_connection();

    find_one_user::execute(conn, request)
  })
  .await??;

  match response {
    Some(response) => Ok(HttpResponse::Ok().json(UserScenarioResponse::from(response))),
    None => Ok(HttpResponse::Ok().body("User not found")),
  }
}
```

match式のネストがなくなり、コードが簡潔になりました。
もっと詳しく知りたい方は[こちら](https://doc.rust-jp.rs/book-ja/ch09-02-recoverable-errors-with-result.html#%E3%82%A8%E3%83%A9%E3%83%BC%E3%82%92%E5%A7%94%E8%AD%B2%E3%81%99%E3%82%8B)を読んでみることをおすすめします

### 認証済みのリクエストしかアクセスできないような制御をする仕組みを用意した

前提として「APIサーバーの前にBFFがあり、APIはプライベートネットワークに置いてあり、BFFからのリクエストしか受け付けないように設定されている」という状態です。
そして今回は、BFFからのリクエストに認証済みのユーザー情報を付与し、APIサーバーではその情報が正しく設定されているかどうかを検証するという構成にしました。
[こちらのやり方](https://tech.emotion-tech.co.jp/entry/actix-web-from-request)をかなり参考にしていますので、ぜひこちらも併せて読んでみてください。

まずは`FromRequest`トレイトを使って、ヘッダーから必要な情報を取得して`AuthUser`という認証済みユーザーを表す構造体に変換する処理を実装します。

まずは`AuthUser`構造体を定義しましょう

```rust
pub struct AuthUser {
  pub id: String,
  pub mail_address: String,
  pub role: String,
}

impl AuthUser {
  pub fn is_admin(&self) -> bool {
    self.role == "admin"
  }
  pub fn is_user(&self) -> bool {
    self.role == "user"
  }
}
```

そして`FromRequest`トレイトを実装します。
ここでヘッダーの中身を見て、送られてきたIDは本当に存在するのか？など色々なチェックをすることも可能です。

```rust
impl FromRequest for AuthUser {
  type Error = ApiError;
  type Future = Pin<Box<dyn Future<Output = Result<Self, Self::Error>>>>;

  fn from_request(req: &HttpRequest, _payload: &mut Payload) -> Self::Future {
    let request = req.clone();
    Box::pin(async move {
      let role = match request.headers().get("auth-user-role") {
        Some(id) => id.to_str(),
        None => return Err(ApiError::new(StatusCode::FORBIDDEN, "forbidden".to_owned())),
      }
      .unwrap();
      if role != "user" && role != "admin" {
        return Err(ApiError::new(StatusCode::FORBIDDEN, "forbidden".to_owned()));
      }
      let id = match request.headers().get("auth-user-id") {
        Some(id) => id.to_str(),
        None => return Err(ApiError::new(StatusCode::FORBIDDEN, "forbidden".to_owned())),
      }
      .unwrap();
      let mail_address = match request.headers().get("auth-user-email") {
        Some(id) => id.to_str(),
        None => return Err(ApiError::new(StatusCode::FORBIDDEN, "forbidden".to_owned())),
      }
      .unwrap();
      Ok(AuthUser {
        id: id.to_string(),
        mail_address: mail_address.to_string(),
        role: role.to_string(),
      })
    })
  }

  fn extract(req: &HttpRequest) -> Self::Future {
    Self::from_request(req, &mut Payload::None)
  }
}
```

そして、この`AuthUser`を`scneario`に定義されている関数の引数に設定します。
これによって`actix-web`が自動的に`AuthUser`を取得してくれるようになり、`FromRequest`の変換中にエラーが起きた際は自動的にエラーレスポンスが返ります。

```rust
pub async fn execute(req_user: AuthUser) -> Result<HttpResponse, ApiError> {
  // 省略
}
```

本番用のコードでは、AuthUser以外に`AdminUser`, `NormalUser`のように更に権限に応じた構造体を定義することで、権限に応じたアクセス制限が簡単にできるようにしてあります。
サンプルリポジトリではコードが増えすぎるし（ちょっと疲れてきてた）ので省略しましたが、もし気になるけどやり方が分からない！という方はコメントで教えてください！
（元気があれば追記します）

### scenario層を用意した

`scenario層`は複数のモジュールの`presentation層`に定義されてある関数を呼び出して処理を進める層です。
複数の`presentation`層の関数を呼んでも良いですし、1つだけでも良いです。
そして、必要であればトランザクションの管理も行います。

ここまで読んでいて勘の良い人は、なんかこれ何かの役割と似てるな？と思ったかもしれませんが、`scenario`層はマイクロサービスにおけるSagaパターンのオーケストレーター的な役割を果たします。

これ以外にも、**ルーティングをどうやって一元で管理するか？**という問題も`scenario`層は解決してくれます。
`scenario`層が存在しない場合、各モジュールでルーティングを定義する形式になってしまい、他のモジュールでそのルーティングが使われていないかを都度確認しなくてはならなくなります。
そのため、例え`presentation`層の関数を呼ぶだけであっても必ず`scenario`層を用意して、「HTTPエンドポイントと紐づけるのは`scenario`層の関数だけ」というルールで運用しています。

## 良かったところ

続いてRustの良かったところです。
良かったところも多くあったんですが、それでもやっぱり辛いということは、つまりそういうことなのです。

### モジュールの可視性を細かく設定できる

あまり細かく書くとこれだけで1つの記事になってしまいそうなので、箇条書きで良かったと感じたところを書いてみます。

- ルートからモジュールを宣言していく必要があるので、モジュールの階層構造が明確になる
- 基本が`private`なので、知らず知らず公開してしまっているものがない
- 公開範囲を細かく設定できるので、全公開が全非公開か！みたいな2択を迫られない
  - 例
  - `pub`: 外部クレートにも公開
  - `pub(super)`: 親モジュールには公開
  - `pub(crate)`: 現在のクレート内には公開

もっと詳しく知りたい方はまず[こちら](https://doc.rust-jp.rs/rust-by-example-ja/mod.html)を読むことをおすすめします。

### 構造体の定義と独自実装、トレイトの実装を分けて書けるので見通しが良い

これは自分の経験が浅いからかもしれないのですが、過去に使っていたクラスベースの言語では、以下のすべてが同じクラスに書かれることが多かったです

- プロパティ
- 独自のメソッド
- インターフェースを継承したメソッド（Rustだとトレイトの実装になります）

この状態になると、まずプロパティを上の方に書いて、次に独自のメソッドを書いて、最後にインターフェースを継承したメソッドを書く、みたいな暗黙の了解が生まれることが多かったです。
そのため、コードに手を入れる時も若干気を使ったり、、、という感じでした。

しかし、**Rustではこれらを全て別々に書くことができます。**

具体的なコードはこんな感じですね。

```rust
// 構造体の定義
pub struct UserId {
  pub value: String,
}

// 独自の振る舞いの定義
impl UserId {
  pub fn restore(value: String) -> Self {
    let value = Ulid::from_string(&value).unwrap().to_string();
    Self { value }
  }
}

// トレイトの実装
impl Default for UserId {
  fn default() -> Self {
    let value = Ulid::new().to_string();
    Self { value }
  }
}

// トレイトの実装
impl TryFrom<String> for UserId {
  type Error = String;

  fn try_from(value: String) -> Result<Self, Self::Error> {
    let ulid = Ulid::from_string(&value);
    match ulid {
      Ok(value) => Ok(Self {
        value: value.to_string(),
      }),
      Err(err) => Err(format!("can't convert to AdminId. error: {err}")),
    }
  }
}
```

どうでしょう？めっちゃ読みやすくないですか？（急に感覚的な話を出して申し訳ないですが）

### 静的解析が強いので、脳死で従うだけでも統一感のあるコードに近づいていく

Rustには、コードの静的解析を行うツールがいくつかあります。
自分が導入したのは、有名どころの以下2つです。

- [rustfmt](https://doc.rust-jp.rs/book-ja/appendix-04-useful-development-tools.html#rustfmt%E3%82%92%E4%BD%BF%E3%81%A3%E3%81%9F%E8%87%AA%E5%8B%95%E3%83%95%E3%82%A9%E3%83%BC%E3%83%9E%E3%83%83%E3%83%88)
- [clippy](https://doc.rust-jp.rs/book-ja/appendix-04-useful-development-tools.html#clippy%E3%81%A7%E3%82%82%E3%81%A3%E3%81%A8lint%E3%82%92)

JavaScript界隈に例えていうならば、rustfmtはprettier、clippyはESLintです。
導入や設定もとても簡単なので、チームで開発する際は必ず入れておいて間違いないと思います。

## 苦労したところ

最後に苦労したところです。
全体的に、それRustが悪い訳じゃなくて、まだWeb開発で使われてないだけやん？という感じです。

### ライフタイムが今でも理解できていない気がしているくらい難しかった

Rustの勉強をしていると、これらの言葉をよく聞きますよね

- [所有権](https://doc.rust-jp.rs/book-ja/ch04-01-what-is-ownership.html)
- [参照と借用](https://doc.rust-jp.rs/book-ja/ch04-02-references-and-borrowing.html)

これらも非常に難しい概念だと思いましたが、自分にとって[ライフタイム](https://doc.rust-jp.rs/book-ja/ch10-03-lifetime-syntax.html)は群を抜いて難しかったです。
今もライフタイムとは？と言われるとうまく説明できる自信はありません。

関数定義の場合はまだ分かりやすいですが、特に構造体の定義において、ライフタイムをどう定義するかが難しかったです。

具体例をいい感じに書けず申し訳ないのですが、**構造体に参照を保持させる**場合は特に注意してほしいです。
可能なら構造体を使わず関数でうまく書ける方法を探すことを自分はおすすめしたいです。

### 可変参照を色々なところで使わざるを得ない状況になってしまった

Rustの可変参照は、**同時に1つしか存在できない**という制約があります。
今回は[diesel](https://diesel.rs/)というORMを使ったのですが、このORMが提供しているDBとのコネクションオブジェクトが可変参照でした。
そしてDDDのやり方を参考にしていたので、ユースケースやドメインサービスにこのコネクションオブジェクトを渡す必要がありました。
（ここはもっとうまく設計すれば、良いやり方があったかもしれないと思っています）

そのため、今回は苦肉の策として、[`Rc`と`RefCell`を組み合わせて無理やり可変参照を複数の場所から参照できるようにしました。](https://doc.rust-jp.rs/book-ja/ch15-05-interior-mutability.html#rct%E3%81%A8refcellt%E3%82%92%E7%B5%84%E3%81%BF%E5%90%88%E3%82%8F%E3%81%9B%E3%82%8B%E3%81%93%E3%81%A8%E3%81%A7%E5%8F%AF%E5%A4%89%E3%81%AA%E3%83%87%E3%83%BC%E3%82%BF%E3%81%AB%E8%A4%87%E6%95%B0%E3%81%AE%E6%89%80%E6%9C%89%E8%80%85%E3%82%92%E6%8C%81%E3%81%9F%E3%81%9B%E3%82%8B)

ただしこの書き方をした場合、実行時までエラーに気づけない可能性があります。
これまではコンパイラが「可変参照を複数のところで使っているよ！直してね！」と怒ってくれていましたが、この書き方だとコンパイラは何も言ってくれません。
しかし、実装が間違っていて複数の場所から可変参照が使われた場合、実行時エラーとなってしまいます。

今回はこれ以外の解決方法を出せなかったのでこの書き方を採用しましたが、もし他に良い方法があれば教えていただきたいです。

### 構造体をJSON化したいだけなのに、コードがいっぱい必要だった

例えばJavaScriptでJSON文字列化したい場合、`JSON.stringify`を使えば簡単にできます。
プロパティにDateオブジェクトがあったとしても、特に意識することなく使えますね。

しかし、Rustはそう簡単には行きません。
具体的にどういう手順を踏んだかを書いてみます。

今回は[serde](https://serde.rs/)というクレートを使ってJSON化の処理を実装しました。

#### 1. Cargo.tomlにserdeの依存関係を追加する

```toml
[package]
# 省略

[dependencies]
# 省略
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

#### 2. JSON化したい構造体の定義

今回は、`scenario`層で使われているレスポンスの構造体を例に挙げてみます。

```rust
pub struct UserScenarioResponse {
  pub id: String,
  pub first_name: String,
  pub last_name: String,
  pub mail_address: String,
  pub age: i16,
  pub created_at: DateTime<Utc>,
  pub updated_at: DateTime<Utc>,
}
```

#### 3. JSON化したい構造体に`Serialize`トレイトを実装する

今回はderiveマクロを使って、自動生成してみましょう。
独自のシリアライズ処理を実装したい場合は、deriveマクロを使わずに自分で実装することもできます。
（試しに一度やってみましたが、まあまあ大変だったので自分はもうやりたくないですｗ）

```rust
#[derive(Serialize)]
pub struct UserScenarioResponse {
  pub id: String,
  pub first_name: String,
  pub last_name: String,
  pub mail_address: String,
  pub age: i16,
  pub created_at: DateTime<Utc>,
  pub updated_at: DateTime<Utc>,
}
```

#### 4. `DateTime<Utc>`は以下のようにフォーマットを指定する必要がある

今回は`DateTime`がエラーを起こしていましたが、他の型でも同様のエラーが出る可能性があります。
また、構造体のプロパティに構造体があった場合、その構造体も`Serialize`トレイトを実装する必要があります。

```rust
pub mod date_format {
  use chrono::{DateTime, TimeZone, Utc};
  use serde::{self, Deserialize, Deserializer, Serializer};

  const FORMAT: &str = "%Y-%m-%d %H:%M:%S";

  pub fn serialize<S>(date: &DateTime<Utc>, serializer: S) -> Result<S::Ok, S::Error>
  where
    S: Serializer,
  {
    let s = format!("{}", date.format(FORMAT));
    serializer.serialize_str(&s)
  }
  pub fn deserialize<'de, D>(deserializer: D) -> Result<DateTime<Utc>, D::Error>
  where
    D: Deserializer<'de>,
  {
    let s = String::deserialize(deserializer)?;
    Utc
      .datetime_from_str(&s, FORMAT)
      .map_err(serde::de::Error::custom)
  }
}

```

```rust
#[derive(Serialize)]
pub struct UserScenarioResponse {
  pub id: String,
  pub first_name: String,
  pub last_name: String,
  pub mail_address: String,
  pub age: i16,
  #[serde(with = "date_format")]
  pub created_at: DateTime<Utc>,
  #[serde(with = "date_format")]
  pub updated_at: DateTime<Utc>,
}
```

これでやっとシリアライズができるようになりました！
一度実装してしまえば他の構造体にも流用できるのですが、やはり面倒だなと感じました。

### （自分的に）使いやすいORMがなかった

今回は`diesel`, `sea-orm`, `sqlx`あたりを検討して、情報量の多さや実績のある`diesel`を採用しました。
他のORMを細かく試した訳ではないのですが、やはりエコシステムの弱さは感じました。

RailsのActiveRecordまでとは言いませんが、もう少し使いやすいORMがあればいいなと思いました。
Rustの言語仕様上仕方ない部分があるとは思いますが、やはりコード量が多くなったり大量の構造体の定義が必要になることが多々ありました。

どこまで簡単になるのかは分かりませんが、Web開発においてORMの強さはかなり重要だと思うので、もっと進歩すると嬉しいなと感じました。

### （自分的に）テストが書きやすいとは言えなかった

これもRustが悪いというより、Web開発におけるエコシステムがまだ充実していないということだと感じました。
DBまで繋げて統合テストを書きたい時に、マイグレーションの実行やテストデータの投入など、処理を挟むみたいなことがあまりできないイメージです。

例えばTypeScriptで開発した際は、Jestを使って色々なテストを簡単に書くことができました。
このあたりは今後に期待したいです。

## まとめ

今回Rustを書いてみて、自分はかなり好きな部類に入ったので、今後もっともっと人気になっていくと嬉しいなと思います。
そのためにも、この記事を参考にしてみなさんがWeb開発でもRustをどんどん採用してくれると嬉しいです。
