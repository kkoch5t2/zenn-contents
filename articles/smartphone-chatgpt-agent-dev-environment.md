---
title: "スマホでバイブコーディング"
emoji: "📱"
type: "idea"
topics: ["AIエージェント", "chatgpt", "mcp", "個人開発", "ubuntu"]
published: true
---

最近の個人開発は、ほとんどスマホのChatGPTに指示して進めています。Androidアプリを開発するだけでなく、Webサイトの更新や運営も同じ環境で行っています。

コードを書く、テストする、ビルドするといった作業を引き受けるのは、自宅に置いたUbuntuのミニPCです。自分はスマホから要望を伝え、報告と実際の画面を確認して、気になるところを追加で修正してもらいます。

この記事では、**環境の全体像 → 各機能の仕組み → 日々の運用**の順に、自分がどう使っているかを紹介します。

## 開発環境の全体像

基本的な流れは、スマホからChatGPTに依頼し、必要なら自宅のPCを起動して、作業が終わったらシャットダウンするというものです。ChatGPTからPCを操作する際は、自作のMCPサーバー「redeco」を使います（後ほど説明します）。

![スマホでの依頼からプラグON、ミニPC起動、redecoでの開発、シャットダウンまでの流れ](/images/smartphone-agent-dev-flow-20261030.webp)

この仕組みを支えるのは、主に次の4つです。

| 役割 | 使っているもの | 何をするか |
| --- | --- | --- |
| 指示・確認 | スマホのChatGPT | 開発や運用を依頼し、作業結果を受け取る |
| PC操作 | 自宅ミニPC上のredeco | ファイル編集、コマンド実行、テスト、ビルド |
| PCとの接続 | OpenAI Secure MCP Tunnel | ChatGPTと自宅PCの間で操作依頼・結果を受け渡す |
| 電源操作 | Cloudflare Workers上のMCPとSwitchBot | PCが停止していてもスマートプラグを操作する |

ここで大事なのは、**ChatGPTがPCを操作する経路と、自分ができあがった画面を確認する経路は別**という点です。

- **Webサイトの確認**：変更を公開先へデプロイしたあと、外出先でもスマホのブラウザから公開URLを開いて確認できます。
- **Androidアプリの確認**：ミニPCでビルドしたAPKを自宅でADB経由で実機に入れ、動作を確認します。

つまり、この環境はスマホから自宅PCの画面を遠隔操作・配信する仕組みではなく、ChatGPTを介してPC上の作業を進める仕組みです。

## 各機能の仕組み

### redeco：ChatGPTからPCのファイルやコマンドを操作する

redecoは、ChatGPTから自分のPCを操作するために作ったMCPサーバーです。正式名称は「redeco」で、Remote Development Connector（リモートデベロップメントコネクター）に由来します。

https://github.com/kkoch5t2/redeco

MCPは、AIが外部のツールを呼び出すための仕組みです。redecoはファイルの読み書き、コマンド実行、長時間処理の開始・出力確認・停止などに対応しています。ChatGPTがこれらの機能を使い、ソースコードの修正からテスト、ビルドまで進めます。

例えば、スマホから「アプリのセキュリティチェックをして」と依頼すると、ChatGPTがredecoを通じてミニPC上のコードを調べ、結果を報告します。作業の流れは次のとおりです。

![スマホからセキュリティチェックを依頼し、redeco経由でUbuntuのコードやテストを調査して結果を報告する流れ](/images/redeco-security-check-flow-20261030.webp)

こちらは、Routiaの公開前セキュリティチェックをChatGPT Workに依頼したときの実際の画面です。赤枠の部分に、redeco経由で実行した操作の履歴が表示されています。

![ChatGPT WorkでRoutiaの公開前セキュリティチェックを行った実際の操作履歴。赤枠はredecoによるコマンド実行と検証](/images/redeco-security-check-screenshot-20261030.webp)

### Secure MCP Tunnel：ChatGPTと自宅PCをつなぐ

ChatGPTとredecoの間は、OpenAIのSecure MCP Tunnelで接続しています。自宅PCの`tunnel-client`がOpenAI側へHTTPS接続を開始し、受け取った操作依頼を同じPC上のredecoへ渡します。[^tunnel]

PCからOpenAI側へ接続する経路を図にまとめました。

![ChatGPTと自宅PCをSecure MCP Tunnelでつなぐ模式図。PC側のtunnel-clientがHTTPS接続を開始し、redecoとローカルで連携する](/images/secure-mcp-tunnel-flow-20261030.webp)

接続を始めるのは自宅PC側なので、redecoのためにルーターのポートをインターネットへ開放する必要はありません。スマホが自宅と同じLANに接続されている必要もありません。

通信経路の詳しい話は、別の記事にまとめています。

https://zenn.dev/kkoch5t2/articles/openai-secure-mcp-tunnel

### 電源操作用MCP：PCが停止していても起動できる

PCの電源にはSwitchBotのスマートプラグを使い、ChatGPTから呼び出す電源操作用MCPをCloudflare Workers上で動かしています。WorkerがSwitchBot APIを呼び出すため、**PC本体が停止していてもプラグのONを依頼できます。**

プラグのONからPC起動までの流れは次のようになります。

![ChatGPTから電源操作用MCPとSwitchBotを経由して、停止中の自宅ミニPCを起動する流れの模式図](/images/switchbot-power-flow-20261030.webp)

https://github.com/kkoch5t2/switchbot-control

ミニPC側では、通電すると自動起動するようBIOS／UEFIを設定しています。OSが起動すると、redecoと`tunnel-client`がsystemdから起動します。なお、通電時の自動起動に対応しているか、どこで設定するかはPCによって異なります。

Cloudflare Workersが行うのは、電源のAPI連携です。コードの修正、ビルド、テストは自宅のミニPCで実行します。

### 権限とセキュリティ：便利な一方で、できる操作は強い

redecoの専用ファイルツールには、操作できるフォルダーの制限、`.env`や`.ssh`などへのアクセス拒否、既存ファイルを置換するときのSHA-256確認があります。[^redeco] これらは専用ファイルツールに対する制限であり、シェルからのコマンド実行にはそのまま適用されません。

コマンドはredecoを動かしているOSユーザーの権限で実行されます。つまり、ファイルツールで拒否したファイルでも、OS上で読み取り権限があればシェル経由でアクセスできます。OpenAI Secure MCP Tunnelは通信経路を提供する仕組みで、コマンドの実行権限までは制限しません。redeco自体にもMCPの認証はなく、ローカルの待ち受け先に到達できるプロセスから呼び出せる構成です。

自分の環境では、redecoを開発用の通常ユーザーとして動かし、sudoも使える設定にしています。専用コンテナやVMによる分離は行っていません。redecoには専用ユーザーを使う隔離モードもありますが、現在使っているのは開発ユーザーモードです。[^redeco-ubuntu]

この構成で気をつけたいのは、ファイルの誤削除、認証情報の外部送信、調査先のページやリポジトリに含まれる文章をAIが操作指示として受け取るプロンプトインジェクションです。Gitで復旧できる変更にも限界があります。また、ChatGPTによるセキュリティチェックの結果は参考になりますが、アプリ全体の安全性を証明するものではありません。

今後はAI用の実行環境と認証情報を分け、sudoを使う操作を必要な範囲に絞りたいと考えています。

## 日々の運用

### PCを起動して、修正を依頼する

PCが停止していたら、スマホのChatGPTからスマートプラグのONを依頼します。PCと必要なサービスが起動し、redeco経由で接続先や状態を確認できてから作業を始めます。

依頼するときは、どの画面の何を直したいのかだけでなく、変えてほしくない部分も伝えるようにしています。例えば、こんな感じです。

> アプリの成果画面を修正して。習慣名が先に分かる表示にしたい。記録データの保存方式は変えず、表示部分を対象にして。変更理由とファイル、実行したテスト、実機でまだ確認が必要なことを報告して。

Webサイトの修正なら、対象の公開先、デプロイ後に確認するURL、既存ページやデータを壊していないかも確認条件に加えます。

使っていて意外と難しかったのが、**チャットのセッション管理**です。同じ会話で次々と別の開発を続けると、会話が重くなり、どの変更がどの依頼によるものかも追いづらくなります。そのため、今は基本的に「1チャット1タスク」にしています。会話を切り替えるときは、対象プロジェクト、変更内容、実行したテスト、未確認事項を引き継ぎます。

### 結果を確認し、必要なら修正し直す

ChatGPTから「修正しました」と報告が来ても、それだけで完了にはしません。変更したファイルやテスト結果を確認し、実際に画面を触ってから判断します。

Webサイトなら公開先にデプロイされた版をスマホのブラウザで確認できます。

Androidアプリは、ミニPCでビルドできても、普段の実機確認には自宅でADB経由のインストールが必要です。外出先では導入済みの版を使って問題点を整理し、修正やビルドを指示するところまで進めています。

画面の見た目だけでなく、元の不具合が出た操作と周辺の操作が壊れていないかを確認します。特にデータ保存、認証、削除、公開設定に関わる変更では、AIの説明だけに頼らず差分や実行結果も確認するようにしています。

### 作業を終了する

終了時は、実行中の処理を確認してからOSをシャットダウンします。PCが停止するとredecoや`tunnel-client`との通信も切れるため、停止状態の確認には別の根拠が必要です。

電源操作用MCPでは、PC用プラグのOFFに`shutdown_confirmed: true`を要求しています。ただし、この値は呼び出し側の申告であり、WorkerがOSの停止を自動検知する機能はまだありません。[^power] そのため、正常終了を確認できた場合にプラグをOFFにし、確認できなければ保留する運用です。

### 運用にかかる費用

自分の使い方で主にかかっているのは、ChatGPT Plusの月額料金と電気代です。ミニPCとスマートプラグは初期費用として購入しました。

電源操作用MCPはCloudflare Workersの無料枠で動かしています。無料枠の条件は公式の料金ページを参照してください。なお、Webサイトの公開先などで別途有料サービスを使う場合は、その費用もかかります。

https://developers.cloudflare.com/workers/platform/pricing/

ChatGPT Plusでも、個人開発には思っていた以上に使えています。自宅のPCに実作業を任せ、**スマホから依頼と確認を繰り返せること**は、アプリ開発とWebサイト運営の両方で大きなメリットになっています。

## 参考資料

[^tunnel]: [OpenAI公式：Secure MCP Tunnel](https://developers.openai.com/api/docs/guides/secure-mcp-tunnels)。PC側から接続する通信経路について参照。
[^redeco]: [redeco README](https://github.com/kkoch5t2/redeco/blob/main/README.md)と[ツールの実装](https://github.com/kkoch5t2/redeco/blob/main/server.py)。ファイル制限、SHA-256確認、コマンド実行の境界を参照。2026年10月30日確認。
[^redeco-ubuntu]: [redeco Ubuntu版の導入・権限設定](https://github.com/kkoch5t2/redeco/blob/main/ubuntu/README.md)。隔離モードと開発ユーザーモード、sudoの設定、サービス名と実行モードの違いを参照。
[^power]: [SwitchBot ControlのPC電源管理](https://github.com/kkoch5t2/switchbot-control/blob/main/README.md#pcの電源管理)と[実装](https://github.com/kkoch5t2/switchbot-control/blob/main/src.mjs)。`shutdown_confirmed`は呼び出し側の確認申告。2026年10月30日確認。