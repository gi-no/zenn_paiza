---
title: "デプロイ・ビルドパイプラインを倍速にするテクニックその３"
emoji: "💨"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["aws","cloud","paiza"]
published: true
published_at: 2023-08-31 10:00
publication_name: "paiza"
---

## はじめに

[paiza株式会社](https://www.paiza.co.jp/)で、エンジニアをやっております。yukimuraと申します。

[paiza](https://paiza.jp)のサービス開発では、ユーザーに価値を届けることを重要視しており、継続的デリバリーを行っています。

日に2桁回のデプロイすることも多く、デプロイにかかる時間が長くなると生産性の悪化に直結してしまいます。
paizaでは、EC2ベースのWebアプリからECSベースのWebアプリへと進化を遂げており、結果的にDockerのイメージのビルド時間やECS FARGATEのタスクの切り替え時間が肥大化し、EC2時代に比べてデプロイに時間がかかるようになってしまったため、様々な高速化施策を行い、デプロイ全体としては、50%(およそ2倍)の改善を、ビルド時間だけでみると、61%(およそ3.17倍)の高速化を実現しました。

![DockerStageの整理](/images/improve-codebuild-performance-1/fig1.png)

本記事では、paizaで施したデプロイ・ビルドパイプラインの高速化のテクニックのうち、パイプラインの見直しによるAWS CodeBuildの並列実行のによる高速化について紹介します。

## 対象読者

- AWS CodePipelineの高速化をしたい方
- paizaのデプロイの仕組みに興味がある方

## デプロイ/ビルドパイプラインの見直しと並列実行による改善

paizaのサービスのデプロイは、以下の4つの工程がありました。

1. staging環境※向けのビルド
1. staging環境へのデプロイ(ECSのタスクの定義の作成と切り替え)
1. production環境向けのビルド
1. production環境へのデプロイ(ECSのタスクの定義の作成と切り替え)

※ staging環境：productionと同等のインフラ構成で、搭載されているアプリケーションも同等である、動作検証・デモ用の環境

ここで、staging環境向けのビルドとproduction環境向けのビルドは、それぞれdocker imageをbuildしていました。
staging環境用とproduction環境用では、ビルド結果が異なるため、imageは別々である必要があるのですが、ビルド自体は同じタイミングで実施しても実害はないものでした。

そこで、staging環境向けのビルドを実施するのと同時に、production環境向けのビルドを実施するように改善を施しました。

冒頭のグラフで、2023-08では、production環境向けのビルドが0になっているのが改善箇所になります。
これは、staging環境向けのビルド時間に含まれるようになったので、production環境向けのビルド時間はなくなったことによります。（実際、productionのビルドをするためのAWS CodeBuildの設定は破棄しました。）

改善前
![Before](/images/improve-codebuild-performance-3/fig1.png)

改善後
![Afterの整理](/images/improve-codebuild-performance-3/fig2.png)

## おまけ：buildspec.yaml内での並列化

前の章で、CodeBuild自体を並列で起動することで、高速化を行う方法を紹介しました。
並列化という間だと、CodeBuild内の処理もバックグラウンド実行を活用することで並列処理できるため、高速化が可能です。

例えば、nginxのコンテナのビルドとfluentdのコンテナのビルドを例にすると、単純に、nginxのビルドを実施して、fluentdのビルドを実施してという処理をするbuildspec.yamlは、以下のようになります。

```buildspec.yaml
# 〜中略〜
  build:
    commands:
      # nginxとfluentd用のdocker builを並列実行する
      - echo Building the Docker image for nginx ...
      - >-
        docker build --file docker/nginx/Dockerfile.nginx
        --tag ${ECR_ENDPOINT}/${NGINX_REPO_NAME}:${IMAGE_TAG}
        ./docker/nginx/

      - echo Building the Docker image for fluentd...
      - >-
        docker build --file docker/fluentd/Dockerfile.fluentd
        --tag ${ECR_ENDPOINT}/${FLUENTD_REPO_NAME}:${IMAGE_TAG}
        ./docker/fluentd/
```

これを、バックグラウンド実行を活用して、nginxとfluentdを並列に実行すると以下のようになります。

```buildspec.yaml
# 〜中略〜
  build:
    commands:
      # nginxとfluentd用のdocker builを並列実行する
      - echo Building the Docker image for nginx & fluentd...
      - >-
        docker build --file docker/nginx/Dockerfile.nginx
        --tag ${ECR_ENDPOINT}/${NGINX_REPO_NAME}:${IMAGE_TAG}
        ./docker/nginx/
        & echo $! > nginx-image-build-pid;

        docker build --file docker/fluentd/Dockerfile.fluentd
        --tag ${ECR_ENDPOINT}/${FLUENTD_REPO_NAME}:${IMAGE_TAG}
        ./docker/fluentd/
        & echo $! > fluentd-image-build-pid;

        wait $(cat nginx-image-build-pid);
        wait $(cat fluentd-image-build-pid);
```

nginxとfluentd用のbuildを一つのセクションにまとめて、それぞれ ` &` を末尾につけてバックグラウンド実行を行います。
また、バックグラウンド実行のプロセスIDをファイルに保存しておき、バックグランド実行の完了を `wait` で待つことで、何か後続処理が必要な場合でも、完了を待って処理することができます。

あまり乱用するとbuildspec.yamlが複雑になってしまうので、保守性とのバランスを取る必要はありますし、並列処理を増やしすぎるとCodeBuildのインスタンスの性能が低いと時間がかかったりエラーになったりするので、注意が必要ですが、改善テクニックの一つとして、覚えておいて損はありません。

## おわりに

AWS CodeBuildを利用する場合、AWS CodeBuild自体のプロビジョニングに数分かかることもあるので、なるべく並列化をすることで、トータルスループットの改善が見込めます。
ビルド/デプロイパイプラインが複雑な場合は、整理をして無駄がないか？並列実行できることがないか？検討してみると改善点が見つかるかもしれませんので、ぜひおためしください。

paizaではさまざまな職種のエンジニアを募集しております。
ミッション・ビジョン・バリューに共感できる方、paizaの取り組みに興味/関心がある方からの応募を心からお待ちしております。

https://paiza.jp/recruiters/9
