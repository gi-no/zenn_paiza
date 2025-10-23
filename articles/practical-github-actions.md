---
title: "GitHub ActionsでCIを構築するときに考えたいこと"
emoji: "😺"
type: "tech"
topics: ["githubactions", "test", "ci", "rspec", "container"]
published: false
publication_name: "paiza"
---

## モチベーション
paizaでは今までCircleCIを利用して自動テストを実施してきました。  
CIを実施するにあたり機能的には十分ではあったのですが、様々な技術的観点から再考し、GitHub Actions（以下 GHA）への移行を推し進めました。

この移行を通じて**GHAの制約を理解し、うまく回避することでポテンシャルを引き出すことが**重要だとわかりました。

本記事では典型的なハマりどころとそれを回避する構成法を紹介します。

## 1.並列ジョブによるCIの実装

一般にプロダクションレベルのCIでは、変更リリース前に大量の自動テストの実行が必要です。  
そこで最低限必要になるのは**並列ジョブ**によるテスト高速化です。  
GHAではマトリックスという機能があり、複数ジョブに分割することでテストを並列実行できます。

### もっとも単純な実装

もっとも単純な実装は、1つのマトリックスジョブを定義し、テスト実行に必要な(ビルド)処理を含めて並列ジョブとして実施することです。  
この構成でも最低限の要求は達成できます。  
ただしすべての並列ジョブでビルド処理も含めて実施するため、**ランナーコストが肥大化**します。  
この方法は効率が悪いためターゲットとしていなかったですが、この方法が有効な場合もあるので示しておきます。

### キャッシュを使ったビルド + テスト分離

まずはテスト実行に必要なビルド処理と実際のテスト実行処理(rspec)の分離を目指しました。  
 
![](/images/practical-github-actions/simple-workflow.png)
*ワークフローの例：ビルドジョブと並列テストジョブの分離(不要情報は消しています)*

並列テストを高速化するためには、以下のようなものをジョブ間でやり取りする必要があります。

- 言語パッケージ・モジュールなど
  - Rubyなら(bundle) gem
  - Node.jsならnode_modules
- テストDBのスキーマ情報
  - Railsならdb/schema.rbなどのマイグレーション結果
- フロントエンドバンドラの出力
  - Rspack, webpack, Viteなどのトランスパイル結果
- railsのアセットパイプラインの出力
  - RailsのSprockets, Propshaftなど
- その他起動の高速化に役立つキャッシュ
  - RailsならBootsnap cacheなど

例えば以下のように記述すればschema.rbをキャッシュできます。
```yaml
- uses: actions/cache@v4
  with:
    path: ./db/schema.rb
    key: db-schema-${{ hashFiles('db/migrate/**/*') }}-${{ hashFiles('Gemfile.lock') }}
```

私はとりあえず「全部cacheでやりとりすればよいのでは」と安直に考えていました。  
その考えで全てを構成していたある日、突如並列テストが失敗するようになりました。  

GHAの**cacheサイズ上限はリポジトリごとに10GB**です。  
(ただし追加課金により上限緩和が申請可能)  

cacheがこの上限に到達すると、古いビルド結果から削除されます。  
必要なファイルがキャッシュから取得できなくなることでテストジョブが失敗したのです。

**cacheは削除(eviction)される前提でワークフローを構成する**必要があります。

### cacheとartifactの使い分け

cacheと違いartifactはほぼ確実にジョブ間でデータを転送できます。  
一方でartifactはストレージ課金の対象なので、大きなデータをワークフロー起動ごとにartifactでやりとりするとコスト(とデータ転送時間)がかかります。

**artifactの保持期間は短縮でき**、例えば1日で削除するようにすれば安くなりますが、翌日に(flaky testなどの)リトライ実行をしようとすると揮発していることがあります。**3~7日に設定すれば週またぎでの再実行も失敗しないのでよい設定**だと考えます。  
以下は5日の保持期間でschema.rbをアーティファクトとしてアップロードする例です。
```yaml
- uses: actions/upload-artifact@v4
  with:
    name: db-schema
    path: ./db/schema.rb
    include-hidden-files: true
    if-no-files-found: error
    retention-days: 5
```

ここでは言語パッケージ以外のファイルサイズは小さいためartifactに保持し、並列テストジョブの中では一切リビルドしないことにしました。

言語パッケージについては標準の言語セットアップアクションに付随しているキャッシュ機能を基本的に利用します。  
(転送量削減のためカスタムしているものもありますが省略)  
例えば以下のような記述でキャッシュできます。
```shell
- uses: actions/setup-node@v4
  with:
    cache: 'npm'
    cache-dependency-path: package-lock.json
```
この方法だと並列テストジョブ内で追加インストールされる可能性があり、それが不安定性につながる可能性はありますがとりあえずバランスが取れた選択肢だと考えます。

![](/images/practical-github-actions/node-modules-are-heaviest.webp)
*node_modulesサイズの大きさは設計を複雑にします*

もう1つ**知っておくとよいGHAの制約**としてcacheキーのマッチングアルゴリズムがあります。  
[ドキュメント](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching?utm_source=chatgpt.com#cache-key-matching)によれば**現在のブランチとデフォルトブランチで作成したキャッシュだけが利用できる**ということです。  
つまりキャッシュを最大限活かすためには(スケジュール実行などを通じて)**デフォルトブランチで作成し、それをそれぞれの開発ブランチが再利用する**という構成が重要です。  
(開発ブランチでキャッシュを作っても、すぐにマージするならその効果は限定的です。)

以上のことを気をつければ、artifactとcacheを使い分けることができるでしょう。

## 2.実行時間に基づくテスト分割

ところでテストを単純にジョブ分割すると、テスト実行時間にばらつきが生じます。  
テストのリードタイム(起動からテスト結果が確定するまでの時間)は、最も遅いジョブに左右されます。

テストの実行時間を加味してテスト分割できれば、ジョブごとのばらつきを抑えリードタイムを短縮できます。

### もっとも単純な構成

とりあえず余計なことは考えずもっとも単純な構成を考えます。
並列テストジョブの中で実行時間情報をアーティファクトとしてそれぞれアップロードし、次回実行時にすべてダウンロードし、テスト分割します。  
余計なジョブは追加しません。  
cacheと同様、実行時間情報を効率的に再利用することを考えると、デフォルトブランチのアーティファクトとしてアップロードすることが大切です。  
(個々の開発ブランチでいくら実行情報をアップロードしてもほとんど再利用できません。)   
しかし標準ダウンロードアクション(actions/download-artifact)では異なるブランチのダウンロードできません。 (2025年10月現在)    
サードパーティアクションを使うかartifact APIを叩く必要があります。   
以下はサードパーティアクションを利用しデフォルトブランチのアーティファクトをダウンロードする例です。
```
- uses: dawidd6/action-download-artifact@v6
  with:
    name: reports
    path: prev
    branch: ${{ github.event.repository.default_branch }}
    workflow_conclusion: success
    if_no_artifact_found: warn
  continue-on-error: true
```

RSpecでは[split-test](https://github.com/mtsmfm/split-test)というツールと互換性のあるXMLが簡単に出力できます。  
それにより効率的なファイル分割が可能でした。

「これですべてうまく行った」と確信し業務終了した直後、テストジョブが失敗しました。そこにはGitHub APIのレートリミットエラーが示されていました。

この構成の問題点は **M 個の並列ジョブがM個のアーティファクトをそれぞれダウンロードするためM²オーダーのAPIコールが必要**だったのです。

### artifact APIコールの最適化

解決策は次の通りです。上述の通りデフォルトブランチでのみ実施します。

1. 各ジョブがメトリクスをアップロード  
2. 後続の「merge」ジョブがそれらを1つに統合  
3. 統合結果をアーティファクトとして再アップロード

これをフルスクラッチで実装するのは結構大変ですが、幸いなことに公式のmergeアクションを利用すれば簡潔に実装できます。

```yaml
if: github.ref_name == github.event.repository.default_branch
steps:
  - uses: actions/upload-artifact/merge@v4
    with:
      name: reports
      pattern: report-*
      delete-merged: true
```

これによりAPIコールは多く見積もっても3×M回に限定されうまくいきます。

### outputsによる動的マトリックス構成

しかしワークフローがたまに失敗するという不安定さ(flakiness)が生じるようになりました。  
というのも並列ジョブがM回並列でartifact APIをコールし、そのうち1つだけでも失敗するとワークフローとしては失敗になるからです。  
そこでoutputsをうまく活用する方法を検討しました。  
outputsを通じて転送できるデータは1MBまでという厳しい制限がありますが、RSpecのファイルリスト程度なら余裕を持って転送できることに気づいたからです。  
実はマトリックスは以下のような記述をすれば前段のジョブ出力を使って動的に構成することができます。
```yaml
strategy:
  fail-fast: false
  matrix:
    include: ${{ fromJson(needs.plan-test-matrix.outputs.matrix) }}
```

そこでテスト分割処理を集約するジョブを前段に配置し、並列テストジョブにファイルリストを上のやり方で渡します。  
並列テストジョブはただ与えられたファイルリストのテスト実行をすればよくなり、API呼び出ししなくて済みます。  
これによりflakinessが大きく削減されました。

![](/images/practical-github-actions/practical-workflow.png)
*より実践的なワークフロー構成*

## 3.parallel_testsによるインスタンス内並列化

GHAではインスタンスごとの最小vCPU数は2です。(CircleCIは1)  
つまりシングルプロセスでテストを実行するなら、CircleCIのほうがコストを安くできます。  
移行を進めるにあたり、移行前のほうが安くできるというのは判断がややこしくなるため、2vcpuを使い切ることはとても重要でした。

RSpecでは [`parallel_tests`](https://github.com/grosser/parallel_tests) を用いることで簡単にこれを実現できます。
```bash
bundle exec parallel_rspec $PATHS
```
実行時間情報を保存するにあたり、並列テストジョブのインデックスとparallel_testsのインデックスを保持することが実装上のポイントです。
```erb
--format RspecJunitFormatter
--out report-<%= ENV['TEST_TYPE'] %>-<%= ENV['GHA_JOB_INDEX'] %>-<%= ENV['TEST_ENV_NUMBER'] %>.xml
--format documentation
```

これによりGHAのvCPUを無駄なく使い切り、コストとリードタイムの両立を実現できます。

## まとめ
制約を回避しようとするとシステムは複雑化するので、いつでも許される限り単純な構造を選択すべきです。  
しかし今回の記事ではそれではうまくいかないケースを説明し、最小限の(実装)複雑性で回避する方法を示しました。  
とくにジョブ間の媒体になるcache, artifact, outputsを使いこなせば、かなり自在にワークフローを構成できることに気づけます。

ほかにも制約はありますが、以下に整理しておきます。

|  | 制約 | 対策 |
|------|------|------|
| cacheサイズ | [10GB](https://docs.github.com/ja/actions/reference/workflows-and-actions/dependency-caching#usage-limits-and-eviction-policy) | キー設計、アーティファクトとの使い分け |
| cacheアクセス |[ブランチに制限あり](https://docs.github.com/en/actions/reference/workflows-and-actions/dependency-caching?utm_source=chatgpt.com#cache-key-matching)|デフォルトブランチでcacheを作りそれを再利用する
| (artifact) API | [制限あり](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28#primary-rate-limit-for-github_token-in-github-actions) | アップロード/ダウンロードをうまく集約 |
| outputs サイズ | [1MB/ジョブ, 50MB/ワークフロー](https://docs.github.com/en/actions/using-jobs/defining-outputs-for-jobs) | ファイルリストの伝達程度には使える、安定性はある |
| インスタンス | 2vCPUが最小 | parallel_testsでリソースフル活用 |

これらを理解した上で設計すれば、GHAでコスト効率よく堅牢なCIを構築できます。  
またこの記事では示さなかった部分として、コンテナ導入によるパッケージダウンロードに起因するflakiness削減に取り組めたり、サードパーティあるいはセルフホストでのCI実行への道筋も開くことができました。  
はっきり言えば、プロダクションレベルの**CIシステム構築はかなり複雑**であり技量が試されます。  
この記事がいくらか参考になればと思います。  
移行にはかなりエネルギーを使いましたが、納得できる結果が得られたと考えています。  
