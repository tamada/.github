# 再利用可能ワークフロー

[English Version 🇺🇸](README.md)

このディレクトリのワークフローは再利用可能ワークフローです。他のリポジトリはこれらをコピーせず、
`uses:` で呼び出します。

このリポジトリの名前が `.github` なので、参照にはディレクトリ名が2回現れます。
`tamada/.github` がリポジトリ、`.github/workflows/…` がその中のパスです。

```
uses: tamada/.github/.github/workflows/<file>.yaml@<ref>
```

拡張子は `.yml` ではなく `.yaml` で、`@<ref>` は省略できません。どちらを間違えても、
ジョブが1つも始まらないうちに `invalid value workflow reference` で失敗します。

| ワークフロー | 種別 | 目的 |
|--------------|------|------|
| [`build-hugo-and-publish.yaml`](#build-hugo-and-publishyaml) | 再利用可能 | Hugoでサイトをビルドし、アーティファクトとしてGitHub Pagesへデプロイする。 |
| [`build-hugo-and-publish-to-branch.yaml`](#build-hugo-and-publish-to-branchyaml) | 再利用可能 | Hugoでサイトをビルドし、Pagesが配信するブランチへpushする。 |
| [`release-start.yaml`](#release-startyaml) | 再利用可能 | リリースにタグを打ち、ドラフトとして作成する。 |
| [`release-finish.yaml`](#release-finishyaml) | 再利用可能 | `release-start.yaml` で作成したドラフトを公開し、`release: published` を実際に発火させる。 |
| [`release.yaml`](#このリポジトリ自身のワークフロー) | このリポジトリ用 | 上の2つを呼び出して、このリポジトリをリリースする。 |
| [`lint.yaml`](#このリポジトリ自身のワークフロー) | このリポジトリ用 | プルリクエストごとにactionlintを実行する。 |

`release-start` と `release-finish` は1つのフローの前半と後半であり、その間に呼び出し側自身の
ビルドジョブを挟んで一緒に使うことを前提としています。

2種類が1つのディレクトリに同居しているのは、そうするしかないからです。`.github/workflows` の
サブディレクトリはサポートされておらず、そこに置いたファイルはそもそもワークフローとして扱われません。
ただし見分けるのに規約は要りません。再利用可能なものは `on: workflow_call` だけを持ち、それ以外の
トリガーを持ちません。裏を返せば単独で起動することがないので、このリポジトリのActionsタブに現れる
実行は自分用の2つだけです。

## refの選び方

このリポジトリはリリースごとに2つのタグを打ちます。何がリリースされたかを正確に記録する不変の
`v1.2.3` と、それを指すように移動させる `v1` です。したがって `v1` は常に最新のv1.x.yを指し、
`v1.2.3` は常にそのリリース1つだけを、永久に指します。

`@v1` はv1系の修正が入るたびに移動するメジャーバージョンタグです。ほとんどの場合はこれが正解で、
呼び出し側を書き換えなくても修正が届き、破壊的変更は `v2` として現れるため、こちらから要求しない
限り届きません。

`@main` は未リリースの変更も含め、最後にマージされたものをそのまま追いかけます。試すには便利ですが、
運用に載せるワークフローには向きません。

`@<full-sha>` は完全に固定します。勝手に動くことがない代わりに、修正も勝手には届きません。
アップグレードは呼び出し側の書き換えで行うことになります。

## 呼び出し側が用意するもの

**権限。** 呼ばれたワークフローは、呼び出し側が許した以上の権限を自分に与えることはできません。
権限は連鎖の中で維持されるか縮小されるだけで、決して昇格しません。つまり `permissions:` ブロックは
呼び出す側のジョブに書くもので、書き忘れると「何を要求し何が許されているか」を示すメッセージとともに
失敗します。必要な権限は各ワークフローの項に記載しています。

**シークレット。** シークレットは暗黙には渡りません。`secrets:` の下で1つずつマッピングするか、
`secrets: inherit` と書いて呼び出し側が持つものをすべて渡します。シークレットを受け取るのは
`release-finish` だけです。

**環境変数は越えてきません。** 呼び出し側のワークフローレベルで設定した `env` は、呼ばれた
ワークフローには伝播しません。ワークフローが必要とするものは入力として渡す必要があります。

**アクセス。** これらはパブリックリポジトリにあるため、どのリポジトリからでも呼び出せます。ただし
呼び出す側の組織が「パブリックな再利用可能ワークフローの利用」を許可している必要があります。

制限も2つ挙げておきます。通常の使い方でどちらにも近づくことはありませんが、連鎖は最大10段、
1つのワークフローファイルから呼べる異なる再利用可能ワークフローは最大50個です。

## Hugoサイトを公開する

これには2つのワークフローがあり、GitHubがPagesサイトを配信する2つの方式それぞれに対応します。
ビルド部分は同一で、ビルド結果の行き先だけが違います。したがって選択は「どちらのファイルを
呼ぶか」になります。

| 呼ぶファイル | サイトの行き先 | 必要なリポジトリ設定 |
|--------------|----------------|----------------------|
| [`build-hugo-and-publish.yaml`](#build-hugo-and-publishyaml) | Pagesアーティファクトとしてアップロードされデプロイされる。コミットは発生しない。 | *Settings → Pages → Source* = **GitHub Actions** |
| [`build-hugo-and-publish-to-branch.yaml`](#build-hugo-and-publish-to-branchyaml) | ブランチへpushされ、Pagesがそれを配信する。 | *Settings → Pages → Source* = **Deploy from a branch**（対象ブランチを指定） |

**リポジトリ設定は、呼ぶファイルと一致していなければなりません。** この2つは同じスイッチを両端から
見たもので、食い違ったときの現れ方が最悪です。実行は緑のまま終わり、公開されているサイトだけが
変わりません。

理由がなければアーティファクト方式を選んでください。ブランチ方式が活きるのは、生成されたサイトを
gitの履歴に残して確認したり差分を見たりしたい場合、別のリポジトリへ公開する場合、あるいはActionsを
ソースにできない場合です。

必要な権限も異なります。ここも間違えやすい点です。

| ワークフロー | 与える権限 |
|--------------|------------|
| `build-hugo-and-publish.yaml` | `contents: read`、`pages: write`、`id-token: write` |
| `build-hugo-and-publish-to-branch.yaml` | `contents: write` |

以下は、断りのない限り両方に当てはまります。

**`baseURL` は利用者の責任です。** ビルドは `hugo --minify` を実行するだけなので、生成される
リンクに入るのはHugo設定ファイルの `baseURL` です。Pagesが実際にサイトを配信する場所と食い違って
いると、ページ自体は表示されるのにスタイルシートや内部リンクだけが別の場所を指します。設定ミスと
いうよりテーマが壊れているように見える種類の失敗なので、ここを疑ってください。

**サブモジュールは再帰的にチェックアウトします。** Hugoのテーマはサブモジュールとして取り込まれる
ことが多いためです。取得されていないサブモジュールは「レイアウトが見つからない」というエラーとして
現れ、まったく見当違いの場所を探すことになります。

**どちらにも出力はありません。** アーティファクト方式はデプロイ先URLを実行結果の `github-pages`
デプロイメントに記録します。ブランチ方式では、公開ブランチ自体が記録になります。

## `build-hugo-and-publish.yaml`

Hugo（extended版）でサイトをビルドし、Pagesアーティファクトとしてアップロードしてデプロイします。
リポジトリには何もコミットされません。

```yaml
name: publish

on:
  push:
    branches: [main]

jobs:
  publish:
    uses: tamada/.github/.github/workflows/build-hugo-and-publish.yaml@v1
    with:
      workdir: docs
      # branch: main
      # hugo-version: latest
    permissions:
      contents: read
      pages: write
      id-token: write
```

### 入力

| 入力 | デフォルト | 意味 |
|------|------------|------|
| `workdir` | `.` | Hugoサイトのあるディレクトリ。成果物は `<workdir>/public` から取得されます。 |
| `branch` | *(空)* | ビルドするgit ref。空の場合は呼び出し側の実行を起動したrefになり、通常のpushトリガーではこれが期待どおりです。 |
| `hugo-version` | `latest` | Hugoのバージョン。常にextended版です。 |

### 補足

**デプロイは `pages` という並行実行グループで直列化され、途中でキャンセルされることはありません。**
GitHubは2つ目の同時デプロイをキューに入れず拒否しますし、実行中のものをキャンセルすると、そこまで
デプロイされた中途半端な状態のままサイトが残るためです。

## `build-hugo-and-publish-to-branch.yaml`

同じサイトを同じ手順でビルドし、そのあとブランチへpushします。Pagesのソースが Actions ではなく
ブランチのときに配信されるのが、このブランチです。

```yaml
name: publish

on:
  push:
    branches: [main]

jobs:
  publish:
    uses: tamada/.github/.github/workflows/build-hugo-and-publish-to-branch.yaml@v1
    with:
      workdir: docs
      # branch: main
      # hugo-version: latest
      # publish-branch: gh-pages
    permissions:
      contents: write
```

### 入力

| 入力 | デフォルト | 意味 |
|------|------------|------|
| `workdir` | `.` | Hugoサイトのあるディレクトリ。サイトは `<workdir>/public` から取得されます。 |
| `branch` | *(空)* | ビルド**元**のgit ref。空の場合は呼び出し側の実行を起動したrefになります。 |
| `hugo-version` | `latest` | Hugoのバージョン。常にextended版です。 |
| `publish-branch` | `gh-pages` | ビルドしたサイトをpushする**先**のブランチ。 |

`branch` と `publish-branch` はそれぞれ入力元と出力先です。前者は読み、後者は書きます。両方を持つ
のはこのワークフローだけなので、名前は二度読む価値があります。

### 補足

**pushは `publish-to-branch` という並行実行グループで直列化され、途中でキャンセルされることは
ありません。** 2つの実行が同時に公開ブランチへpushすると、片方がrebaseするか失われることになり、
中途半端にpushされたサイトは、遅れて反映されるサイトより悪いためです。

## `release-start.yaml`

リリースにタグを打ち、*ドラフト*として作成します。

意図的に言語非依存です。行うのはタグ付けとドラフト作成だけで、バイナリのビルドやレジストリへの公開
といったプロジェクト固有の処理は、このワークフローと `release-finish` の間で動く呼び出し側自身の
ジョブに属します。

ドラフトにするのも意図的です。`GITHUB_TOKEN` が公開状態で作成したリリースは `release: published`
を発火させないため、下流の誰も気づきません。ドラフトを外すのは、イベントを発火させられるトークンを
使う `release-finish` の役目です。

### リリースの切り方

1. タグを打つブランチから `releases/v0.0.5` の形でブランチを作ります。プレフィックスの後ろが
   `v` 付きのバージョンです。
2. バージョン番号の更新や変更履歴など、そのリリースに必要な作業をそのブランチで行います。
3. `main` へのプルリクエストを作成し、マージします。

マージがトリガーです。関わるブランチは2つあり、それらは同じものではありません。

```
releases/v0.0.5  ──── プルリクエスト ────▶  main
      │                                       │
バージョンを持つ側                        タグが打たれる側
```

バージョンはマージされたプルリクエストのheadブランチから読み取られ、タグはそのマージ先ブランチに
打たれます。

```yaml
name: release

on:
  pull_request:
    types: [closed]

jobs:
  start:
    if: startsWith(github.head_ref, 'release') && github.event.pull_request.merged == true
    uses: tamada/.github/.github/workflows/release-start.yaml@v1
    permissions:
      contents: write
```

`if` の条件は両方とも重要です。`types: [closed]` はマージされた場合だけでなく単に閉じられた場合
にも発火するため、`merged == true` がないと、放棄されたリリースブランチがタグを切ってしまいます。
`startsWith` のほうは、失敗しかしないワークフローに無関係なプルリクエストが届くのを防ぎます。

### 入力

| 入力 | デフォルト | 意味 |
|------|------------|------|
| `version` | *(空)* | リリースするバージョン。先頭の `v` は付いていても付いていなくても受け付けます。空ならheadブランチから読み取ります。リリースブランチ起因でない実行では明示的に渡してください。 |
| `version-branch-prefixes` | `releases/v release/v` | 空白区切りのプレフィックス。最初に一致したものの後ろがバージョンになります。 |
| `tag-branch` | `main` | チェックアウトしてタグを打つブランチ。 |
| `appname` | *(リポジトリ名)* | 成果物の命名に使うアプリケーション名。バイナリ名がリポジトリ名と同じなら常に正しい値です。 |

### 出力

| 出力 | 例 | 意味 |
|------|----|------|
| `version` | `0.0.5` | 先頭の `v` が付かないバージョン。 |
| `tag` | `v0.0.5` | 作成されたgitタグ。 |
| `appname` | `myapp` | 解決されたアプリケーション名。 |

### 補足

**`version` は `0.0.5`、`tag` は `v0.0.5` で、どちらの名前も相手の意味になることはありません。**
バイナリやパッケージ定義、ファイル名に埋め込む値には `version` を、gitや `release-finish` に
渡す値には `tag` を使ってください。途中で `v` を付けたり外したりする必要はどこにもありません。
なお `version` **入力**だけは出力より緩く、`0.0.5` でも `v0.0.5` でも受け付けます。素直に書いた
バージョンを拒む理由がないからです。

**どのプレフィックスにも一致しないブランチは、推測せずに実行を失敗させます。** 代わりに最後の `v`
までを削る実装にすると、`releases/v1.0-preview` を `iew` と読み、何にも一致しなかったときには
ブランチ名全体を返してしまいます。どちらも文句を言わずに間違ったものにタグを打つ挙動です。
プレフィックスを列挙しておけば、規約に従わないブランチはそこで明確に止まります。

**プレフィックス一致だけでは足りません。** それらしい名前のブランチが、バージョンでないものを
持っていることはあるからです。`releases/vNEXT` はプレフィックスの条件を満たし、そのままなら
`vNEXT` というタグになります。`releases/v0.0.5/hotfix` はスラッシュ入りのタグになります。
そこでバージョン自体も検証します。先頭が数字で、数字・英字・ドット・`+`・`-` だけで構成されて
いることが条件です。`1.0-preview` や `1.2.3+build5` は通り、先の2つは通りません。

**`github.head_ref` はプルリクエストイベントでしか設定されません。** それ以外の方法で起動した
場合はブランチを読み取れないため、`tag` 入力が必須になります。

**リリースノートは**前のタグ以降のコミットからGitHubが自動生成します。

## `release-finish.yaml`

`release-start` が作成したドラフトを外して公開します。

これが独立したワークフローになっている理由は1つだけです。どのトークンがリリースを公開するかが、
下流がそのリリースを知れるかどうかを決めるからです。

> `workflow_dispatch` と `repository_dispatch` を除き、`GITHUB_TOKEN` が起こしたイベントは
> ワークフロー実行をまったく作成しません。

つまり `GITHUB_TOKEN` で公開されたリリースは `release: published` を発火させず、そのイベントを
待っているワークフローは何も残さないまま沈黙します。失敗した実行もログもなく、ただ何も起きません。
代わりにGitHub Appのトークンで公開すれば、イベントは他と同じように振る舞います。

App の資格情報がない場合も、リリースは `GITHUB_TOKEN` を使って公開されます。そのうえで、下流の
ワークフローは動かないという警告がジョブのログに明示されます。`release: published` を待つものが
何もないのであれば、この使い方でまったく問題ありません。

```yaml
jobs:
  finish:
    needs: [start, build]
    uses: tamada/.github/.github/workflows/release-finish.yaml@v1
    with:
      tag: ${{ needs.start.outputs.tag }}
    permissions:
      contents: write
    secrets: inherit
```

### 入力

| 入力 | 必須 | 意味 |
|------|------|------|
| `tag` | はい | 公開するリリースのgitタグ。先頭に `v` が付いた形（`v0.0.5`）です。`release-start` の `tag` 出力そのものなので、そのまま渡せます。 |

### シークレット

| シークレット | 必須 | 意味 |
|--------------|------|------|
| `APP_CLIENT_ID` | いいえ | リリースを編集できるGitHub AppのClient ID。 |
| `APP_PRIVATE_KEY` | いいえ | そのAppの秘密鍵。 |

名前にアンダースコアを使っているのは、リポジトリシークレットの名前に使えるのが英字・数字・
アンダースコアだけだからです。ハイフン区切りのほうが見た目は良いのですが、`secrets: inherit`
は名前の一致で渡すため、その名前のリポジトリシークレットは作れず、一度も届きません。

### GitHub Appの設定

`release: published` を待つものがある場合にだけ必要です。

1. 自分のアカウントでGitHub Appを作成します（*Settings → Developer settings → GitHub Apps
   → New GitHub App*）。webhookもcallback URLも不要です。
2. リポジトリ権限を1つだけ与えます。**Contents: Read and write** です。リリースはContentsの
   配下にあります。
3. 秘密鍵を生成し、Client IDを控えます。
4. Appをアカウントにインストールし、リリースを公開するリポジトリへのアクセスを許可します。
   `owner` には呼び出し側リポジトリのオーナーが設定されるため、トークンはそのオーナー配下で
   Appがインストールされたリポジトリを対象とします。
5. 2つの値を `APP_CLIENT_ID` と `APP_PRIVATE_KEY` という名前でリポジトリまたはOrganizationの
   シークレットとして登録します。

トークンは実行ごとに発行され、ジョブの終了時に破棄されます。

## リリースワークフローを組み合わせる

```yaml
name: release

on:
  pull_request:
    types: [closed]

jobs:
  start:
    if: startsWith(github.head_ref, 'release') && github.event.pull_request.merged == true
    uses: tamada/.github/.github/workflows/release-start.yaml@v1
    permissions:
      contents: write

  build:
    needs: start
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      # ... needs.start.outputs.version（0.0.5）、needs.start.outputs.tag（v0.0.5）、
      # needs.start.outputs.appname を使ってビルドし、成果物をアップロードする

  finish:
    needs: [start, build]
    uses: tamada/.github/.github/workflows/release-finish.yaml@v1
    with:
      tag: ${{ needs.start.outputs.tag }}
    permissions:
      contents: write
    secrets: inherit
```

`build` はリリースがまだドラフトの間に動きます。これが分割の狙いで、リリースの存在が誰か——あるいは
何か——に伝わる前に、成果物を添付し終えることができます。

## このリポジトリ自身のワークフロー

`release.yaml` と `lint.yaml` は呼び出す対象ではありません。このリポジトリ自身をリリースし、
テストするためのものです。読む価値があるとすれば、ここまでの内容をひととおり実践した例として
でしょう。

### `release.yaml`

このリポジトリを、READMEが説明しているのと同じ手順でリリースします。`releases/vX.Y.Z` を切り、
`main` へプルリクエストを出し、マージする、という流れです。

再利用可能ワークフローの呼び出しは `uses: ./.github/workflows/release-start.yaml` の形です。
同一リポジトリへのローカル参照には `@ref` を付けられず、**呼び出し元と同じコミットのもの**が
使われます。つまりリリースは、タグ付けされた時点のワークフローではなく `main` にある現在の
ワークフローで実行されます。`release-start` を壊す変更は、まずこのリポジトリ自身のリリースを
壊します。タグが打たれる前、そして `@v1` を使っている人に届く前にです。

`start` と `finish` の間、通常のプロジェクトなら成果物をビルドする位置には、ワークフローを配布
するリポジトリにだけ必要でそれ以外にはまず不要な、`v1` を今回のリリースへ移動させるジョブが
入っています。プレリリースの場合はスキップします。`@v1` は、リリース候補を望んでいない人にまで
配るためのものではないからです。ビルドメタデータ（`1.2.3+build`）は正式リリース扱いで、
プレリリースを示すのはハイフンだけです。

### `lint.yaml`

プルリクエストごとにactionlintを実行します。ここにあるファイルはすべてワークフローなので、これが
テストスイートのすべてです。そして再利用可能ワークフローの間違いは、他人のリポジトリの間違いに
なります。

`-ignore` に2つのパターンを渡しています。これらは指摘ではなく、actionlintが同梱している
`actions/create-github-app-token` の入力定義が古いというだけのものです。このアクションはv2から
`client-id` を受け付けており、現在は `app-id` のほうが非推奨と表示されます。どちらのパターンにも
このアクション名が含まれているので、他のアクションで本物の問題があればジョブは失敗します。

実行にはバージョンを固定した `rhysd/actionlint` イメージを使い、上流が案内している
`curl | bash` 形式のインストーラは使っていません。このリポジトリが配布するものは他人のリポジトリで
動くことになるので、移動するブランチから取得したスクリプトを実行し始める場所としては不適切です。

## トラブルシューティング

| 症状 | 原因 |
|------|------|
| ジョブが始まる前に `invalid value workflow reference` | `.yaml` ではなく `.yml` になっている、`@<ref>` がない、またはリポジトリのパスを2回ではなく1回しか書いていない。 |
| `The workflow is requesting 'pages: write', but is only allowed 'pages: none'` | 呼び出し側のジョブに `permissions:` ブロックがない。呼ばれたワークフローは自分の権限を昇格できません。 |
| `head branch '…' starts with none of: …` | リリースブランチが `releases/v` でも `release/v` でも始まっていない。ブランチ名を変えるか、`version-branch-prefixes` を追加するか、`version` を明示的に渡してください。 |
| `version '…' does not start with a digit` | ブランチ名はリリースブランチの形をしているが、バージョンではないものを持っている（`releases/vNEXT` など）。 |
| `version '…' contains characters that do not belong in a version` | 多くはスラッシュ。`releases/v0.0.5/hotfix` はそのままだと `/` 入りのタグになってしまう。 |
| リリースは公開されるのに、`release: published` を待つワークフローが動かない | Appの資格情報がなく `GITHUB_TOKEN` で公開されている。実行ログの警告を確認し、[GitHub Appの設定](#github-appの設定)を参照してください。 |
| 既存のタグがあってタグ作成に失敗する | そのバージョンはすでにリリース済みです。タグは記録なので、動かすのではなくバージョンを上げてください。 |
| Pagesを一度も公開していないリポジトリでデプロイが失敗する | PagesのSourceがGitHub Actionsになっていないのに `build-hugo-and-publish.yaml` を呼んでいる。 |
| 実行は緑なのに公開されているサイトが変わらない | 呼んでいるワークフローとリポジトリのPages Sourceが食い違っている。ブランチへpushしているのにPagesがActionsのアーティファクトを配信している、またはその逆。 |
| デプロイはされるがスタイルとリンクが壊れる | Hugo設定の `baseURL` が、Pagesの実際の配信先と一致していない。 |
| Hugoがレイアウト不足で失敗する | テーマのサブモジュールが取得されていない。空ディレクトリではなくサブモジュールとしてコミットされているか確認してください。 |
