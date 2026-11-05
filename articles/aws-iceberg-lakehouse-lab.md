---
title: "AWSでデータレイクハウスを構築してApache Icebergの更新・履歴管理を検証"
emoji: "🧊"
type: "tech"
topics: ["aws", "apacheiceberg", "athena", "s3", "glue"]
published: false
---

AWSでデータレイクハウスを構築する場合、どのようなサービスを組み合わせ、データをどのように管理するのか。まずは全体像を把握したくて、小さな検証環境を作ってみました。

今回はAmazon S3、AWS Glue Data Catalog、Amazon Athena、Apache Icebergを組み合わせ、テーブルの作成からデータの追加・更新・削除、過去のスナップショットの参照（Version Travel）、ファイルの最適化まで一通り試しています。

SQLの実行結果だけでなく、S3上のParquetファイルやメタデータも確認しながら、それぞれのサービスの役割とデータが管理される仕組みを整理します。

## 背景と目的

データレイクハウスは、データレイクの柔軟なストレージと、データウェアハウスに近いテーブル管理機能を組み合わせるアーキテクチャです。

データ基盤の歴史や、データレイクハウスが登場した背景については、以前の記事で整理しています。

https://zenn.dev/kkoch5t2/articles/41ad155b966681

AWSではAmazon S3をデータの保存先にし、Amazon Athenaなどのクエリエンジンから分析する構成が考えられます。

ただし、S3にParquetファイルを配置しただけでは、データウェアハウスと同じような更新処理や履歴管理はできません。

そこで使うのがApache Icebergです。

Apache Icebergは、Parquetなどのデータファイルをテーブルとして管理するためのオープンテーブルフォーマットです。スナップショットを利用した履歴管理や、行単位の更新・削除などに対応しています。[^iceberg]

Icebergのメタデータ階層や更新・削除の仕組みについては、以下の記事で詳しく解説しています。

https://zenn.dev/kkoch5t2/articles/9b1b800a58a6b7

今回は次の点を確認しました。

- AthenaからIcebergテーブルを作成し、SQLで操作できるか
- UPDATE・DELETEによってS3上のファイルがどう変化するか
- Version Travelで過去のスナップショットを参照できるか
- OPTIMIZE・VACUUMによってファイルがどう管理されるか

なお、今回はAWSデータレイクハウスの構成とIcebergの基本動作を理解するための最小構成です。Glue ETLによるデータ変換、Raw・Curated・Martの多層化、Lake Formationによる詳細な権限管理などは対象としていません。

## 前提

今回の検証環境は以下のとおりです。

| 項目 | 設定 |
| --- | --- |
| AWSリージョン | Sydney（ap-southeast-2） |
| 構築方法 | CloudFormation |
| ストレージ | Amazon S3 |
| データカタログ | AWS Glue Data Catalog |
| クエリエンジン | Amazon Athena engine version 3 |
| テーブルフォーマット | Apache Iceberg v2 |
| ファイル形式 | Parquet（Snappy圧縮） |
| データベース | `lakehouse_lab` |

今回は通常のS3バケットを使用しており、Amazon S3 Tablesは使用していません。また、GlueはData Catalogとして利用し、ETLジョブは実行していません。

## システム構成

今回の構成はシンプルです。

```mermaid
flowchart LR
    User["利用者"] -->|"SQL"| Athena["Amazon Athena"]
    Athena <-->|"テーブル登録情報"| Glue["AWS Glue Data Catalog"]
    Athena <-->|"読み書き"| Data["Amazon S3 データ・メタデータ"]
    Athena --> Results["Amazon S3 クエリ結果"]
```

Amazon S3にはIcebergのデータファイルとメタデータを保存します。AWS Glue Data Catalogにはテーブルの登録情報を保持し、AthenaがSQLの実行を担当します。

今回の構築で重要になるのは、IcebergがS3内のデータファイルを直接管理するのではなく、メタデータを通してテーブルの状態を管理していることです。

IcebergテーブルのS3上の構成は、概ね次のようになります。

```text
warehouse/orders/
├── data/
│   └── .../*.parquet
└── metadata/
    ├── 00000-....metadata.json
    ├── 00001-....metadata.json
    ├── snap-....avro
    └── ....-m0.avro
```

データファイルを格納する`data/`と、スナップショットなどを管理する`metadata/`が分かれていることが特徴です。

![S3のorders配下にdataフォルダとmetadataフォルダが並んでいる画面](/images/aws-iceberg-lakehouse/03-s3-orders.png)
*実際のS3の`warehouse/orders/`。`data/`と`metadata/`が分かれている。*

## 環境構築

環境はCloudFormationを使って構築しました。

### AWSリソースの作成

CloudFormationのスタック名は`lakehouse-lab`としました。

テンプレートでは次のリソースを作成しています。

- Icebergデータ保存用のS3バケット
- Athenaクエリ結果保存用のS3バケット
- Glueデータベース`lakehouse_lab`
- Athenaワークグループ`lakehouse-lab-wg`
- 検証用SQLを含むAthenaの保存済みクエリ9本

S3バケットにはパブリックアクセスブロックと暗号化を設定し、HTTPS以外のアクセスを拒否しています。また、Athenaのクエリ結果には7日間のライフサイクルルールを設定しました。

Athenaには1クエリあたり1GiBのスキャン量上限も設定しています。ただし、これは料金の上限ではないため、S3やAthenaなどの利用料金には別途注意が必要です。

今回使用したCloudFormationテンプレートは、検証用SQLも含めてGitHubで公開しています。同じ構成を試したい場合や、個々のリソースの設定を確認したい場合は、以下を参照してください。

https://github.com/kkoch5t2/aws-iceberg-lakehouse-cloudformation

CloudFormationの作成が完了したらAthenaを開き、`lakehouse-lab-wg`を選択します。

![CloudFormationスタックlakehouse-labのリソース一覧とCREATE_COMPLETE状態](/images/aws-iceberg-lakehouse/08-cloudformation.png)
*CloudFormationの`lakehouse-lab`スタック。作成したリソースが`CREATE_COMPLETE`となっている。*

ここで注意したいのが、CloudFormationが作成するのはIcebergテーブルそのものではなく、テーブルを作成するためのSQLも含む実行環境だという点です。

### Icebergテーブルの作成

Athenaで次のSQLを実行します。

```sql
CREATE TABLE lakehouse_lab.orders (
    order_id BIGINT,
    customer_name STRING,
    amount DECIMAL(10,2),
    status STRING,
    order_date DATE
)
PARTITIONED BY (month(order_date))
LOCATION 's3://<DATA_BUCKET>/warehouse/orders/'
TBLPROPERTIES (
    'table_type'='ICEBERG',
    'format'='parquet',
    'write_compression'='snappy'
);
```

`<DATA_BUCKET>`にはCloudFormationで作成されたデータ用S3バケット名を指定します。

`table_type=ICEBERG`でIcebergテーブルとして作成し、`month(order_date)`によって注文日を月単位にパーティション分割しています。[^create]

実行後、Glue Data Catalogから`lakehouse_lab.orders`が確認できました。

![AWS Glue Data Catalogでデータベースlakehouse_labにordersテーブルが登録されている画面](/images/aws-iceberg-lakehouse/09-glue-catalog-orders.png)
*Glue Data Catalogのテーブル一覧。データベース`lakehouse_lab`に`orders`が登録され、S3の保存先も確認できる。*

## 動作検証

### データの追加と参照

まずは3件の注文データを登録します。

```sql
INSERT INTO lakehouse_lab.orders
    (order_id, customer_name, amount, status, order_date)
VALUES
    (1, 'Alice', DECIMAL '1200.00', 'new', DATE '2026-11-01'),
    (2, 'Bob', DECIMAL '2500.00', 'new', DATE '2026-11-02'),
    (3, 'Carol', DECIMAL '800.00', 'new', DATE '2026-11-03');
```

`SELECT`を実行すると、期待どおり3件が取得できました。

| order_id | customer_name | amount | status |
| --- | --- | --- | --- |
| 1 | Alice | 1200.00 | new |
| 2 | Bob | 2500.00 | new |
| 3 | Carol | 800.00 | new |

この時点で、S3にはデータを格納するParquetファイルが作成されています。

### データの更新と削除

Athenaでは、Icebergテーブルに対する`UPDATE`と`DELETE`をサポートしています。[^data]

続いてAliceの金額とステータスを更新します。

```sql
UPDATE lakehouse_lab.orders
SET amount = DECIMAL '1350.00',
    status = 'paid'
WHERE order_id = 1;
```

さらにCarolの注文を削除します。

```sql
DELETE FROM lakehouse_lab.orders
WHERE order_id = 3;
```

再度SELECTすると、結果は以下のようになりました。

| order_id | customer_name | amount | status |
| --- | --- | --- | --- |
| 1 | Alice | 1350.00 | paid |
| 2 | Bob | 2500.00 | new |

![UPDATEとDELETE後のAthenaでの現在のSELECT結果（2件）](/images/aws-iceberg-lakehouse/01-current-rows.png)
*UPDATE・DELETE後に取得できた現在のデータは2件。*

通常のSQLによってUPDATEとDELETEが正常に実行できました。

ここで気になるのが、S3上の元のParquetファイルがどうなっているのかという点です。

### スナップショットの確認

Icebergのスナップショットは、Athenaから次のSQLで確認できます。

```sql
SELECT *
FROM "lakehouse_lab"."orders$snapshots"
ORDER BY committed_at DESC;
```

今回はINSERT、UPDATE、DELETEそれぞれに対応する3つのスナップショットが作成されていました。

| 処理 | スナップショットID |
| --- | --- |
| INSERT | 7508948322727723283 |
| UPDATE | 3135078563431764598 |
| DELETE | 1651369462038471101 |

スナップショットIDは実行環境ごとに異なります。

![Athenaで確認したIcebergの4つのスナップショット一覧](/images/aws-iceberg-lakehouse/07-snapshots.png)
*上の表はINSERT・UPDATE・DELETE時点の3件。スクリーンショットは後からOPTIMIZEも実行したため、合計4件となっている。*

これを利用すると、変更前のデータを参照できます。

```sql
SELECT *
FROM lakehouse_lab.orders
FOR VERSION AS OF 7508948322727723283
ORDER BY order_id;
```

このSQLでは、INSERT直後のAlice、Bob、Carolの3件が取得できました。

![AthenaのVersion TravelでINSERT直後の3件を取得した結果](/images/aws-iceberg-lakehouse/02-time-travel.png)
*Version Travelで取得した過去の状態。現在は削除済みのCarolも含まれている。*

つまり、現在のテーブルからCarolが削除されていても、保持されている過去のスナップショットを指定すれば、当時のデータを参照できます。

これはスナップショットIDを指定するVersion Travelです。Athenaでは、日時を指定するTime Travel（`FOR TIMESTAMP AS OF`）も利用できます。[^time-travel]

### S3のファイル構成

次に、Athenaで現在参照しているデータファイルを確認します。

```sql
SELECT
    content,
    file_path,
    record_count,
    file_size_in_bytes
FROM "lakehouse_lab"."orders$files";
```

UPDATE・DELETEを実行したあとの結果は次のとおりでした。

| ファイル | レコード数 | サイズ |
| --- | ---: | ---: |
| INSERT時のParquet | 3件 | 813バイト |
| UPDATE時のParquet | 1件 | 775バイト |

![OPTIMIZE前にAthenaのorders filesで確認したデータファイル2件](/images/aws-iceberg-lakehouse/05-files-before-optimize.png)
*OPTIMIZE前。現在のIcebergテーブルが参照するデータファイルは2つ。*

ここで重要なのは、**INSERT時のParquetファイルに元の3レコードが残っている**ことです。

S3上のファイルをさらに確認すると、行の削除情報を保持するPosition Deleteファイルが2つ作成されていました。

Position Deleteファイルには、削除対象となるデータファイルのパスと行の位置を記録します。

今回のUPDATEでは、元のAliceを削除扱いにしてから更新後のAliceを新しいParquetファイルに追加しています。DELETEでは、Carolの位置が削除対象として記録されていました。

IcebergのメタデータJSONにも、データファイルが2つ、削除ファイルが2つという情報が記録されていました。

![S3のmetadata配下に保存されたIcebergメタデータJSONとAvroのファイル一覧](/images/aws-iceberg-lakehouse/04-s3-metadata.png)
*S3の`metadata/`には、スナップショットやファイルの管理に使うJSON・Avroファイルが保存されている。*

したがって、物理的なデータファイルには合計4レコードがあっても、削除情報を適用したSQLの結果は2件になります。

### ファイルの最適化

:::message
**OPTIMIZEとは？**

Apache Icebergテーブルのデータファイルを整理し、読み込みやすい構成に書き換えるメンテナンス処理です。[^optimization]

データの追加や更新・削除を繰り返すと、小さなParquetファイルや削除情報を記録したファイルが増え、クエリ実行時の処理が複雑になることがあります。OPTIMIZEでは、小さなデータファイルをまとめたり、削除情報を反映したデータファイルに書き換えたりすることで、読み込み時の負荷を減らします。

今回の`REWRITE DATA USING BIN_PACK`は、ファイルを効率よくまとめ直す方式です。なお、OPTIMIZEは古いスナップショットを即座に削除する処理ではありません。保持期間を過ぎたスナップショットや不要なファイルの整理には、後述する`VACUUM`を使用します。また、実行時にはAthenaの処理料金が発生する場合があります。
:::

続いてOPTIMIZEを実行します。

```sql
OPTIMIZE lakehouse_lab.orders
REWRITE DATA USING BIN_PACK;
```

実行は正常に完了しました。

再度`orders$files`を確認すると、データファイルの構成が変わっていました。

| 項目 | OPTIMIZE前 | OPTIMIZE後 |
| --- | ---: | ---: |
| データファイル数 | 2 | 1 |
| データファイル内のレコード数 | 4件 | 2件 |
| データファイルの合計サイズ | 1,588バイト | 760バイト |

![OPTIMIZE後にAthenaのorders filesで確認したデータファイル1件](/images/aws-iceberg-lakehouse/06-files-after-optimize.png)
*OPTIMIZE後。データファイルが1つにまとまり、現在有効な2件を保持している。*

更新・削除による変更を反映した、新しいParquetファイルに再編成されたことが分かります。

今回は最適化後のデータファイルに、現在有効なAliceとBobの2件のみが保存されました。

### 不要ファイルの整理

最後にVACUUMを実行しました。

```sql
VACUUM lakehouse_lab.orders;
```

こちらも正常終了しました。

VACUUMは期限切れスナップショットの整理や、不要になったファイルの削除を行うための処理です。

ただし、正常終了したからといって古いファイルが必ず削除されるわけではありません。Athenaのスナップショット保持期間のデフォルトは5日間で、今回の検証では設定を変更していません。

そのため、実行直後の古いスナップショットも保持されていました。物理的に削除されたS3オブジェクト数は今回確認していません。

メンテナンス後に改めて確認したところ、現在のSELECTではAliceとBobの2件が、INSERT直後のスナップショットを指定したVersion Travelでは3件が取得できました。

**OPTIMIZE・VACUUM実行後も、現在のデータと保持期間内の過去のデータの両方を参照できることが確認できました。**

## まとめ

今回は、Amazon S3、AWS Glue Data Catalog、Amazon Athena、Apache Icebergを使って、小さなデータレイクハウスを構築しました。

実際に試してみて印象的だったのは、Athenaからは普通のSQLでUPDATEやDELETEを実行できる一方、**S3上ではデータファイル、削除情報、スナップショットを分けて管理している**という点です。

S3上のファイルとIcebergのメタデータを確認すると、SQLから見えるテーブルの状態と物理的なファイル構成が異なることもよく分かりました。

今回は最小構成でしたが、Icebergの基本的な動きを理解できたので、次はデータ収集・変換パイプラインや権限管理なども組み合わせ、より実際のデータ基盤に近い構成を検討したいと思います。

### 参考資料

[^iceberg]: [Amazon Athena公式：Apache Icebergテーブルの利用](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg.html)。Athenaで対応しているIcebergの機能や利用上の前提を参照。
[^create]: [Amazon Athena公式：Icebergテーブルの作成](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-creating-tables.html)。`CREATE TABLE`、`TBLPROPERTIES`、パーティション指定について参照。
[^data]: [Amazon Athena公式：Icebergテーブルのデータ更新](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-updating-iceberg-table-data.html)。`INSERT`・`UPDATE`・`DELETE`の操作とPosition Deleteの仕様を参照。
[^time-travel]: [Amazon Athena公式：Time TravelとVersion Travel](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-time-travel-and-version-travel-queries.html)。スナップショットを指定した過去データの参照方法を確認。
[^optimization]: [Amazon Athena公式：Icebergテーブルの最適化](https://docs.aws.amazon.com/athena/latest/ug/querying-iceberg-data-optimization.html)。`OPTIMIZE`や`VACUUM`の仕組みと注意点を参照。
