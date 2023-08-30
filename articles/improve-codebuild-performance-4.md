---
title: "デプロイ・ビルドパイプラインを倍速にするテクニックその４〜Dockerのscratchイメージの有効活用術〜"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloud","paiza"]
published: true
published_at: 2023-08-31 19:00
publication_name: "paiza"
---

## はじめに

[paiza株式会社](https://www.paiza.co.jp/)で、エンジニアをやっております。yukimuraと申します。

[paiza](https://paiza.jp)のサービス開発では、ユーザーに価値を届けることを重要視しており、継続的デリバリーを行っています。

日に2桁回のデプロイすることも多く、デプロイにかかる時間が長くなると生産性の悪化に直結してしまいます。
paizaでは、EC2ベースのWebアプリからECSベースのWebアプリへと進化を遂げており、結果的にDockerのイメージのビルド時間やECS FARGATEのタスクの切り替え時間が肥大化し、EC2時代に比べてデプロイに時間がかかるようになってしまったため、様々な高速化施策を行い、デプロイ全体としては、50%(およそ2倍)の改善を、ビルド時間だけでみると、61%(およそ3.17倍)の高速化を実現しました。

![DockerStageの整理](/images/improve-codebuild-performance-1/fig1.png)

本記事では、paizaで施したデプロイ・ビルドパイプラインの高速化のテクニックのうち、ビルド時のcacheをS3やEFSに永続化することによる高速化テクニックを紹介いたします。

## 対象読者

- AWS CodePipelineの高速化をしたい方
- Webpackのビルドが遅くて困っている方
- Sprocketsのビルドが遅くて困っている方
- Dockerのscratchイメージの活用方法が気になる方

## Webpack・Sprocketsのbuildが遅い問題

paizaでは、比較的新しいfrontendの実装は、React/Typescriptを使って、Webpack(5系)でbuildするアーキテクチャを採用しています。
また、古いfrontendの実装は、昔RubyOnRailsに標準で組み込まれていたSprocketsを利用して、asset:precompileというコマンドでビルドを実施しています。

どちらも、多くのscript(typescriptやcoffeescript)、scss、画像ファイル群を対象に、ビルドが実行されるため、多くのCPUリソース・メモリリソースを消費します。また、実行時間もかかります。
paizaのビルドの時間も、半分近くはフロントエンドのビルド時間に費やしていました。

## AWS CodeBuildにおけるWebpack・Sprocketsのキャッシュの永続化

WebpackもSprocketsもどちらも、ファイルシステム上にキャッシュファイルを保持する機能があり、CircleCIやGithubActionsなどでビルドする場合は、キャッシュを残しておいて次のビルドで再利用することで、この問題に対応することが多いです。

paizaにおいても、自動テストの際には、キャッシュを保存して次のテスト時に再利用するということをやっていました。

AWS CodeBuildも同じ用に、Webpackのビルド時のキャッシュとSprocketsのビルド時のキャッシュを次のビルド時に再利用する方法を検討しました。

AWS CodeBuildで、この手のキャッシュを永続化する方法としては、

- S3を利用する
- EFSを利用する

といった選択肢が考えられますが、今回は既にCodeBuildにEFSをマウントして利用している状況であったため、EFSを利用することにしました。

## docker buildにおけるキャッシュの再利用のコツ

EFSを利用してcacheの永続化をすることは簡単にできます。
しかしながら、docker buildを実施したときに、Dockerfile内の命令として、WebpackのビルドやSprocketsのビルドを実行しているため、以下のような問題があり、高速化に至るまでにいくつか工夫する必要がありました。

問題１：docker buildを実施するhost側から、キャッシュファイルをCOPYする必要があるが、COPYのオーバーヘッドが大きい
問題２：docker build時に生成されたキャッシュファイルを、EFSに保存する必要があるが、docker buildのoutput機能でtarボールに書き出すと、キャッシュ以外も含まれてしまうのでサイズが肥大化してオーバーヘッドが大きい

### 問題１：COPYのオーバーヘッドの低減

COPYのオーバーヘッドは、主に、ファイル数が多すぎることによるIOの問題だったため、tarに固めて転送する工夫をしました。

```yaml:buildspec.yaml
# 〜中略〜
# 前回ビルドのキャッシュ保存時に、tar化してEFSに置いておいたものを、docker buildを実行する領域にコピー
cp -p ${BUILD_CACHE_EFS_DIRECTORY}/rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar ${CODEBUILD_SRC_DIR}/
```

```Dockerfile:Dockerfile
# 〜中略〜
# キャッシュのCOPYと復元部分のみ抜粋
ARG RAILS_BUILD_CACHE_VERSION
ENV RAILS_BUILD_CACHE_VERSION ${RAILS_BUILD_CACHE_VERSION}
COPY .ruby-version rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar* ./
# /efsに保存しているrails-build-stageのcacheを復元
# NOTE: COPYで、.ruby-version(実在するファイルなら何でもよい)を指定しているのは、rails-build-cache-stage-vX-output.tar.gz が無くても動くようにするためのトリック
COPY .ruby-version rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar* ./
RUN [ -f ./rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar ] \
  && tar -xf ./rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar \
  && rm -f ./rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar \
  || true

RUN bundle exec rails assets:precompile
```

このような形で、host側からキャッシュを転送する際に、tarに固めておいたものをCOPYしてあげて、Dockerfile内で、tarを展開すると転送のオーバーヘッドを抑えることができます。

### 問題２：docker buildした結果のキャッシュを取り出し方を工夫する

`docker build` には、オプションで、`--output` オプションがあり、このオプションを活用すると、docker buildの結果をtarファイルとしてアウトプットすることが可能です。

[docker buildの--output](https://docs.docker.jp/engine/reference/commandline/build.html#id9)

このオプションは、便利なのですが、docker buildした対象のステージが丸ごとtarファイルとして保存することになるため、何も考えないで利用すると、キャッシュ以外の情報も含まれてしまうため、キャッシュだけを保存したい場合には適しません。

このような場合に利用できるのが、scratchという、極めてミニマルなimageです。今回のようにビルドキャッシュを使いたい場合に有効です。
具体的には以下のように利用しました。

```Dockerfile:Dockerfile
#　〜中略〜
RUN bundle exec rails assets:precompile

# NOTE: tmp/cache/assets public/assetsは、ファイル数が多くrails-build-cache-stageにおけるhost側との
# 転送に時間がかかるため、予めtarに固めておく
RUN tar -cf rails-build-stage-tar-for-cache-stage.tar tmp/cache/assets public/assets

# --- rails-build-stageの結果をoutputして、cacheとして次回のbuildに利用するためのステージ
FROM scratch as rails-build-cache-stage

COPY --from=rails-build-stage /usr/src/app/rails-build-stage-tar-for-cache-stage.tar rails-build-stage-tar-for-cache-stage.tar
```

このように`FROM scratch` を利用して、キャッシュファイルをtarに固めてCOPYしておきます。
そして、buildspec.yaml側で、このキャッシュファイルをEFSに保存しておきます。

```yarml:buildspec.yaml
# 〜中略〜
- >-
    docker build --file docker/Dockerfile
    --target rails-build-cache-stage .
    --build-arg RAILS_BUILD_CACHE_VERSION=${RAILS_BUILD_CACHE_VERSION}
    --output type=local,dest=rails-build-cache-stage-tmp-output
- mv rails-build-cache-stage-tmp-output/rails-build-stage-tar-for-cache-stage.tar ${BUILD_CACHE_EFS_DIRECTORY}/rails-build-cache-stage-${RAILS_BUILD_CACHE_VERSION}-output.tar
```

scratchイメージに、必要なキャッシュファイルだけをCOPYしておくことで、`docker build --outout` を実施した際に生成される内容がミニマルになるので、余計なオーバーヘッドは発生しなくなります。

この例では、Sprocketsのキャッシュを保持しておく部分を例示しましたが、webpackのキャッシュも同様の方法で次回にキャッシュを再利用することができます。
また、他にもこういったビルドキャッシュを再利用したいケースは多いと思うので、EFSなりS3に保存したい場合に、いろいろ応用が効くんじゃないかなと思います。

ただし、前回のビルドキャッシュを破棄したいケースや破棄すべきケースはあると思いますので、キャッシュのバージョニングを切り替えられるような変数を用意（今回の例では、RAILS_BUILD_CACHE_VERSION）したり、例えば、yarn.lockファイルやGemfile.lockファイルが変わったらキャッシュを破棄するなどの仕組みも考慮して、適切にキャッシュの生存期間を設計するとよいかと思います。

Docker Layer Cacheと違って、キャッシュがヒットするかしないかの二択ではなく、部分的にキャッシュが利用されるので、ソースコードがちょっとずつしか変化しないような場合には、絶大な効力を発揮しますので、ぜひおためしください。

paizaではさまざまな職種のエンジニアを募集しております。
ミッション・ビジョン・バリューに共感できる方、paizaの取り組みに興味/関心がある方からの応募を心からお待ちしております。

https://paiza.jp/recruiters/9
