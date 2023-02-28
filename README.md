# zenn_paizaリポジトリについて

## これは何？

zenn publicationsを使って、paizaのテックブログを運用するのに利用するリポジトリです。

## どう運用するの？

zenn publicationsでは、考え方として、記事は書いた個人に帰属するという考え方のため、以下のような運用を想定しています

①  記載する内容が、paizaでの業務に深く関わる部分のみ、本リポジトリを使ってください。
②  そうではない場合は、個々人で管理していただく想定のため、本リポジトリを利用する必要はありません。

※ 運用方法については、いつもどおり、チームでブラッシュアップしていきましょう

## 使い方は？

<summary>

### 前提条件

<details>

```bash
# 前提としてpnpmでCLIツールをインストールしてください。
brew install pnpm
```

</details>
</summary>

### blog用記事の作成準備

```bash
# 執筆者がわかるようなディレクトリを作成してください
# 一度だけ実施すればOKです。
pnpm init
pnpm install zenn-cli
```

### blog用記事の作成

```bash
# 記事の雛形を作成しましょう
pnpm exec zenn new:article --slug yukimura-sample --title タイトルをここに書く --type idea # typeはidea or tech
# 上記で作成されたmarkdownを加筆修正することでblogの作成ができます。
```

### blogのプレビュー

```bash
pnpm exec zenn preview --port 8000
```

### 画像の保存先について

画像は、images/${slug}ディレクトリを作成して、そこに格納するようにしましょう。

```bash
# slugがyukimura-sampleの場合
mkdir -p images/yukimura-sample
```

### その他

便利な利用方法等は、zenn公式ホームページをご参照ください。

https://zenn.dev/zenn/articles/connect-to-github