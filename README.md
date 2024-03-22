# zenn-qiita

公開の記事を書いたらここに置く。

自作ツールを使用して、一括管理をしている
    --> https://github.com/TyomoGit/zeta

- `zeta init`: Zetaを初期化
- `zeta new 記事のslag`: 記事を作成
    - `zeta`ディレクトリにファイルができる。ここに書いていく。
- `zeta build 記事のslag`: 記事をビルド
    - Zenn用（`articles`ディレクトリ）とQiita用（`public`ディレクトリ）に成果物ができる。
- プッシュで公開する。

Zennの本は`books`ディレクトリで管理する。
`npx zenn new book:name`
