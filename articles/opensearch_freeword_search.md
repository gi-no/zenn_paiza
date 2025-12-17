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

例えば「しすてむや株式会社」という企業があったとします。
paizaのフリーワード検索を使って「しすてむ」で検索するとヒットしない状態です。

企業名の検索を期待しているので、like検索のようにヒットしてほしいところです。
なぜ検索でヒットしないのか、OpenSearchやkuromojiの仕組みを少し見ていきたいと思います。

# 調査

## 現在のフリーワード検索

まずは現状の実装について説明したいと思います。

paizaのフリーワード検索ですが、OpenSearchのインデックスにはフリーワード検索用のproperty(`free_word`)を用意しています。
企業名のようにフリーワード検索に含めたい求人情報には`"copy_to": ["free_word"]`を設定しています。

`free_word`プロパティには求人票の概要情報も含まれているため、kuromojiという日本語形態素解析エンジンを入れています。


```json:インデックスのイメージ
PUT tmp_index
{
  "mappings": {
    "properties": {
      "recruiter_name": {
        "type": "text",
        "copy_to": ["free_word"]
      },
      "free_word": {
        "type": "text",
        "analyzer": "kuromoji"
      }
    }
  }
}
```

## kuromojiによる形態素解析

今回の「しすてむや株式会社」はkuromojiによってどのように解析されているのでしょうか？
```json:形態素解析の結果
GET tmp_index/_analyze
{
  "analyzer": "kuromoji",
  "text": "しすてむや株式会社"
}

## 実行結果 ##
{
  "tokens" : [
    {
      "token" : "しす",
      "start_offset" : 0,
      "end_offset" : 2,
      "type" : "word",
      "position" : 0
    },
    {
      "token" : "むや",
      "start_offset" : 3,
      "end_offset" : 5,
      "type" : "word",
      "position" : 2
    },
    {
      "token" : "株式",
      "start_offset" : 5,
      "end_offset" : 7,
      "type" : "word",
      "position" : 3
    },
    {
      "token" : "株式会社",
      "start_offset" : 5,
      "end_offset" : 9,
      "type" : "word",
      "position" : 3,
      "positionLength" : 2
    },
    {
      "token" : "会社",
      "start_offset" : 7,
      "end_offset" : 9,
      "type" : "word",
      "position" : 4
    }
  ]
}
```

どうやら「しすてむや株式会社」は「しす / むや / 株式 / 株式会社 / 会社」に分解されているようです。

この結果を踏まえて、改めていくつかのワードにヒットするのか試してみたいとと思います。
```json
GET tmp_index/_search
{
  "query": {
    "bool": {
      "must": [{
        "match_phrase": {
          "free_word": "しすてむ"
        }
      }]
    }
  }
}
```
|検索ワード|検索結果|
|---|--|
|しす|ヒットする|
|むや|ヒットする|
|しすてむ|ヒットしない|


kuromojiの形態素解析がかかっているためか、「しすてむ」というlike検索のようなヒットの仕方はせず、
代わりに形態素解析の出力結果にあった「しす」「むや」というワードでヒットするようになっているようです。

企業名は固有の単語なので、形態素解析が行われないようにしたほうがよさそうです。

## kuromojiのfilterオプション

では、この形態素解析のやり方は変えられるのでしょうか？

kuromojiには、どのように解析をかけるかいくつかオプションが用意されており、これらの設定を有効/無効にすることで変えることができます。

kuromojiのフィルターには以下のようなものがあります。

| オプション名                  | 概要                   | 例                          |
|-------------------------|----------------------|----------------------------|
| kuromoji_baseform       | 基本形を活用形に変換する         | 飲み → 飲む                    |
| kuromoji_part_of_speech | 助詞や記号などを除外する         | 寿司がおいしいね → 寿司 / おいしい       |
| ja_stop                 | 一般的な日本語のストップワードを除外する | 寿司などがおいしいね → 寿司 / おいしい / ね |
| kuromoji_number         | 漢数字をアラビア数字に正規化する     | 十二万三千円 → 123000円           |
| kuromoji_stemmer        | JISZ8301の表記揺れを正規化する  | **サーバー** => **サーバ**        |
| lowercase               | アルファベットを小文字にする       | OpenSearch => opensearch   |
| cjk_width               | 英字を半角に、CJKを全角に統一する   | ｈｅｌｌｏﾊﾛｰ => hello**ハロー**   |

ElasticSearchの記事ですが、こちらが参考になるかと思います。
[https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/analysis-kuromoji-analyzer.html](https://www.elastic.co/guide/en/elasticsearch/plugins/7.9/analysis-kuromoji-analyzer.html) 


フィルターの組み合わせをいくつかためしたみたところ、どうやらデフォルトではcjk_width以外が適用されているようでした。

# 改善へのアプローチ

では、この結果を踏まえて具体的な改善を行っていきます。

## kuromojiオプションを調整する

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

## 企業名に形態素解析をかけない
企業名のプロパティ(`recruiter_name`)をフリーワード用のプロパティ(`free_word`)にコピーしていますが、
`recruiter_name`は`type: "text"`となっており、kuromojiの形態素解析をかけていません。
`free_word`に対して「しすてむ」で検索してもヒットしませんでしたが、`recruiter_name`に対しては同様に検索したらヒットすることがわかりました。

```json
# free_wordはkuromojiの形態素解析がかかっており、期待通りにヒットしない
GET tmp_index/_search
{
  "query": {
    "bool": {
      "must": [{
        "match_phrase": {
          "free_word": "しすてむ"
        }
      }]
    }
  }
}

# recruiter_nameはkuromojiの形態素解析がかかっていないので、企業名の一部でもヒットする
GET tmp_index/_search
{
  "query": {
    "bool": {
      "must": [{
        "match_phrase": {
          "recruiter_name": "しすてむ"
        }
      }]
    }
  }
}
```

つまり、コピー元とコピー先で結果が異なったことから、
copy_toを使った場合、typeの設定は引き継がれず、コピー先のtypeが適用されるという動きのようです。


そのため、

- 企業名はフリーワード検索用のプロパティには含めない
- フリーワード検索時は、従来の検索ロジックに加え、or条件で企業名のLIKE検索を加える

という対応をすることにしました。


具体的には、

① 企業名をcopy_toの対象から外し、typeをtextからkeywordに変更
```json
"recruiter_name": {
  "type": "text",
  "copy_to": "free_word"
},

↓

"recruiter_name": {
  "type": "keyword",
},
```

② OpenSearchへの検索クエリをor条件に変更して、企業名のLINE検索を加える
```json
GET temp_index/_search
{
  "query": {
    "bool": {
      "should": [
        {
          "wildcard": {
            "recruiter_name": { "value": "*しすてむ*" }
          }
        },
        {
          "match_phrase": { "free_word": "しすてむ" }
        }
      ]
    }
  }
}
```

# 結果

では、上記対応を行ったうえで、再度「しすてむや株式会社」に対して、同じワードで検索をやってみます。

|検索ワード| 検索結果  |
|---|-------|
|しす| ヒットする |
|むや| ヒットする |
|しすてむ| ヒットする |

「しすてむ」というワードでヒットするようになりました！

# まとめ

今回、企業名という固有名詞を形態素解析の対象から外し、LIKE検索とすることで期待した検索結果を得ることができました。

固有名詞などフィールド特性に応じて、アナライザーや形態素解析を調整することが、検索精度の向上には重要そうです。
