---
title: "デプロイ・ビルドパイプラインを倍速にするテクニックその２"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloud","paiza"]
published: true
published_at: 2023-08-30 12:00
publication_name: "paiza"
---

## はじめに

[paiza株式会社](https://www.paiza.co.jp/)で、エンジニアをやっております。yukimuraと申します。

[paiza](https://paiza.jp)のサービス開発では、ユーザーに価値を届けることを重要視しており、継続的デリバリーを行っています。

日に2桁回のデプロイすることも多く、デプロイにかかる時間が長くなると生産性の悪化に直結してしまいます。
paizaでは、EC2ベースのWebアプリからECSベースのWebアプリへと進化を遂げており、結果的にDockerのイメージのビルド時間やECS FARGATEのタスクの切り替え時間が肥大化し、EC2時代に比べてデプロイに時間がかかるようになってしまったため、様々な高速化施策を行い、デプロイ全体としては、50%(およそ2倍)の改善を、ビルド時間だけでみると、61%(およそ3.17倍)の高速化を実現しました。

![DockerStageの整理](/images/improve-codebuild-performance-1/fig1.png)

本記事では、paizaで施したデプロイ・ビルドパイプラインの高速化のテクニックのうち、dockerのインラインキャッシュとECRへのキャッシュ永続化をつかったビルドの高速化について紹介します。

## 対象読者

- Dockerのbuildの高速化をしたい方
  - 特に、CodeBuildを使う前提での高速化に興味がある方

## 悲報： docker buildx buildはAWS ECRを利用しているとエラーになってしまった

docker buildxを利用すると、通常のdockerでは実現できない、キャッシュの永続化を行うことができます。

https://docs.docker.jp/engine/reference/commandline/buildx_build.html#id43

とても魅力的な機能なのですが、AWS CodeBuildのubuntu(standard7.0)でbuildx buildでキャッシュとともに、イメージのビルドを行い、それをECRにpushしたところエラーになり、断念しました。

## buildxを使わずインラインキャッシュを利用する

docker buildのインラインキャッシュの機能を活用すると、docker imageに、dockerのlayer cacheに関するメタ情報を含めるようにすることができます。
インラインキャッシュを使うには、以下のように、BUILDKIT_INLINE_CACHEをbuild-argとして渡すことで実現できます。

```Dockerfile
# REPOSITORY_NAME,TAG_NAMEは適宜設定してください
DOCKER_BUILDKIT=1 \
  docker build \
  --tag ${REPOSITORY_NAME}:${TAG_NAME}
  --file Dockerfile . \
  --build-arg BUILDKIT_INLINE_CACHE=1
```

ただし、これだけではメタ情報が含まれるようになっただけで、ビルドの高速化には繋がりません。
このメタ情報を有効活用するには、BUILDKIT_INLINE_CACHEを付与したimageをpushしておき、次回以降のビルド時に--cache-fromで参照することで、このメタ情報を使って、既に構築済みのDocker Layerは再構築されずにCacheが利用されることで高速になります。
また、メターデータ自体の容量は、Docker Image全体からすると微々たるものなので、この分で遅くなることはまずないです。
※ ただし、cache-fromで指定したimageのpullが走るので、その分のオーバーヘッドがかかるので、実際に高速化できているかどうかは、確認が必要です。

具体的には、以下のようになります。

```sh
# push時は、特に気にすることはありません。
# REPOSITORY_ENDPOINT,REPOSITORY_NAMEは適宜設定してください
docker push --all-tags ${REPOSITORY_ENDPOINT}/${REPOSITORY_NAME}
```

```sh
# build時に、--cache-fromを指定することで、build時に埋め込まれたメタデータを参照して、変更がなければキャシュが活用されます。
DOCKER_BUILDKIT=1 \
  docker build \
  --tag ${REPOSITORY_NAME}:${TAG_NAME}
  --file Dockerfile . \
  --build-arg BUILDKIT_INLINE_CACHE=1
  --cache-from ${REPOSITORY_ENDPOINT}/${REPOSITORY_NAME}:${TAG_NAME}
```

例えば、同じDockerfileに対して、docker buildを二回実行すると、二回目はCacheが効くので極めて高速にbuildが完了します。
これは、ローカルのストレージにCacheが残っているためです。
paizaでは、AWS CodeBuildを利用しているのですが、AWS CodeBuildの場合は、同じインスタンスが利用されるわけではないため、ローカルのストレージにCacheが残っていることはほとんどありません。
そこで、今回紹介したように、ECRなどのDockerのコンテナイメージを保存するレジストリに、キャッシュ用のメタ情報を含めておき、それをキャッシュとして利用するように--cache-fromで指定することで、前回buildから変更がない部分については、キャッシュが効いて高速にbuildができます。

### Dockerfileを工夫してキャッシュのヒット率を上げる

Dockerfileで、COPYなどをする際に、COPYした対象に変更があったら、それ以降はCacheが効かないため、変更がないものをなるべく前半に、変更が頻繁にあるものをなるべく後半に配置すると良いです。

```Dockerfile
# よくない例(頻繁に更新されるファイルが前半にあるのはキャッシュヒット率を下げる要因になる)
COPY 頻繁に更新される軽いファイル ./ # ここが更新されていると。。。
COPY 頻繁に更新される重いファイル ./ # 以下の行はすべてキャッシュが効かない
COPY めったにに更新されない重いファイル ./
COPY めったにに更新されない軽いファイル ./
```

```Dockerfile
# よい例(めったに更新されないファイルを前半にもってくることで、キャッシュのヒット率が上がる)
COPY めったにに更新されない重いファイル ./ # 更新されない限りキャッシュが効く
COPY めったにに更新されない軽いファイル ./ # めったに更新されない重いファイルと軽いファイルが更新されない限りキャッシュが効く
COPY 頻繁に更新される重いファイル ./ # ここが更新されても、キャッシュが効かないのは、以下の行だけ
COPY 頻繁に更新される軽いファイル ./
```

例えば、マルチステージビルドを使って、gem(Rubyのライブラリ群)をインストールするようなケースや、node_modules(NodeJSのライブラリ群)をインストールするような場合など、頻繁にライブラリの追加や更新がないものに対しては、大きな改善効果が見込めますので、ぜひお試しください。

paizaではさまざまな職種のエンジニアを募集しております。
ミッション・ビジョン・バリューに共感できる方、paizaの取り組みに興味/関心がある方からの応募を心からお待ちしております。

https://paiza.jp/recruiters/9
