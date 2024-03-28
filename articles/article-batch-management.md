---
title: ZennとQiitaの2重管理を解消してみた
emoji: ⚔️
type: tech
topics:
- Zenn
- Qiita
- Rust
- clap
- markdown
published: true
---

## 概要

ZennとQiitaに同じ記事を投稿する際には、同じ内容を2つのファイルで管理する必要があります。変更があるたび、忘れずにコピー&ペーストをして、プラットフォーム独自の記法[^abstract.1]を書き換えなくてはなりません。手間がかかる上に、手作業なのでミスが起こるかもしれません。

[^abstract.1]: 例えば、注意書きを挿入する際に、Zennでは、独自の`:::message`記法を使うことができます。Qiitaでは、独自の`:::note`記法を使うことができます。

- 複数の場所で同じ記事を管理するのが面倒
- プラットフォーム独自の記法を手作業で書き換えるのが面倒

これらの問題を解決するために、Rust言語で簡単なツールを作ってみたのでご紹介します。「**Zeta**」と呼ぶことにします。

https://github.com/TyomoGit/zeta


:::details 動作確認環境

- MacBook Pro 14インチ 2021 M1 Pro

```sh
$ sw_vers
ProductName:            macOS
ProductVersion:         14.4.1
BuildVersion:           23E224
```
```sh
$ node --version
v21.7.1

$ npm --version
10.5.0

$ npx zenn --version
(node:11561) [DEP0040] DeprecationWarning: The `punycode` module is deprecated. Please use a userland alternative instead.
(Use `node --trace-deprecation ...` to show where the warning was created)
0.1.153

$ npx qiita version
1.4.0

$ rustc --version
rustc 1.76.0 (07dca489a 2024-02-04) (Homebrew)

$ cargo --version
cargo 1.76.0
```
```sh
$ git --version
git version 2.44.0
```

- Rust 2021 edition
- `Cargo.lock`はGitHubリポジトリにあります
```toml:Cargo.toml 依存関係の定義
[dependencies]
clap = { version = "4.5.3", features = ["derive"] }
serde = { version = "1.0.197", features = ["derive"] }
serde_yaml = "0.9.33"
toml = "0.8.12"
```


```sh
$ code --version
1.87.2
863d2581ecda6849923a2118d93a088b0745d9d6
arm64

$ code --list-extensions --show-versions | grep "runonsave"
emeraldwalk.runonsave@0.2.0

$ firefox --version
Mozilla Firefox 124.0.1
```
:::

## 方針

このツールを使って以下のことが実現できるようにします。
- 簡単なコマンドでZennとQiita両方のファイルを管理
- マークダウンファイルをZenn/Qiita形式に変換(ビルドと呼んでいます)
- 保存時に自動でビルド（VSCode）
- ビルド後にmainブランチにプッシュで公開

## 仕様

このツールでは、ひとつのファイルに記事を書き、それを変換してZenn/Qiitaに対応するという方法を採りました。記法はZennライクな独自記法です。

:::details Front Matterとは
Front MatterはZenn CLIやQiita CLIを使って記事を書く際に、ファイルの一番上の、yaml形式でタイトルなどを定義できる部分のことです。CLIを使って管理をする場合、Zennでは以下のように記述します。
```yaml
---
title: "title"
emoji: "👍"
type: "info"
topics: ["topic1", "topic2"]
published: false
---
```
:::

:::details Zennのマークダウンと違うところ
- Front Matter（記事の最初に書くyaml）に`only`フィールドを指定できる（optional）
    - 特定のプラットフォームのみに変換するよう指定できる
    - 「Zennだけ」、「Qiitaだけ」への変換に対応できる
- `<macro>`記法
    - マクロ機能（後述）
- `:::message`が3種類ある（`info`、`warn`、`alert`）
    - Qiita向けの対応

:::

:::details Qiitaのマークダウンと違うところ
- Front Matterのフィールド
- Front Matter（記事の最初に書くyaml）に`only`フィールドを指定できる（optional）
    - 特定のプラットフォームのみに変換するよう指定できる
    - 「Zennだけ」、「Qiitaだけ」への変換に対応できる
- `<macro>`記法
    - マクロ機能（後述）
- `:::note`ではなく`:::message`を使う
- `<details>`ではなく`:::details`を使う
- インライン脚注が使える
    - `^[内容]`
- 空行がなくても、改行で囲まれたリンクはカードになる
:::


ディレクトリ構造は次のようになります。

```txt
.
├── .gitignore
├── .github
├── README.md
├── articles
├── books
├── public
├── zeta
├── images
├── Zeta.toml
└── qiita.config.json
```
`npx zenn init`と`npx qiita init`を実行した後に、Zeta用のディレクトリ・ファイルを追加した状態です。
`zeta/`内に記事ファイルを作り、そこに書き込んでいきます。`images/`内に画像ファイルを置きます。変換を行うと、Zenn用の記事が`articles/`に、Qiita用の記事が`public/`に生成されます。



### このツールがすること

このツールが行うことを確認します。このツールの「変換」では、以下のことをします。

#### URLの上下に空行を入れる
URLをカード形式で表示するため、Qiitaでは上下に空行が必要です。

#### インライン脚注を展開する

Zennのインライン脚注をQiita向けに展開します。
新しい識別子を生成して、脚注の内容をファイルの一番下に書き込みます。
`インライン脚注のテスト^[あああ]^[いいい]`は次のように変換されます。
```markdown
インライン脚注のテスト[^zeta.inline.1][^zeta.inline.2]

[^zeta.inline.1]: あああ
[^zeta.inline.2]: いいい
```

脚注の識別子は重複しないようにします。

#### `:::message`を調整する

以下のように変換します。

| Zetaの形式 | Zennの形式 | Qiitaの形式 |
| :--- | :--- | :--- |
| `:::message info` | `:::message` | `:::note info` |
| `:::message warn` | `:::message` | `:::note warn` |
| `:::message alert` | `:::message alert` | `:::note alert` |

#### `:::details [title]`を調整する
Zeta・Zennの形式
```
:::details title
content
:::
```

Qiitaの形式
```
<details><summary>title</summary>

content
</details>

```

#### マクロを展開する

マクロは、プラットフォームごとに違う記述をしたいときに使います。`macro`タグの中に、yaml形式でZenn/Qiita用の文字列を指定します。マクロの中にはマクロ以外なら何でも書けます。
```html
- 以前投稿した記事に<macro>
zenn: "Like"
qiita: "いいね"
</macro>を頂きました。嬉しいです。

- 変更履歴は<macro>
zenn: 'この記事のGitHubリポジトリ'
qiita: '「編集履歴」'
</macro>から確認できます。

- いつか<macro>
zenn: 「本」を投稿してみたいなぁ
qiita: アドベントカレンダーに参加してみたいなぁ
</macro>💭
```

実際に展開するとこんな感じです。
👇

- 以前投稿した記事にLikeを頂きました。嬉しいです。

- 変更履歴はこの記事のGitHubリポジトリから確認できます。

- いつか「本」を投稿してみたいなぁ💭

☝️

マクロを使う機会は少ないと思いますが、「 **一元管理にしたが故に不自由** 」という状態をなくすために作りました。「Like」と「いいね」のような、細かな違いを表現するのに便利です。

#### 写真のパスを調整する

`images/`ディレクトリ以下の写真が指定されたときに、パスを調整します。Zennの場合は何も変えません。Qiita向けには、記事のGitHubリポジトリの写真へのリンクに変更します。この機能はGitHubリポジトリがpublicになっていないと完全に動作しません。^[`images/`に写真を置く場合で、リポジトリがprivateになっているとき、Qiitaでは表示できません]privateなリポジトリで写真を掲載する場合は、写真をどこかにアップロードして、そこへのリンクを貼り付けます。このときにマクロを使うこともできます。
```
<macro>
zenn: "![alt](/images/article-slag/photo.png)"
qiita: "![alt](https://url/to/photo.png)"
</macro>
```

## 実装

ソースコードを示しつつ、要点を解説します。
解析では、マークダウンファイルのFront Matterと本文を分けて扱います。すべての実装はGitHubリポジトリを参照してください。

```rust
#[derive(Debug, Clone)]
pub struct MarkdownDoc<F, E> {
    pub frontmatter: F,
    pub elements: Vec<E>,
}
```

### 字句解析

文字列をトークンに分ける作業をします。文章全体を以下の`TokenizedMd`で表します。

```rust
pub type TokenizedMd = MarkdownDoc<String, Token>;
```
```rust
#[derive(Debug, Clone, PartialEq, Eq)]
pub struct Token {
    pub token_type: TokenType,
    pub row: usize,
    pub col: usize,
}

#[derive(Debug, Clone, PartialEq, Eq)]
pub enum TokenType {
    /// string
    Text(String),
    /// http:// or https://
    Url(String),
    /// image
    Image {
        alt: String,
        url: String,
    },
    /// inline footnote
    InlineFootnote(String),
    /// footnote
    Footnote(String),
    /// :::message
    MessageBegin {
        level: usize,
        r#type: String,
    },
    /// :::details
    DetailsBegin {
        level: usize,
        title: String,
    },
    /// :::
    MessageOrDetailsEnd {
        level: usize,
    },
    /// macro
    Macro(TokenizedMacro),
}
```

解析をするために、スキャナを用意します。
```rust
pub struct Scanner {
    source: Vec<char>,

    current: usize,
    start: usize,

    row: usize,
    col: usize,

    tokens: Vec<Token>,
    errors: Vec<ScanError>,
}
```

エラーは`errors`で管理しています。エラーが発生しても、最後まで解析を行うことで、複数のエラーに対応できます。


スキャナは **バッファ** を持っています。現在位置を進める（`advance()`）と、バッファに文字が追加されていきます。バッファを消費する（`consume_buffer()`）と、バッファの文字列を得ると同時に、バッファは空になります。バッファと呼んでいますが、実際に文字のコピーを行っているわけではなく、インデックスを動かすだけです。`start..current`の位置にあたる文字がバッファに入っています。`start`と`current`はシャクトリムシ🐛のように動きます。


```rust
impl Scanner {
    fn advance(&mut self) -> Option<char> {
        let result = self.source.get(self.current).copied();
        self.current += 1;
        if let Some('\n') = result {
            self.row += 1;
            self.col = 1;
        } else {
            self.col += 1;
        }

        result
    }

    fn consume_buffer(&mut self) -> String {
        let result = self
            .source
            .get(self.start..self.current)
            .unwrap_or_default();
        self.start = self.current;
        result.iter().collect()
    }
}
```

トークンの解析に関係する、`!`（写真）、`^`（インライン脚注）や`<`（マクロ）などの文字を「 **興味のある文字** 」と呼ぶことにします。興味のある文字に出会うまで、文字をバッファに溜め込んでいきます。興味のある文字を発見したら、バッファを消費し、`TokenType::Text(String)`としてトークンを作ります。

```rust
fn collect_text(&mut self) {
    let text = self.consume_buffer();
    self.tokens.push(self.make_token(TokenType::Text(text)));
}
```

そして、その興味ある文字に合った処理を実行します。例えば、脚注は次のようにして解析しています。
ここでは、文章中に現われる脚注`[^脚注の識別子]`から識別子を取り出しています。脚注は特に変換の必要性がありませんが、後にすべての脚注の識別子を参照する場面があるため、解析しています。脚注の本文`[^脚注の識別子]: 本文`は無視します。すでに識別子を取り出しているためです。

```rust:matchの一部
'[' => {
    if !self.matches_keyword("[^") {
        self.advance();
        return Ok(());
    }
    let start = self.current;
    self.collect_text();
    self.expect_string("[^");
    self.delete_buffer();
    self.extract_until("]")?;
    let footnote = self.consume_buffer();
    self.expect_string("]");
    self.delete_buffer();
    if self.matches_keyword(":") {
        // [^1]: 脚注
        // 無視する
        self.start = start;
        return Ok(());
    }
    self.tokens
        .push(self.make_token(TokenType::Footnote(footnote)));
}
```

このようにして、マークダウンファイルをトークンに分解します。

### 構文解析

マークダウンの構造を解析します。
```rust
pub type ParsedMd = MarkdownDoc<ZetaFrontmatter, Element>;
```

```rust
#[derive(Debug, Clone, Default, serde::Serialize, serde::Deserialize)]
pub struct ZetaFrontmatter {
    pub title: String,
    pub emoji: String,
    pub r#type: String,
    pub topics: Vec<String>,
    pub published: bool,
    /// compile only specified platform
    #[serde(skip_serializing_if = "Option::is_none")]
    pub only: Option<Platform>,
}

#[derive(Debug, Clone, serde::Serialize, serde::Deserialize, clap::ValueEnum)]
pub enum Platform {
    #[serde(alias = "zenn")]
    Zenn,
    #[serde(alias = "qiita")]
    Qiita,
}
```
```rust
#[derive(Debug, Clone)]
pub enum Element {
    Text(String),
    Url(String),
    Macro(ParsedMacro),
    Image {
        alt: String,
        url: String,
    },
    InlineFootnote(String),
    Footnote(String),
    Message {
        level: usize,
        msg_type: MessageType,
        body: Vec<Element>,
    },
    Details {
        level: usize,
        title: String,
        body: Vec<Element>,
    },
}

#[derive(Debug, Clone)]
pub enum MessageType {
    Info,
    Warn,
    Alert,
}
```

`Element`は`TokenType`と似ています。違うのは、`Message`と`Details`の構造が表わされている点です。この二つの要素はネストする可能性があります[^parse.1]。もしネストしていたら、その解析もこのパートで行います。

[^parse.1]: ただし、`:::message`の中に、さらに`:::message`や`:::details`を入れることはできない仕様です。

パーサです。

```rust
pub struct Parser {
    source: Vec<Token>,
    frontmatter: String,

    position: usize,

    nesting_levels: Vec<usize>,

    errors: Vec<ParseError>,
}

impl Parser {
    pub fn new(md: TokenizedMd) -> Self {
        Self {
            source: md.elements,
            frontmatter: md.frontmatter,
            position: 0,
            nesting_levels: Vec::new(),
            errors: Vec::new(),
        }
    }
}
```

パーサの`nesting_levels`はネストの深さを管理するのに使用しています。`:::details`のネストは、外側の要素の`:`の数が一番多くなるように書く必要があります。
```md
::::::details 1
:::::details 2
::::details 3
深淵
::::
:::::
::::::
```
:::::details 1
::::details 2
:::details 3
深淵
:::
::::
:::::

これをスタックを用いて管理し、下記のような、不正なネストを検出しています。
```
:::details outer
::::::::::details inner
ビルド❌
::::::::::
:::
```

パースの処理です。`parse_block`は`end`で指定されたトークンを見つけるまで解析を続けます。`None`が指定されると、ファイルの終わりまで解析を続けます。初めは`None`を渡して解析を始めます。

```rust
fn parse_block(&mut self, end: Option<TokenType>) -> Vec<Element> {
    let mut elements = Vec::new();

    while let Some(token) = self.peek() {
        if let Some(ref end) = end {
            if token.token_type == *end {
                break;
            }
        }

        let element = match self.parse_element() {
            Ok(element) => element,
            Err(error) => {
                self.errors.push(error);
                break;
            }
        };

        elements.push(element);
    }

    if let Some(end) = end {
        if self.peek().is_none() {
            self.errors.push(ParseError::new(
                ParseErrorType::CouldNotFindEndToken(end),
                    0,
                    0
            ));
        }
    }

    elements
}
```

`:::details`の解析を行います。
「`:::`に加えて何個`:`が書かれているか」を**レベル**と呼んでいます。

```rust
TokenType::DetailsBegin { level, title } => {
    self.nest(level, token.row, token.col)?;
    let body = self.parse_block(Some(TokenType::MessageOrDetailsEnd { level }));
    self.advance();
    self.unnest();
    Element::Details { level, title, body }
}
TokenType::MessageOrDetailsEnd { level: _ } => Element::Text("".to_string()),
```
対応するレベルの`:::`を見つけるまで解析を続けます。ネストに入るとき、出るときには以下の関数を実行します。

```rust
fn nest(&mut self, level: usize, row: usize, col: usize) -> Result<()> {
    if let Some(last) = self.nesting_levels.last() {
        if level >= *last {
            return Err(ParseError::new(
                ParseErrorType::InvalidNestingLevel(level),
                row,
                col,
            ));
        }
    }

    self.nesting_levels.push(level);

    Ok(())
}

fn unnest(&mut self) {
    self.nesting_levels
        .pop()
        .expect("unnest() should be called only when nesting_levels is not empty");
}
```
解析するレベルと現在のレベルを比較して、不正な場合はエラーにします。

また、Front Matterのデジリアライズもパーサで行なっています。
```rust
fn parse_frontmatter(&mut self) -> Result<ZetaFrontmatter> {
    let content = &self.frontmatter;

    serde_yaml::from_str::<ZetaFrontmatter>(content).map_err(|error| {
        let (row, col) = if let Some(location) = error.location() {
            (location.line(), location.column())
        } else {
            (0, 0)
        };

        ParseError::new(ParseErrorType::InvalidFrontMatter, row, col)
    })
}
```
`serde_yaml`クレートを使って、簡単にデジリアライズすることができます。問題がある場合は`location`を参照して行数と列数を取得することができます。

### ビルド

`ParsedMd`を対応するプラットフォームのマークダウンに変換します。 `ZennCompiler`と`QiitaCompiler`によって、プラットフォームごとの記法に変換されます。まずはFront Matterの変換を示します。

#### Front Matterの変換

Zenn形式への変換は、単に対応するフィールドを取り出すだけです。
```rust
let frontmatter = ZennFrontmatter {
    title: frontmatter.title,
    emoji: frontmatter.emoji,
    r#type: frontmatter.r#type,
    topics: frontmatter.topics,
    published: frontmatter.published,
};
```

Qiita形式への変換は、すでにファイルが存在しているかどうかで分岐があります。

```rust
let frontmatter = if let Some(existing_fm) = &self.existing_fm {
    QiitaFrontmatter {
        title: frontmatter.title,
        tags: frontmatter.topics,
        private: existing_fm.private,
        updated_at: existing_fm.updated_at.clone(),
        id: existing_fm.id.clone(),
        organization_url_name: existing_fm.organization_url_name.clone(),
        slide: existing_fm.slide,
        ignorePublish: !frontmatter.published,
    }
} else {
    QiitaFrontmatter {
        title: frontmatter.title,
        tags: frontmatter.topics,
        private: false,
        updated_at: "".to_string(),
        id: None,
        organization_url_name: None,
        slide: false,
        ignorePublish: !frontmatter.published,
    }
};
```
`title`や`topics`などはZeta形式のフィールドと同様です。`private`や`updated_at`については、Qiita形式のファイルがすでに存在するならその値を、なければデフォルトの値を使います。

#### 本文の変換

本文の変換処理を比べます。

Zenn形式への変換です。

```rust
fn compile_element(&mut self, element: Element) -> String {
    match element {
        Element::Text(text) => text,
        Element::Url(url) => format!("\n{}\n", url),
        Element::Macro(macro_info) => self.compile_elements(macro_info.zenn),
        Element::Image { alt, url } => {
            format!("![{}]({})", alt, url)
        }
        Element::InlineFootnote(content) => format!("^[{}]", content),
        Element::Footnote(name) => format!("[^{}]", name),
        Element::Message {
            level,
            msg_type,
            body,
        } => {
            let msg_type = match msg_type {
                MessageType::Info => "",
                MessageType::Warn => "",
                MessageType::Alert => "alert",
            };

            let mut compiler = ZennCompiler {};
            let body = compiler.compile_elements(body);

            format!(
                ":::{0}message {1}{2}:::{0}",
                ":".repeat(level),
                msg_type,
                body
            )
        }
        Element::Details { level, title, body } => {
            let mut compiler = ZennCompiler {};
            let body = compiler.compile_elements(body);
            format!(
                ":::{0}details {1}{2}:::{0}",
                ":".repeat(level),
                title,
                body
            )
        }
    }
}
```

Qiita形式への変換です。

```rust
fn compile_element(&mut self, element: Element) -> String {
    match element {
        Element::Text(text) => text,
        Element::Url(url) => format!("\n{}\n", url),
        Element::Macro(macro_info) => self.compile_elements(macro_info.qiita),
        Element::Image { alt, url } => {
            let url = if url.starts_with("/images") {
                image_path_github(url.as_str())
            } else {
                url
            };
            format!("![{}]({})", alt, url)
        }
        Element::InlineFootnote(content) => {
            let mut i: usize = 1;
            let name = loop {
                let name = format!("zeta.inline.{}", i);
                if !self.inline_footnotes.contains_key(&name) {
                    break name;
                }
                i += 1;
            };

            self.inline_footnotes.insert(name.clone(), content);

            format!("[^{}]", name)
        }
        Element::Footnote(name) => {
            let result = format!("[^{}]", &name);
            self.footnotes.insert(name);
            result
        }
        Element::Message {
            level: _,
            msg_type,
            body,
        } => {
            let msg_type = match msg_type {
                MessageType::Info => "info",
                MessageType::Warn => "warn",
                MessageType::Alert => "alert",
            };

            let mut compiler = QiitaCompiler::new(None);
            let body = compiler.compile_elements(body);

            format!(":::note {}\n{}:::", msg_type, body)
        }
        Element::Details {
            level: _,
            title,
            body,
        } => {
            let mut compiler = QiitaCompiler::new(None);
            let body = compiler.compile_elements(body);
            format!(
                "<details><summary>{}</summary>\n{}</details>\n",
                title, body
            )
        }
    }
}
```

`Macro`では、それぞれのプラットフォームに対応するフィールドを参照します。
`Image`では、Qiitaのみ、写真ファイルへのパスを書き換えます。以下のように、写真のURLをGitHubにリンクするように変更します。
```rust
format!(
    "https://raw.githubusercontent.com/{}/{}{}",
    repository, main_branch, path
)
```
`Message`では、Zennの`:::message`記法とQiitaの`:::note`記法への変換を行います。Zennは「何も指定しない」と「`alert`」の2種類だけなので、`info`と`warn`のどちらを指定しても、「何も指定しない」を使うようにしています。
`Details`では、Zennのみ、レベルに応じて`:`の数を調整しています。


### CLI

Zenn/Qiita CLIのように、コマンドラインから簡単に実行できるようにします。Rustの`clap`クレートを使ってコマンドライン引数を解析します。

```rust
#[derive(Debug, Clone, clap::Parser)]
#[command(version, about)]
struct Cli {
    /// Subcommand
    #[command(subcommand)]
    command: ZetaCommand,
}

#[derive(Debug, Clone, Subcommand)]
enum ZetaCommand {
    /// Initialize Zeta
    Init,
    /// Create new article
    New {
        target: String,
        #[arg(long)]
        only: Option<Platform>,
    },
    /// Build article
    Build { target: String },
    /// Rename article
    Rename { target: String, new_name: String },
    /// Remove article
    Remove { target: String },
}

fn main() {
    let cli = Cli::parse();
    match cli.command {
        ZetaCommand::Init => init(),
        ZetaCommand::New { target, only } => new(&target, &only),
        ZetaCommand::Build { target } => build(&target),
        ZetaCommand::Rename { target, new_name } => rename(&target, &new_name),
        ZetaCommand::Remove { target } => remove(&target),
    }
}
```

サブコマンドや引数に対応する構造を定義するだけで、簡単に解析を行うことができ、すごく便利です。引数を受け取って、対応する関数を呼び出しています。

**`init`コマンドでは以下のことを行います。**
- GitHub Repositoryの指定
- `Zeta.toml`ファイルの作成
- `npm init -y`の実行
- `npm install zenn-cli`の実行
- `npm install @qiita/qiita-cli --save-dev`の実行
- `npx zenn init`の実行
- `npx qiita init`の実行
- `images/`ディレクトリの作成
- `zeta/`ディレクトリの作成
- `git init`の実行
- `.gitignore`ファイルの作成
両方のCLIを初期化し、諸々のファイルを作成しています。

**`new`コマンドでは以下のことを行います。**
- `images/{target}`ディレクトリの作成
- `zeta/{target}.md`ファイルの作成
- `zeta/{target}.md`ファイルへのFront Matterの書き込み
執筆を始められるよう、必要なディレクトリとファイルを作成しています。

**`build`コマンドでは以下のことを行います。**
- `zeta/{target}.md`ファイルの変換
    - Zenn形式の成果物は`articles/{target}.md`に、Qiita形式の成果物は`public/{target}.md`に保存される
    - Front Matterの`only`が指定されている場合は、指定されたプラットフォームのみに変換される
Zeta形式から変換を行うコマンドです。

`rename`コマンドは記事の名前（slag）を変更します。`remove`コマンドは記事を削除します。

## 使い方
1. 記事を管理したい場所で`zeta init`を実行します。
1. リモートリポジトリを指定します。
1. ZennのGitHub連携を行います。
1. Qiitaのアクセストークンをリポジトリに設定します。
1. 記事を書くたびにループ
    1. `zeta new`コマンドを実行して記事を作成します。
    1. 気が済むまでループ
        1. 記事を書きます。
        1. 適宜 `zeta build`を実行して確認します。
    1. コミットしてリモートリポジトリにpushします。

いちいち`zeta build`を実行するのが面倒なので、VSCodeの拡張機能[Run On Save for Visual Studio Code](https://marketplace.visualstudio.com/items?itemName=pucelle.run-on-save)を使って、自動的に`zeta build`を実行すると快適です。
ワークスペースの`.vscode/settings.json`に以下の設定を追記します。
```json
"emeraldwalk.runonsave": {
    "commands": [
        {
            "match": "zeta/.*.md",
            "cmd": "zeta build ${fileBasenameNoExt}"
        }
    ]
}
```

## 完成

これで、**ひとつのファイルに書き込み、自動でプラットフォームに合った形式に変換し、pushするだけで記事を公開できる環境**を作ることができました⚔️　参考になれば幸いです。

早速、Zetaを使って本記事を書いてみました。本記事の（Zenn形式に変換する前の）Zetaファイルは以下に置いてあります。

https://github.com/TyomoGit/zenn-qiita/blob/main/zeta/article-batch-management.md


## 参考文献

主に「Front Matterの変換」にて参考にさせていただきました。大変参考になりました。

https://zenn.dev/ot07/articles/zenn-qiita-article-centralized


Zennのマークダウン記法とCLIについて参考にさせていただきました。

https://zenn.dev/zenn/articles/markdown-guide


https://zenn.dev/zenn/articles/install-zenn-cli


Qiitaのマークダウン記法とCLIについて参考にさせていただきました。

https://qiita.com/Qiita/items/c686397e4a0f4f11683d


https://qiita.com/Qiita/items/666e190490d0af90a92b


自動保存の方法について参考にさせていただきました。

https://qiita.com/taruscript/items/63a829a2f76413db4b2a


https://github.com/emeraldwalk/vscode-runonsave


`clap`のドキュメントなどです。

https://docs.rs/clap/latest/clap/


https://docs.rs/clap/latest/clap/_derive/_tutorial/index.html


`serde`関連のドキュメントです。

https://docs.rs/serde/latest/serde/


https://docs.rs/serde_yaml/latest/serde_yaml/index.html


## 変更履歴
- 2024-03-28: 「完成」にあるGitHubへのリンクを修正、見出しレベルを修正、タグを追加
