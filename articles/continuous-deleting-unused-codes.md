---
title: "不要コードを継続的に削除し、技術的負債に対抗する"
emoji: "🧹"
type: "tech"
topics: ["rails", "ruby", "coverage", "technicaldebt", "ci"]
published: true
published_at: 2023-08-21 07:00
publication_name: "paiza"
---

paizaでWebエンジニアをやっています藤田と申します。  
「今期は技術記事6本書きます」と自ら目標にしておいて、4か月ぐらい記事を滞納していた良心の呵責に耐えかねて投稿いたします。  
今回の記事では、不要コードの削除に関するモチベーションをあらためて整理するとともに、[以前noteで私が執筆した記事](https://note.com/paiza/n/n84a659795d95)の続編として実運用について説明します。

## CI/CDであまり語られない課題 -不要コード-
プロダクション開発を進めるにあたり、自動テスト(ユニットテストやブラウザテスト)を書くとか、CIやlintingはほぼ常識化していると考えます。  
自動テストを書けば、ある挙動が維持されていることを保証できるとともに、コードパスの検証状況(カバレッジ)を可視化できます。  

実装とテストがどんどん増えていき、正常系と異常系の動作は十分に確認できたとします。  
一方時の試練に耐えられず、不要となったコードはどうなるのでしょうか。  
テストは保証したい挙動が維持されていることは示せても、その機能がすでに利用されていないことについて何も語りません。  
したがって一般的なCI/CDが整備された状況であったとしてもプロダクションコードの中に不要コードは山積していきます。

## 不要コードはなぜ生じ、なぜ消されないのか
エンジニアは開発段階において不要コードを作りたいと思って当然開発するわけではないでしょう。  
開発対象の仕様が明確に決まっているとは限らず、よい仕様をむしろ発見するための仮説検証の意図を含んだコードというものは、サービスの初期段階ほど必要になります。  
そしてそのようなコードが、時の試練を経ても必要であり続けるコードと、不要コードに分かれていきます。  
それが(とくにスタートアップにおける)自然状態であると考えられます。    
実際のところ必要であり続けるコードに含まれた技術的負債をなんとかすることは難しく価値があるわけですが、不要コードはそれに取り巻く形で複雑性を増加させる要因になります。  
**なぜ不要コードは消されないのでしょうか。**  
それはさまざまな理由があるわけですが、以下のような理由は意識されなければなりません。
- (ビジネス的には)消す動機がない
  - 別に消さなくても、ユーザからみた振る舞いは変わらない
    - 外部品質は同じ
    - つまりコードを消すプロジェクトが立ち上がることなんてない
- (エンジニアリング的には)消した結果、本番で何かが動作しなくなる可能性がある(リスクテイク)
  - そのコードが本番環境で参照されていないことを保証できるだろうか
    - 動的型付け言語ではとくに難しい
    - 静的片付け言語でもコンパイル単位によっては(たとえばライブラリとしてビルドされていると)難しい
      - それでも圧倒的に動的型付けより形式的に削除の安全性を保証できる部分が多い
    - 書いた本人が消すことで、一番効率的かつリスク低く消せる
      - 特定の人に紐付いたタスクアサインをスクラムに織り込めるだろうか
      - 書いた人が退職していたらどうすればいいのだろうか
      - 書いていない人が消す判断をするのは大変だ
- 削除には合意形成がいるケースも多い
  - 新規開発にはそれほど大掛かりな合意形成はいらない
  - 機能を削除するには、どこにユースケースがないかを洗い出さないといけない
    - 誰の合意を得たら、これを削除してよいのだろうか

以上を要約すると以下のようになります。
- コストがかかる
- 削除することのリスク(不確実性)もある
- ビジネス上のリターンがあるわけではない

つまり**目に見える結果**だけを考えると、コードは消さないほうがあらゆる観点からよいです。
したがって何も力学を働かせなければそのまま不要コードに囲まれた開発ライフを過ごすことになります。

## 不要コードはなぜ削除されるべきか
以上のことを踏まえれば、わかりやすいファクターだけで課題を捉えると消すべきではないという帰結になります。  
一方でそのことはたとえば不要品に取り囲まれた工場の中で生産活動をするようなもので、直観的に何かがおかしいはずだとなるでしょう。

不要コードを放置した**目に見えない結果**とは、たとえば以下のようなものが考えられます。
- 関連性のないコードの解釈複雑性を増加させる(可読性を低下させる)
  - とくにリファクタリングを進めるに当たり、不要コードが邪魔になることが多い
  - grepで引っかかったりすると、実行時文脈を読み解く作業が必要になる
- 我々の認識能力というリソースを知らずのうちに削り、本来の開発対象への集中を妨げる
  - 工場であれば、不要品が空間というリソースを奪い、生産活動を停止させるという視覚的に明瞭な状況が簡単に想起できる
  - 認識能力は目に見えないものであり、減少していることを認識するには一定のセンスが必要になる
- オンボーディングコストを増大させる
  - これは(マネジメントを含む)潜在的課題に派生します

目に見えないものは存在しないという考え方に立脚すれば、これらは幽霊同様無視できるものですが、目に見えない課題というのは目に見える課題よりも遥かに扱いに困るものなのです。

## 不要コードに対抗する方法
不要コードに対抗するにはどうするべきか、さまざまなアプローチがあります。  
1つ目は不要になる開発を避ければよいですが、それは言い換えると精度の高い企画と要件定義を行うことを意味し、すなわちウォーターフォールを採用することになります。(極めて属人的であるためオススメはしないですが、筆者はこのプロセスを深く経験しており資質と気合があれば、成立させられると考えます。)  

この方法を再現性の観点から棄却するならば、不要コードを落とすための継続的なプロセス(**Continuous Deleting**)が必要になります。  
それはプログラミング言語に**ガベージコレクション**機構が付いている理由と似ています。  
工場とは異なりソフトウェアには不要コードを蓄積しきる十分なストレージがありますが、人の認識能力はそれほど大きくないのです。  

2つ目は仮に不要になったとしても、誰から見ても消しやすいようにコードを徹底的にシンプルに構成する方法です。これは現実的に執行しうるアプローチで、これに関する別の記事を書く予定があります。  
3つ目は、例外通知・監視ツール・アクセスログを活用して、利用状況を把握する手法です。  
この方法も使えますがたとえばアクセスログだけを見て、使われていないエンドポイントを洗い出すことは難しいです。    
4つ目はよりメカニカルに執行できる方法で、本番サーバでの実行時カバレッジを取得するという考え方です。
この方法の構成方法は昔にまとめているので[そちらの記事](https://note.com/paiza/n/n84a659795d95)を参照してください。
(時を経てコードは修正していますが、基本設計は変わっていません。)

## 不要コード削除の運用
paizaでは、DX向上デーという開発生産性を維持向上するための取り組みがあり、その中で上記の記事で示した不要コード検出ツールを作成しました。
今回は実運用していく上で作成したツールと取り組み結果について軽く紹介いたします。

不要コードを検出し、カバレッジデータをDBに蓄積することだけでは、`unused_lines = [1,3,5,9,...78]`のようなデータがあるだけで、それだけで削除行動をするのは大変です。  
不足しているのは実際のコードと紐付け、どの行が明確に使われていないかを**可視化**することです。
その方針でツールを作成することにしました。

![](/images/continuous-deleting-unused-codes/visualize-unused-code.png =500x)  
*不要コードの可視化の雰囲気、実際のコードを伏せるためマスキングしています*  

React × Vite × TypeScriptを使って1日で集中して作成しましたが、こういったツールをやっつけで作れるのがReactの強力さです。  
このツールを使って削除を進めていきましたが、3人×6日の活動でRuby不要コードの約1/4を削除できました。
基本的に削除しやすいところから削除していったというのもありますが、相手にしないといけないコードの数が減らせるだけでも、継続的な機能改修を進める上で有用です。

## まとめ
- 不要コードが生成され、削除されないインセンティブを説明しました
- 継続的なコード削除を戦略、戦術に織り込む重要性を説明しました
  - 象徴的に言えばContinuous Deleting
    - 残念ながらCD(Continuous Delivery)と頭文字が被っています
- 形式的にコードを不要と判断できるツールは有用です
  - それを起点として削除活動を推し進められます
    - 監視ツールやログ分析などを複合して利用することで削除によるリグレッションリスクを極限まで低減できます
    - 「この機能削除できますよね」のような問いかけも可能になります
      - いまのところそういった活動はやっていないですが、必要に応じてできます
- 可視化ツールをプロトタイピング的に作れるとたまに捗ります
  - Reactは表現したいものを端的に表現する上で便利です

paizaではさまざまな職種のエンジニアを募集しております。MVV(ミッション・ビジョン・バリュー)に共感できる方、paizaの取り組みに興味、関心がある方からの応募をお待ちしております。

https://paiza.jp/recruiters/9
