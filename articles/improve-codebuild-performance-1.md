---
title: "デプロイ・ビルドパイプラインを倍速にするテクニックその１"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloud","paiza"]
published: true
published_at: 2023-08-29 12:00
publication_name: "paiza"
---


## はじめに

[paiza株式会社](https://www.paiza.co.jp/)で、エンジニアをやっております。yukimuraと申します。

[paiza](https://paiza.jp)のサービス開発では、ユーザーに価値を届けることを重要視しており、継続的デリバリーを行っています。

日に2桁回のデプロイすることも多く、デプロイにかかる時間が長くなると生産性の悪化に直結してしまいます。
paizaでは、EC2ベースのWebアプリからECSベースのWebアプリへと進化を遂げており、結果的にDockerのイメージのビルド時間やECS FARGATEのタスクの切り替え時間が肥大化し、EC2時代に比べてデプロイに時間がかかるようになってしまったため、様々な高速化施策を行い、デプロイ全体としては、50%(およそ2倍)の改善を、ビルド時間だけでみると、61%(およそ3.17倍)の高速化を実現しました。

![DockerStageの整理](/images/improve-codebuild-performance-1/fig1.png)

本記事では、paizaで施したデプロイ・ビルドパイプラインの高速化のテクニックのうち、マルチステージビルドを使ったビルドの高速化について紹介します。

## 対象読者

- Dockerのbuildの高速化をしたい方
  - 特に、CodeBuildを使う前提での高速化に興味がある方

## Docker BuildKitをつかって、マルチステージビルドにする

既に利用されている方は、「何をいまさら。。。」という内容かもしれませんが、簡単にできて改善効果が大きいので、もしマルチステージビルドを活用していない場合は、今すぐ活用することをおすすめします。

Dockerfileのチューニングだけでできるもので、AWS CodeBuildに限らず活用できます。

### BuildKitを有効化する

[BuildKit](https://docs.docker.jp/develop/develop-images/build_enhancements.html) を利用していない場合は、今すぐ有効化しましょう。

BuildKitを利用することによるメリットは様々ありますが、高速化という観点では、以下のようなメリットが得られます。

- Before：ビルドが直列に実施されるため、時間がかかってしまう
- After： ビルド時に、並列実行可能な部分が、自動的に並列実行してくれるため、高速化が期待できる

やり方は簡単で、docker build時に、DOCKER_BUILDKITの変数を設定するだけです。
※ Docker Engineのバージョン18.09以降にする必要があります。

```sh
# BuildKitを利用したdocker buildの例
DOCKER_BUILDKIT=1 docker build --file Dockerfile .
```

AWS CodeBuildを利用している場合は、buildspec yamlに設定を追加しましょう。

```yaml:buildspec.yaml
version: 0.2
env:
  variables:
    DOCKER_BUILDKIT: '1'
# 〜中略
phases:
  build:
    commands:
      - docker build --file Dockerfile .
      # 以下略
```

### マルチステージビルドにして並列化の恩恵を最大化する

[マルチステージビルド](https://docs.docker.jp/develop/develop-images/multistage-build.html) を利用することで、高速化の観点では以下のメリットがあります。

- 並列実行できる箇所が増えるので、高速化が期待できる
- 最終的に必要なものを精査することで、imageサイズを小さくすることができる
  - ECS FARGATEを利用する場合、ECSのタスクの切り替え時間を短縮することができる
- build時のキャッシュを意識して組むことで、Cacheのヒット率があがり高速化が期待できる

#### マルチステージビルドのポイント

マルチステージビルド自体は、Dockerfile内に、複数FROMを書くだけなので、簡単にできます。 ※ Docker Engineのバージョン17.05以降にする必要があります。

例えば、Gemfile,Gemfile.lockで指定されたライブラリをインストールするステージと、実際に本番稼働するRailsアプリを構築するステージを分離すると以下のようになります。
実用的な例ではないですが、bundle install時に生成されたゴミなどが、Railsアプリ用のステージに入らなくなるメリットに加えて、各ステージが並列で動くので、アプリに必要な資材をコピーしつつ、ライブラリのインストールが動くので高速になります。

```Dockerfile
FROM ruby:3.2.2-slim as gem-resolve-stage
COPY Gemfile
COPY Gmfile.lock
RUN bundle install --path vendor/bundle

FROM ruby:3.2.2-slim as rails-app-stage
COPY 適宜アプリ動作に必要な資材をコピーする
COPY --from=gem-resolve-stage vendor/bundle ./
```

ただ、闇雲にやっても効果は出づらいので、以下のように整理をするとよいかと思います。

- ステージ分割区切りを整理する
  - ざっくりと役割別にステージの分割を考えます
    - ライブラリのインストール、コンパイルやトランスパイルの実行など
- 各ステージのインプットは何かを整理する
  - RUNコマンドなどで必要なものが何かを精査する
    - そのために必要なCOPYするべきものに着目する
- 各ステージのアウトプットは何かを整理する
  - RUNコマンドなどで生成される無駄なものがないかを精査する

以下のように、簡単に図で整理するのもおすすめです。

![DockerStageの整理](/images/improve-codebuild-performance-1/fig2.png)

このような形で、どのステージがどんなインプット/アウトプットになるか整理ができていると、依存関係もシンプルにできるので、並列化の恩恵も大きくなります。

Dockerのbuildで、マルチステージビルドをやっていない方は、これだけで大きく高速化できる可能性があるので、ぜひぜひためしてみてください。

paizaではさまざまな職種のエンジニアを募集しております。
ミッション・ビジョン・バリューに共感できる方、paizaの取り組みに興味/関心がある方からの応募を心からお待ちしております。

https://paiza.jp/recruiters/9
