---
title: "SendGridのSandboxモードを導入して開発環境のメール送信を抑制した話"
emoji: "🕌"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: [sendgrid, email, ruby, sidekiq]
published: false
---

## 想定読者
この記事は以下の方を対象としています：
- 開発環境でのメール送信コストや誤送信リスクに悩んでいるエンジニア
- SendGridを利用している、または導入を検討している開発チーム  
- Ruby on Rails + Sidekiqを使ったアプリケーションを運用している方
- 環境別に柔軟なメール送信制御を実現したい方

## この記事で伝えたいこと

開発環境で大量のテストメールが実際に送信されていた問題を、SendGridのSandboxモードとSidekiqミドルウェアの組み合わせで解決し、以下を実現しました：

1. **コスト削減**: 送信リクエスト数を**約95%削減**し、大幅なコストカットを実現
2. **誤送信リスクゼロ**: 開発環境からの実ユーザーへの誤送信を完全防止
3. **柔軟な制御**: ジョブ単位・ドメイン単位での送信制御を実現

本記事では、Sidekiqのミドルウェアを活用した実装方法と、運用で得られた知見を共有します。

## はじめに：なぜSandboxモードが必要だったのか
### 直面していた3つの課題

#### 1. コスト問題
- 開発環境で大量のテストメールが実際に送信されていた
- SendGridの送信リクエスト数ベース課金による無駄なコスト発生
- テスト実行のたびに課金が積み上がる状況

#### 2. 誤送信リスク
- 開発環境から実ユーザーへの誤送信の危険性
- 情報漏洩やユーザー混乱のリスク
- 一度の設定ミスで大量誤送信の可能性

具体的な例：
- テスト用アカウントデータが本番ユーザーのメールアドレスを含んでいた場合、テスト内容が実ユーザーに送信される
- 開発環境でパスワードリセット機能をテストした際に、誤って実ユーザーにリセットメールが送信される  
- 一括送信機能のテストで、意図せず本番ユーザー全員にテストメールが配信される

#### 3. テスト環境の制約
- 実際の送信フローを検証したいが、実送信はしたくない
- 「現実的なテスト」と「安全性」の両立が困難
- 環境ごとの柔軟な制御が必要

## 解決策：SendGrid Sandboxモードの活用

### Sandboxモードとは
SendGridのAPIに対して送信リクエストは行うが、実際のメール配信はスキップする機能です。

### 📧 通常モード vs Sandboxモード の動作比較

```mermaid
flowchart LR
  subgraph "通常モード(本番環境)"
    App1[アプリ] -->|API呼び出し| SG1[SendGrid]
    SG1 -->|✅ メール送信| User1[👤 ユーザー]
    SG1 -.->|200 OK| App1
  end
  
  subgraph "Sandboxモード(開発環境)"
    App2[アプリ] -->|API呼び出し| SG2[SendGrid]
    SG2 -->|❌ 送信しない| Block[🚫]
    SG2 -.->|202 ACCEPTED| App2
  end
  
  style App1 fill:#ffd4d4
  style SG1 fill:#ffd4d4
  style User1 fill:#ffe4e4
  
  style App2 fill:#d4ffd4
  style SG2 fill:#d4ffd4
  style Block fill:#e4ffe4
```

### ポイント解説

#### 通常モード（本番環境）
- アプリがSendGrid APIを呼び出す
- SendGridが**実際にメールを送信**
- ユーザーのメールボックスに届く
- APIは「200 OK」を返す

#### Sandboxモード（開発・ステージング環境）
- アプリがSendGrid APIを呼び出す（**同じコード**）
- SendGridは**メール送信をスキップ**
- ユーザーには何も届かない（**安全**）
- APIは「202 ACCEPTED」を返す

## 実装方針：Sidekiqミドルウェアでジョブ単位制御
### 設計のポイント
1. **環境別デフォルト設定**: 開発/ステージングはSandbox有効、本番は無効
2. **ジョブ単位の制御**: Sidekiqミドルウェアでジョブごとにフラグ管理
3. **例外処理**: 社内ドメイン宛など特定条件での実送信を許可
4. **透過的な実装**: 既存コードへの影響を最小限に

### 🎯 Sidekiqミドルウェアを使った実装アプローチ

```mermaid
flowchart LR
  subgraph "📨 メール送信ジョブの流れ"
    App[アプリ] -->|1.ジョブ登録| CMW[🔧 Client MW]
    CMW -->|2.Sandboxフラグ付与| Redis[(Redis)]
    Redis -->|3.ジョブ取得| SMW[🔧 Server MW]
    SMW -->|4.フラグ設定| Worker[Worker]
    Worker -->|5.メール送信| SG[SendGrid API]
  end
  
  style CMW fill:#ffe4b5
  style SMW fill:#ffe4b5
  style SG fill:#e6f3ff
```

#### 動作の流れ
1. **ジョブ登録**: アプリがメール送信ジョブを登録
2. **フラグ付与**: ClientミドルウェアがSandboxフラグをジョブに追加
3. **ジョブ保存**: Redisキューにフラグ付きジョブを保存
4. **フラグ読み込み**: Serverミドルウェアがフラグを読み込み
5. **API呼び出し**: SendGrid APIにSandboxモード付きでリクエスト

## 実装の詳細

### Step 1: 環境変数でデフォルト設定
`PREFER_SENDGRID_SANDBOX_MODE` を boolean で管理します。

```ruby
# config/sendgrid.rb など
PREFER_SENDGRID_SANDBOX_MODE = ENV["PREFER_SENDGRID_SANDBOX_MODE"] == "true"
```

### Step 2: Sidekiq Clientミドルウェア（ジョブ登録時）
ジョブにSandboxフラグを付与します。

```ruby
module Sidekiq
  module Middleware
    module Client
      class SendGridSandboxMarker
        # SendGrid のサンドボックスモード設定をジョブに付与する
        def call(_worker_class, job, _queue, _redis_pool)
          job['prefer_sendgrid_sandbox_mode'] = true if ENV.fetch('PREFER_SENDGRID_SANDBOX_MODE', 'false') == 'true'

          yield
        end
      end
    end
  end
end
```

### Step 3: Sidekiq Serverミドルウェア（ジョブ実行時）
Worker実行時にSandboxフラグをThread localに設定します。

```ruby
module Sidekiq
  module Middleware
    module Server
      # SendGrid のサンドボックスモードをスレッドに設定する
      class SendGridSandboxFlagger
        # @param worker [Object] 実行中のワーカーインスタンス
        # @param job [Hash]    ジョブ情報
        # @param queue [String] キュー名
        def call(_worker, job, _queue)
          Thread.current[:sidekiq_prefer_sendgrid_sandbox_mode] = true if job['prefer_sendgrid_sandbox_mode']

          yield
        ensure
          Thread.current[:sidekiq_prefer_sendgrid_sandbox_mode] = nil
        end
      end
    end
  end
end
```

### Step 4: 例外処理の実装（社内ドメイン宛は実送信）

```ruby
module SendGridSandbox
  INTERNAL_DOMAINS = ['xxx.jp', 'xxx.co.jp'].freeze

  class << self
    # sandbox が「実際に」有効かどうか判定
    # グローバル設定が有効かつ社内ドメインを含まない場合のみ true
    # @param recipients [Array<String>] 送信先メールアドレス一覧
    # @return [Boolean]
    def enabled?(recipients:)
      sandbox_enabled = Thread.current[:sidekiq_prefer_sendgrid_sandbox_mode] ||
                        ENV.fetch('PREFER_SENDGRID_SANDBOX_MODE', 'false') == 'true'
      return false unless sandbox_enabled

      # 社内ドメインチェック：含まれている場合は false
      return false if recipients.any? do |email|
        domain = email.split('@')[1]
        INTERNAL_DOMAINS.include?(domain)
      end

      true # 社内ドメインが含まれていない
    end

    # サンドボックスモードを有効にした SendGrid::MailSettings を生成して返す
    # @return [::SendGrid::MailSettings]
    def sandbox_mail_settings
      mail_settings = ::SendGrid::MailSettings.new
      mail_settings.sandbox_mode = SendGrid::SandBoxMode.new(enable: true)
      mail_settings
    end
  end
end
```

### Step 5: SendGrid API呼び出し時の適用

```ruby
    sg_mail = ::SendGrid::Mail.new
    .
    .
    .

    # 全受信者を対象にサンドボックスが有効か判定し、有効なら設定を適用
    all_recipients = to_emails | cc_emails | bcc_emails

    # サンドボックスモードが有効な場合、SendGridのMailSettingsにサンドボックスモードを設定
    if SendGridSandbox.enabled?(recipients: all_recipients)
      sg_mail.mail_settings = SendGridSandbox.sandbox_mail_settings
    end
    .
    .
    .
```

## 導入効果

### 📉 主要指標の改善

| 指標 | 改善結果 |
|:---|:---|
| **送信リクエスト数** | **約95%削減** |
| **コスト** | **80%以上削減** |
| **誤送信リスク** | **100%防止** |

### 📊 環境別の改善効果

- **開発環境・ステージング環境**: 95%削減（社内ドメイン宛のみ実送信）
- **本番環境**: 影響なし（通常通り送信）

特に開発・ステージング環境での大量配信テストの送信抑制が大きく貢献しています。

## 運用上のポイント

### ✅ メリット
- 環境別の柔軟な制御が可能
- 既存コードへの影響が最小限
- ジョブ単位での細かい制御が可能
- 社内ドメイン宛などの例外処理が容易

### ⚠️ 注意点
- SandboxモードでもAPIリクエスト課金は発生（ただし大幅削減）
- SMTP経由の送信には未対応（Web APIのみ）  
- 本番環境での誤適用防止のため、環境変数の管理が重要
- デプロイ時の設定確認が必須

## まとめ

SendGridのSandboxモードとSidekiqミドルウェアを組み合わせることで、以下の3つの目標を達成しました：

### 🎯 達成した成果
1. **コスト削減**: 送信リクエスト数を95%削減し、大幅なコストカットを実現
2. **リスク排除**: 開発環境からの誤送信ゼロを実現  
3. **柔軟な制御**: 環境別・ジョブ別・ドメイン別の細かい制御を実現

### 💡 得られた知見

#### 技術面
- Sidekiqミドルウェアによるジョブ単位制御が非常に効果的
- Thread localを活用したスコープ限定の設定管理が有効
- 既存コードへの影響を最小限にした設計が重要

#### 運用面  
- 環境変数による制御でデプロイがシンプルに
- 社内ドメイン宛の例外処理が検証効率を大幅改善

### 🚀 今後の展望
- SMTP経由の送信への対応検討
- 他サービスへの横展開

本取り組みは、コスト削減とリスク低減を両立しながら開発効率を維持する、実用的なソリューションとなりました。同様の課題を抱えるチームの参考になれば幸いです。
