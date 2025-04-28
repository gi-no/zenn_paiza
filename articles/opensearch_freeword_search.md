---
title: "OpenSearchを用いたフリーワード検索の改善"
emoji: "👻"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["opensearch", "kuromoji", "paiza"]
published: false
---

# はじめに

paizaでエンジニアをやってる林です。

paizaでは求人票の全文検索（フリーワード検索）にOpenSearchを使用しています。

このOpenSearchを導入してから、「一部の企業で、自社名を入力してもヒットしない」という事象が起きていました。

そこで、社内でこの課題を解決したい！というメンバーでチームを組み、改善を図ることとなりました。

今回は、その改善に向けて調査した内容と、具体的な改善アクションについてご紹介したいと思います。

## 現状の課題

今回挙げた課題について、具体的な企業名は挙げられないので…今回は「はやしかぶしきがいしゃ」という企業名を題材にしていきます。（全部ひらがなの会社ってあまりないかと思いますが…説明用として許してください）

まずはいくつかのワードで検索し、この企業がヒットするかまずは確認してみます。

| 検索ワード | 検索結果 |
| --- | --- |
| はや | ◯ |
| はやし | × |
| ぶす | ◯ ← ???? |

本来であれば「はやし」でもワードでもヒットしてほしいところがヒットしていませんね…。しかも「ぶす」というワードでヒットしてしまう状況です。ユーザー体験としてはよくなさそうですね…。

# 調査

というわけで、この原因を探っていきたいと思います。

## 現在のフリーワード検索

まずは現状の実装について説明したいと思います。

paizaのフリーワード検索ですが、OpenSearchのIndexにフリーワード検索用のpropertyを用意し、そこに対して検索をかけるようになっています。そのpropertyには、検索対象としたいカラムをcopy_toで追加しています。今回課題としている「企業名」もこのカラムに追加されています。

```json
{
  "mappings": {
    "properties": {
      "recruiter_name": { <= 企業名のカラム
        "type": "text",
        "copy_to": "free_word"
      },
      ...
      "free_word": { <= 企業名・求人名などが挿入されている
        "type": "text",
        "analyzer": "kuromoji"
      }
    }
  }
}
```

また、上記設定を見ていただくと分かる通り、日本語の検索にはkuromojiを使用しています。
kuromojiのカスタマイズはしておらず、デフォルト設定で使用しています。

このcopy_toとanalyzerですが、copy_to元のanalyzerはcopy_to先には適用されませんでした。

今回の場合ですと、コピー元であるrecruiter_nameにanalyzerは指定されていませんが、コピー先にはkuromojiが指定されております。そのため、フリーワード検索をした際に、企業名はkuromojiの形態素解析がかけられている状態となっています。

コピー元では、analyzerを指定していないので、copy_to先ではLIKE検索のような動きになると思っていたのですが…どうやら違いました。

今回意図した検索結果とならなかったのは、ここに問題がありそうです。

## kuromojiによる形態素解析

では、「はやしかぶしきがいしゃ」はどのように形態素解析されているのでしょうか？

kuromojiの形態素解析にかけてみましょう。

```
GET sample_index/_analyze
{
  "analyzer": "kuromoji",
  "text": "はやしかぶしきがいしゃ"
}
```

上記を実行すると以下のような結果が返ってきました。

```json
{
  "tokens" : [
    {
      "token" : "はや",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "ぶす",
      "start_offset" : 4,
      "end_offset" : 6,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "いしゃ",
      "start_offset" : 8,
      "end_offset" : 11,
      "type" : "word",
      "position" : 5
    }
  ]
}
```

「はやしかぶしきがいしゃ」という文字列は「はや / ぶす / いしゃ」で分割されていることがわかります。

そのため「はやし」ではヒットせず、「ぶす」でヒットするという状況だったと考えられます。

## kuromojiのfilterオプション

では、この形態素解析のやり方は変えられるのでしょうか？

kuromojiには、どのように解析をかけるかいくつかオプションが用意されており、これらの設定を有効/無効にすることで変えることができます。

kuromojiのフィルターには以下のようなものがあります。

（詳しくは [https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/analysis-kuromoji-analyzer.html](https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/analysis-kuromoji-analyzer.html) などを参照ください）

| kuromoji_baseform | 基本形を活用形に変換する         | はや / しか / ぶす / き / が / いしゃ |
| --- |----------------------| --- |
| kuromoji_part_of_speech | 助詞や記号などを除外する         | はや / ぶし / いしゃ |
| ja_stop | 一般的な日本語のストップワードを除外する | はや / しか /ぶし / いしゃ |
| kuromoji_number | 漢数字をアラビア数字に正規化する     | 壱百満天原サロメ => 壱100満天原サロメ |
| kuromoji_stemmer | JISZ8301の表記揺れを正規化する  | **サーバー** => **サーバ** |
| lowercase | アルファベットを小文字にする       | OpenSearch => opensearch |
| cjk_width | 英字を半角に、CJKを全角に統一する   | ｈｅｌｌｏﾊﾛｰ => hello**ハロー** |

フィルターの組み合わせをいくつかためしたみたところ、どうやらデフォルトではcjk_width以外が適用されているようでした。

## 改善へのアプローチ

では、この結果を踏まえて具体的な改善をいっていきたいと思います。

### kuromojiオプションを調整する

現状kuromojiのデフォルト設定、つまりcjk_width以外のオプションが適用されている状態でしたが、特段外すべきオプションが見当たらなかったので現状のままでよさそうという結論となりました。

また、cjk_widthのオプジョンについては、有効にしてもデメリットがほとんどなく、むしろ検索精度の向上が期待できそうなので、こちらは追加することにしました。

```json
"analyzer": {
  "kuromoji_custom_analyzer": {
    "type": "custom",
    "tokenizer": "kuromoji_tokenizer",
    "filter": [
      "kuromoji_baseform",
      "kuromoji_part_of_speech",
      "ja_stop",
      "kuromoji_number",
      "kuromoji_stemmer",
      "lowercase",
      "cjk_width"
    ]
  }
}
...   
"free_word": {
  "type": "text",
  "analyzer": "kuromoji_custom_analyzer"
}
```

### 企業名に形態素解析をかけない

copy_toを行った場合、コピー先のanalyzerが適用されてしまうため、そもそも「企業名」のような固有名詞はいわゆるLIKE検索で切り出したほうがよさそうという結論になりました。

具体的には、

① 企業名をcopy_toの対象から外す

```
"recruiter_name": {
  "type": "text",
  "copy_to": "free_word"
},

↓

"recruiter_name": {
  "type": "keyword",
},
```

そして、フリーワード検索の際は、従来のkuromojiによるフリーワード用のpropertyに対する検索と企業名へのLIKE検索をORで処理する形としました。

```
※検索クエリのイメージ
{
  "bool": {
    "should": [
      {
        "wildcard": {
          "recruiter_name": { "value": "*はやし*" }
        }
      },
      {
        "match_phrase": { "free_word": "はやし" } }
      }
    ]
  }
}
```

# 結果

では、上記対応を行ったうえで、再度「はやしかぶしきがいしゃ」に対して、同じワードで検索をやってみます。

| 検索ワード | 検索結果 |
| --- | --- |
| はや | ◯ |
| はやし | ◯ |
| ぶす | × |

「はやし」でヒットするのようになり、「ぶす」ではヒットしなくなりましたね！

ﾔｯﾀｰ!

# まとめ

今回、企業名という固有名詞を形態素解析の対象から外し、LIKE検索とすることで期待した検索結果を得ることができました。

固有名詞などフィールド特性に応じて、アナライザーや形態素解析を調整することが、検索精度の向上には重要そうです。
