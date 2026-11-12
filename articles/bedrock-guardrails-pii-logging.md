---
title: "Bedrock Guardrailsで個人情報はどこまで隠せる？"
emoji: "🔐"
type: "tech"
topics: ["aws", "bedrock", "security", "cloudwatch", "個人情報"]
published: true
---

> **2026年11月13日（日本時間）時点**のAWS公式ドキュメントと、AWS上での検証に基づく備忘録です。仕様は変わる可能性があります。

最近、仕事でAmazon Bedrockを設定することになりました。そこでまず気になったのが、個人情報の扱いです。社内のルールで、入力された個人情報がシステムに保存されると、セキュリティ審査が厳しくなります。最初から保存しない形にできないかと考えました。

調べてみると、Bedrockには「Bedrock Guardrails」という機能があると知りました。これで個人情報をマスクすれば、保存も防げるのでは？と思ったのですが、そう単純ではありませんでした。モデルに渡す入力はマスクできても、**モデル呼び出しログには元の入力が残り得ます**。

この記事では、Bedrock Guardrailsで隠せる範囲と、Bedrockのログに個人情報を残さないための設定をまとめます。今回は架空の氏名・メールアドレスを使い、**入力のマスクとブロック、CloudWatch Logs・S3への記録、ログ表示時のマスクまで実際に確認しました**。

## まず結論

| データの流れ | Bedrock Guardrailsを有効にすると？ |
| --- | --- |
| ユーザー → モデル | 入力中の個人情報をマスクできる |
| モデル → ユーザー | 回答中の個人情報をマスクできる |
| ユーザー → Bedrockの呼び出しログ | ログを有効にすると、指定したCloudWatch LogsのロググループかS3バケットに**入力の原文**が記録される |

つまり、**Bedrock Guardrailsでモデルへの入力をマスクしても、Bedrockのログまでマスクされるわけではありません**。なお、個人情報の検出に漏れがないという保証もありません。[^sensitive]

:::details 今回の検証条件
| 項目 | 条件 |
| --- | --- |
| 検証日 | 2026年11月13日 |
| リージョン | シドニー（ap-southeast-2） |
| モデル | Amazon Nova Micro（amazon.nova-micro-v1:0） |
| 対象の機密情報 | 組み込みPIIのNAME・EMAIL |
| API | ApplyGuardrail、Converse |
| ログ保存先 | 専用のCloudWatch LogsロググループとS3バケット |
| ログのデータ保護 | 検証用ロググループにEmailAddressのポリシーを適用 |

氏名・メールアドレスはすべて架空の値です。キャプチャーは同じ設定で再実行して撮影し、アカウント情報が写らない範囲を掲載しています。検証後はモデル呼び出しログの設定を元に戻し、作成したリソースを削除しました。
:::

## データはどこを通るのか

```mermaid
flowchart TD
    U["👤 入力（原文）"] --> B["📨 Bedrock"]
    B --> G["🛡️ 入力をマスク"]
    G --> M["🤖 モデル"]
    M --> O["🛡️ 回答をマスク"]
    O --> R["💬 ユーザーへ回答"]
    B -.-> L["🗄️ 呼び出しログ"]
```

点線は、Bedrockの「モデル呼び出しログ記録」を有効にした場合だけ発生します。この設定は初期状態では無効です。[^logging]

### Bedrockのログはどこに保存される？

日本語表示のAmazon Bedrockコンソールでは、左側のメニューから「設定」を開き、「モデル呼び出しログ記録」を有効にします。続いて、ログの保存先として次のいずれかを選びます。[^logging]

- **Amazon S3のみ**：指定したバケットに圧縮JSONのログを保存。
- **CloudWatch Logsのみ**：指定したロググループ内のログイベントに保存。
- **Amazon S3とCloudWatch Logsの両方**：両方へ出力。

**保存先を選んでログを有効にする場所は、CloudWatch LogsもS3もBedrockの設定画面です。** IAMでログのオン・オフを設定するわけではありません。CloudWatch Logsへ出す場合、Bedrockがロググループへ書き込めるIAMロールが別途必要です。S3へ出す場合は、Bedrockがバケットへ書き込めるバケットポリシーが必要です。AWS公式手順では、設定者に所定のS3権限があれば、バケットポリシーは設定時に自動で追加されると説明しています。[^logging]

例えばCloudWatch Logsを選び、ロググループを`/aws/bedrock/my-invocations`と名付ければ、そこにログが入ります。この名前は例で、固定名ではありません。CloudWatch Logsを選んでも、大きな本文の保存先としてS3を別途設定する構成があります。[^logging]

### ① モデルへの入力と回答はマスクできる

**入力側：ユーザー → モデル**

```mermaid
flowchart TD
    I1["👤 送信前<br/>山田太郎のメールは<br/>taro@example.com です"]
    I1 --> G1["🛡️ マスク"]
    G1 --> I2["🤖 モデルへ<br/>{NAME}のメールは<br/>{EMAIL} です"]
```

**出力側：モデル → ユーザー**

```mermaid
flowchart TD
    O1["🤖 マスク前の回答<br/>連絡先は<br/>hanako@example.com です"]
    O1 --> G2["🛡️ マスク"]
    G2 --> O2["💬 ユーザーへ<br/>連絡先は<br/>{EMAIL} です"]
```

例えば「山田太郎のメールはtaro@example.comです」と入力したとします。対象の個人情報を検出し、入力側のマスクを設定していれば、モデルには氏名やメールアドレスを`{NAME}`や`{EMAIL}`へ置き換えて渡せます。出力側にも設定すれば、回答中の個人情報もマスクできます。`BLOCK`を選べば、内容をブロックすることもできます。[^sensitive]

#### 実際に日本語の氏名・メールアドレスをマスクしてみる

Bedrockコンソールで「Use ApplyGuardrail API」を有効にし、評価対象に「Input」を選択して、次の架空情報を送りました。この操作ではモデルを呼び出さず、ガードレール単体の変換結果を確認できます。

> 以下は検証用の架空情報です。顧客の氏名は山田太郎で、連絡用メールアドレスは taro@example.com です。

結果は、氏名が`{NAME}`、メールアドレスが`{EMAIL}`に置き換わりました。同じ文章を「Output」として評価した場合も、同様にマスクされました。

![Bedrockコンソールで日本語の氏名とメールアドレスがマスクされた結果](/images/bedrock-guardrails-pii-logging/01-guardrail-mask.jpg)
*上が入力した架空情報、下のGuardrails Responseがマスク後の文章です。*

#### 入力側の動作まで確認する

今回、PII種類の`action`だけを`ANONYMIZE`にし、`inputEnabled`・`outputEnabled`を有効にした条件では、入力中の氏名・メールアドレスは検出されたものの、入力評価の`action`は`NONE`でした。出力側ではマスクされていたため、**回答が隠れているだけでは、モデルに渡す前の入力もマスクされたとは判断できません**。

そこで、入力・出力それぞれの動作を明示しました。EMAILの設定例は次のとおりです。NAMEにも同じ動作を設定しています。[^actions]

```json
{
  "type": "EMAIL",
  "action": "ANONYMIZE",
  "inputAction": "ANONYMIZE",
  "outputAction": "ANONYMIZE",
  "inputEnabled": true,
  "outputEnabled": true
}
```

この設定では、ApplyGuardrailのInput・Output両方で置換され、Converseでも入力評価の`action`が`ANONYMIZED`になることを確認できました。これは今回のリージョン・DRAFT・設定の組み合わせでの結果です。入力のマスクを目的にする場合は、**入力側の動作を指定し、トレースの入力評価まで確認する**ようにします。

ただし、**元の入力はBedrock Guardrails（AWSサーバー）には届きます**。「AWSへ送る前に個人情報を消したい」場合は、送信する側で先にマスクします。

検出は確実ではありません。日本固有の番号や社内番号などは、対応するPII種類を確認し、必要ならカスタム正規表現や送信前のチェックを追加します。[^sensitive]

> **補足：Bedrockのモデルはどこで動く？**  
> Bedrockでは、Amazon NovaなどのAWS製モデルと、Claudeなど外部企業が開発したモデルを利用できます。外部モデルも提供会社のAPIへ入力を転送するのではなく、AWSが所有・運用する専用環境に配置されます。そのため、モデル提供会社はBedrockの入力や回答にアクセスできません。第三者モデルの一部はAWS Marketplace経由で課金されます。契約や料金の仕組みはモデルによって異なります。[^deployment][^billing]

### ② 呼び出しログには原文が残る

AWSは、Bedrock Guardrailsを適用しても、CloudWatch Logsのモデル呼び出しログの`input`には、加工前のリクエストが記録されると説明しています。[^sensitive] また、AWS公式Security Blogの[「Implementing least privilege access for Amazon Bedrock」](https://aws.amazon.com/blogs/security/implementing-least-privilege-access-for-amazon-bedrock/)でも、CloudWatch Logs・S3のログにはガードレール適用前の入力が含まれると説明されています。

#### 入力をマスクしてもログには原文が記録された

今回は、入力評価が`ANONYMIZED`になったConverseの呼び出しを、CloudWatch LogsとS3の両方で照合しました。どちらも、次のフィールドに**山田太郎・taro@example.comを含む原文**が残っていました。

```text
input.inputBodyJson.messages[0].content[0].text
```

![CloudWatch Logsのモデル呼び出しログに原文が記録されている画面](/images/bedrock-guardrails-pii-logging/02-cloudwatch-original.jpg)
*入力側のマスクを有効にした呼び出しでも、ログのinputには置換前の氏名・メールアドレスが含まれています。画像はCloudWatch Logsの入力フィールドの抜粋です。S3のログも取得して、同じ原文を確認しました。*

入力ブロックでも、`LAB_INPUT_BLOCKED`が返り、モデルの入出力トークンがともに0になった一方、入力の原文は両保存先に記録されました。逆に、送信前にメールアドレスを`[EMAIL_REDACTED]`へ置き換えた場合は、ログにも置換後の文章が記録されました。

| 実際に試した条件 | CloudWatch Logs・S3の入力ログ |
| --- | --- |
| Bedrock Guardrailsで入力をマスク | マスク前の原文が残る |
| Bedrock Guardrailsで入力をブロック | ブロック前の原文が残る |
| Bedrockへ送る前にマスク | 送信したマスク済みの文が記録される |

#### トレースにも元の値が含まれる

マスクが成功したConverseのトレースにも、検出結果の`match`として、元のメールアドレスが含まれていました。[^sensitive]

```json
{
  "match": "taro@example.com",
  "type": "EMAIL",
  "action": "ANONYMIZED",
  "detected": true
}
```

このため、**APIレスポンスのトレースをアプリ側でそのままログに出す場合も、原文の扱いを確認する必要があります**。

モデル呼び出しログはデフォルトで無効です。ログを有効にすると、CloudWatch Logs、S3、または両方へ入力・出力などを記録できます。CloudWatch Logsを選んでも、大きな本文などの保存先としてS3を設定していれば、そのデータはS3へ送られます。[^logging]

## CloudWatch Logsにはログの個人情報を隠す機能がある

CloudWatch Logsには「データ保護ポリシー」という機能があります。Bedrockの呼び出しログを保存するロググループに設定すると、メールアドレスなどの機密情報を検出し、**通常のログ表示でマスクできます**。原文を確認するには、`logs:Unmask`権限が必要です。[^cloudwatch]

### 通常表示と原文表示を比べる

検証用ロググループにEmailAddressのデータ保護ポリシーを設定し、反映を確認してから、`verified.mask@example.com`を含む架空情報をBedrockへ送りました。CloudWatch Logsの通常表示では、入力中のメールアドレスがアスタリスクになりました。

![CloudWatch Logsのデータ保護ポリシーによってメールアドレスがマスクされた通常表示](/images/bedrock-guardrails-pii-logging/03-cloudwatch-masked.jpg)
*通常表示では、入力中のメールアドレスがアスタリスクで隠れています。*

続いて、同じログを表示メニューの「保護されたデータのマスキングを一時的に解除」で確認すると、元の`verified.mask@example.com`が見えました。APIでも、`unmask=false`と`unmask=true`で同じ違いを確認しています。

![同じCloudWatch Logsのログをマスク解除して原文のメールアドレスを確認した画面](/images/bedrock-guardrails-pii-logging/04-cloudwatch-unmasked.jpg)
*上の画像と同じ呼び出しのログです。権限のある利用者がマスクを解除すると、元の値を確認できます。*

今回は、`logs:Unmask`権限を持つ同じ利用者で表示方法を切り替えた比較です。**権限があっても通常表示ではマスクされますが、原文を取得することは可能**でした。S3に直接保存されたログには、このCloudWatch Logsのポリシーによるマスクは適用されず、元の値が残っていました。

### 原文を保存しないための機能ではない

この結果からも、**表示時のマスクと、原文を保存しないことは区別する必要があります**。データ保護ポリシーは原文を削除する機能ではありません。設定前に取り込まれたログには遡って適用されず、S3へ直接保存したデータも対象外です。[^cloudwatch]

実際に、ポリシー設定前に取り込まれたログは、設定後も通常表示で原文が見えました。また、設定直後にはマスクされなかったログもあったため、新しく投入したテスト用ログがマスクされることを確認してから、Bedrockを再度呼び出しています。

呼び出しログに原文を残したくないなら、**ログを有効にしない**か、**Bedrockへ送る前に入力をマスクする**ことを検討します。

## 「学習に使われない」ならBedrock Guardrailsは不要？

AWSのAmazon Bedrock紹介ページは、Bedrockがデータをモデル学習に使用しないと説明しています。文言は次のとおりです。[^faq]

> 「Bedrock は、モデルのトレーニングを行うためにデータを保存または使用することはなく」

それでもBedrock Guardrailsには意味があります。**「学習に使わない」と「回答やログから漏れない」は別**だからです。具体的には、次のような情報漏洩を防ぐのに役立ちます。

- 顧客との会話を要約する前に、不要な氏名やメールアドレスを入力側でマスクする。
  - **理由：** モデルにはマスク後の文章が渡るため、回答に同じ氏名やメールアドレスが再び出てくる可能性を下げられます。
- モデルの回答に個人情報が含まれていたら、出力側でマスクする。
  - **理由：** ユーザーの画面に表示される前に隠せるため、そのままメールへ転記したり、ファイルに保存してGitへ上げたりする事故を防ぎやすくなります。
- AWSアクセスキーなどを誤って入力したらブロックする。
  - **理由：** モデルへ渡る前に処理を止められます。ただし、モデル呼び出しログを有効にしている場合は、ログに入力の原文が残り得ます。

ただし、ツール呼び出しの引数やツールの結果は、このフィルターの対象外です。AIにファイルを書かせてGitへ上げる運用なら、コミット内容の検査も必要です。[^sensitive]

また、**「呼び出しログが無効＝AWS側にも一切保持されない」ではありません**。AWS側の推論データ保持は別の設定とモデルごとの条件で決まります。公式文書には、次の注意書きがあります。[^retention]

> 「`store=false`を設定しても、データ保持がゼロであるとは限りません。」

「`store=false`でもゼロ保持は保証されない」という意味です。保存を禁止する要件がある場合は、対象モデルの保持設定も確認します。[^retention]

## 何を設定すればいい？

目的ごとに対策を選びます。

| 目的 | 対策 |
| --- | --- |
| モデルに個人情報を渡したくない | 入力側のBedrock Guardrailsでマスク動作を指定し、入力評価まで確認。AWSへの送信自体を避けるなら送信前にマスク |
| 回答で個人情報を見せたくない | 出力側のBedrock Guardrailsと、ツール出力・ファイル内容の検査 |
| Bedrockのログに原文を残したくない | モデル呼び出しログを有効にしない。またはBedrockへ送る前に入力をマスク |
| ログを見る人に原文を見せたくない | CloudWatch Logsのデータ保護と閲覧権限 |
| AWS側でも保持させたくない | 対象モデルのデータ保持モード |

覚えておきたいのは一つだけ。**Bedrock Guardrailsでマスクしても、ログに残る原文まで消えるわけではない**、ということです。

## 参照したAWS公式ドキュメント

[^sensitive]: [機密情報フィルターを使用して会話から PII を削除する — Amazon Bedrock](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/guardrails-sensitive-filters.html)。入力・出力のマスク、モデル呼び出しログとトレースの例外、ツール利用時の対象外を確認。
[^logging]: [CloudWatch Logs と Amazon S3 を使用してモデル呼び出しをモニタリングする — Amazon Bedrock](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/model-invocation-logging.html)。コンソールでの設定手順、デフォルト設定、保存先、ログ形式、S3への大容量データ保存を確認。
[^cloudwatch]: [機密性の高いログデータをマスキングで保護する — Amazon CloudWatch Logs](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/logs/mask-sensitive-log-data.html)。閲覧・転送時のマスク、`logs:Unmask`、既存ログへの非遡及を確認。
[^faq]: [Amazon Bedrock — AWS](https://aws.amazon.com/jp/bedrock/)。入力データをモデルのトレーニング目的で保存・使用しないことを確認。
[^retention]: [データ保持 — Amazon Bedrock](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/data-retention.html)。AWS側の推論データ保持が、モデル・設定によって異なる点を確認。
[^deployment]: [データ保護 — Amazon Bedrock](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/data-protection.html)。モデルはAWSが所有・運用するModel Deployment Accountへ配置され、モデル提供元はその環境や入力・出力へアクセスできないことを確認。
[^billing]: [モデルへのアクセスをリクエストする — Amazon Bedrock](https://docs.aws.amazon.com/ja_jp/bedrock/latest/userguide/model-access.html)。第三者モデルのAWS Marketplace購読とEULAを確認。

[^actions]: [Options for handling harmful content detected by Amazon Bedrock Guardrails — Amazon Bedrock](https://docs.aws.amazon.com/bedrock/latest/userguide/guardrails-harmful-content-handling-options.html)。入力・出力ごとの動作指定と、マスク・ブロック・検出のみの違いを確認。
