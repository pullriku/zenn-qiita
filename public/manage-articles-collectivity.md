---
title: ZennとQiitaの2重管理を解消してみた
tags:
- Zenn
- Qiita
- Rust
- clap
- serde
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: true
---

## はじめに

QiitaとZennに同じ記事を投稿する際には、同じ内容を2つのファイルで管理する必要があります。この状態では、変更漏れが発生する可能性があります。
実際、私はZenn用とQiita用のリポジトリを作成して二重管理をしていました。すると、やっぱり変更漏れが起きました。

https://x.com/0x0041/status/1770597541568221591


記事投稿後に、ソースコードや説明文をわかりやすくするために、記事を変更しようとしました。しかし、Zennに上げた記事だけを更新して満足してしまいました。

このようなことを防ぐために、Rust言語で簡単なツールを作ってみたのでご紹介します。名前は「Zeta」です。

https://github.com/TyomoGit/zeta


## 方針・仕様

このツールでは、ひとつのファイルに記事を書き、それを変換して他のプラットフォームに対応するという方法を採りました。
「ZennからQiitaへの変換」や「QiitaからZennへの変換」でもよかったのですが、今回はZennライクな独自形式から双方への変換を行うことにしました。「マクロ機能」を使えるようにするための記法を入れたり、Qiitaとの兼ね合いで`:::message`記法を変更したりしたかったからです。

**Zenn CLI**で初期化コマンド`npx zenn init`と、**Qiita CLI**で初期化コマンド`npx qiita init`を実行します。そして、ここにZeta用のディレクトリを加えると以下のような構成になります。

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

`zeta/`内に記事ファイルを作り、そこに書き込みます。`images/`内に画像ファイルを置きます。変換を行うと、Zenn用の記事が`articles/`に、Qiita用の記事が`public/`に生成されます。

### このツールがすること
このツールの「変換」では、以下のことをします。

- URLはカードにするために上下に改行を入れる（Qiitaの仕様）
- マクロを展開する
```
あなたが見ているのは<macro>
zenn: "Zenn"
qiita: "Qiita"
</macro>です。
```
-> あなたが見ているのはZennです。
-> あなたが見ているのはQiitaです。
- 写真のパスを調整する（Zennは`/images`、QiitaはGitHubへのリンクにする）
- インライン脚注を調整する（Zennは`[^名前]`、Qiitaは`[^名前]`
- `:::message`を調整する

| Zetaの形式 | Zennの形式 | Qiitaの形式 |
| :--- | :--- | :--- |
| `:::message info` | `:::message` | `:::note info` |
| `:::message warn` | `:::message` | `:::note warn` |
| `:::message alert` | `:::message alert` | `:::note alert` |

- `:::details [title]`を調整する
Zenn
  ```
  :::details title
  content
  :::
  ```
  Qiita
  ```
  <details><summary>title</summary>

  content
  </details>
  
  ```

マクロは、プラットフォームごとに違う記述をしたいときに使います。`macro`タグの中に、yaml形式でZennとQiita用の文字列を指定します。
```html
- 以前投稿した記事に<macro>
zenn: "Like"
qiita: "いいね"
</macro>を頂きました。嬉しいです。

- 変更履歴は<macro>
zenn: "この記事のGitHubリポジトリ"
qiita: "「編集履歴」"
</macro>から確認できます。
```

実際に展開するとこんな感じです。
👇
- 以前投稿した記事にいいねを頂きました。嬉しいです。

- 変更履歴は「編集履歴」から確認できます。

☝️

マクロを使う機会は少ないと思いますが、「一元管理にしたが故に不自由」という状態をなくすために作りました。

このような変換を行ってくれるツールを作ります。

## 実装

### 構文解析

変換に関係する構造を解析します。それ以外はただの文字列として扱います。
マークダウンファイルは以下のような構造として解析されます。

```rust
#[derive(Debug, Clone)]
pub struct MarkdownFile {
    pub frontmatter: ZetaFrontmatter,
    pub elements: Vec<Element>,
}
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
    Macro(Macro),
    Image {
        alt: String,
        url: String,
    },
    InlineFootnote(String),
    Message {
        msg_type: MessageType,
        body: Vec<Element>,
    },
    Details {
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

`frontmatter`というのは、Zenn CLIやQiita CLIを使って記事を書くときに、ファイルの一番上にyaml形式でタイトルなどを定義できるものです。CLIを使って管理をする場合、Zennでは以下のように記述します。
```txt
---
title: "title"
emoji: "👍"
type: "info"
topics: ["topic1", "topic2"]
published: false
---
```

Zetaではこれに`only`というフィールドを追加したものをfrontmatterとしています。これは、特定のプラットフォームのみに変換するためのオプションです。
`Element`はマークダウンの構造の一部を解析したものです。

### Frontmatterの変換

プラットフォームごとに変換を行います。このことをソースコード内では「compile」と呼んでいます。まずはFrontmatterの変換を見てみましょう。

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

### 本文の変換

変換処理を比べてみます。

Zenn形式への変換です。

```rust
fn compile_element(&mut self, element: Element) -> String {
    match element {
        Element::Text(text) => text,
        Element::Url(url) => format!("\n{}\n", url),
        Element::Macro(macro_info) => macro_info.zenn.unwrap_or_default(),
        Element::Image { alt, url } => {
            format!("![{}]({})", alt, url)
        }
        Element::InlineFootnote(name) => format!("^[{}]", name),
        Element::Message { msg_type, body } => {
            let msg_type = match msg_type {
                MessageType::Info => "",
                MessageType::Warn => "",
                MessageType::Alert => "alert",
            };

            let mut compiler = ZennCompiler {};
            let body = compiler.compile_elements(body);

            format!(":::message {}\n{}:::", msg_type, body)
        }
        Element::Details { title, body } => {
            let mut compiler = ZennCompiler {};
            let body = compiler.compile_elements(body);
            format!(":::details {}\n{}:::", title, body)
        }
    }
}

Qiita形式への変換です。

fn compile_element(&mut self, element: Element) -> String {
    match element {
        Element::Text(text) => text,
        Element::Url(url) => format!("\n{}\n", url),
        Element::Macro(macro_info) => macro_info.qiita.unwrap_or_default(),
        Element::Image { alt, url } => {
            let url = if url.starts_with("/images") {
                image_path_github(url.as_str())
            } else {
                url
            };
            format!("![{}]({})", alt, url)
        }
        Element::InlineFootnote(name) => format!("[^{}]", name),
        Element::Message { msg_type, body } => {
            let msg_type = match msg_type {
                MessageType::Info => "info",
                MessageType::Warn => "warn",
                MessageType::Alert => "alert",
            };

            let mut compiler = QiitaCompiler {
                existing_fm: None,
            };
            let body = compiler.compile_elements(body);

            format!(":::note {}\n{}:::", msg_type, body)
        }
        Element::Details { title, body } => {
            let mut compiler = QiitaCompiler {
                existing_fm: None,
            };
            let body = compiler.compile_elements(body);
            format!(
                "<details><summary>{}</summary>\n\n{}</details>\n",
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

サブコマンドや引数に対応する構造を定義するだけで、簡単に解析を行うことができます。
引数を受け取って、対応する関数を呼び出しています。

`init`コマンドでは以下のことを行います。
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

`new`コマンドでは以下のことを行います。
- `images/{target}`ディレクトリの作成
- `zeta/{target}.md`ファイルの作成
- `zeta/{target}.md`ファイルへのFrontMatterの書き込み
執筆を始められるよう、必要なディレクトリとファイルを作成しています。

`build`コマンドでは以下のことを行います。
- `zeta/{target}.md`ファイルの変換
    - Zenn形式の成果物は`articles/{target}.md`に、Qiita形式の成果物は`public/{target}.md`に保存される
    - Frontmatterの`only`が指定されている場合は、指定されたプラットフォームのみに変換される
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
ワークスペースの`settings.json`に以下の設定を追記します。
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

これで、**ひとつのファイルに書き込み、自動でプラットフォームに合った形式に変換し、pushするだけで記事を公開できる環境**を作ることができました✏️

## 参考文献

主に「Frontmatterの変換」にて参考にさせていただきました。

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


`toml`のドキュメントです。

https://docs.rs/toml/latest/toml/

