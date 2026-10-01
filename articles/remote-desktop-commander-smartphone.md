---
title: "スマホのChatGPTから自宅PCを操作する。Remote Desktop Commanderを使ってみた"
emoji: "📱"
type: "tech"
topics:
  - "chatgpt"
  - "mcp"
  - "ubuntu"
  - "ai"
published: true
---

最近、**ChatGPT × Remote Desktop Commander × UbuntuミニPC × スマホ**という構成で開発しています。

スマホからChatGPTに話しかけるだけで、自宅のUbuntuミニPC上にあるコードやファイルを確認し、必要ならコマンド実行・修正・テストまで進めてもらう使い方です。

実際に、[公共データ可視化サイト「DATLUME」](https://datlume.com/) と [無料Web便利ツール集](https://utility-tools-jp.com/) も、この構成を使って開発しています。

## Remote Desktop Commanderとは？

Remote Desktop Commanderは、ChatGPTなどのAIから、自分のPC上のファイルやターミナルを扱えるようにする仕組みです。

普通のリモートデスクトップのようにスマホでPC画面を直接操作するのではなく、ChatGPTに「Gitの状態を見て」「テストして」と頼みます。

```mermaid
flowchart LR
    A["📱 スマホ<br/>ChatGPT"] --> B["🤖 ChatGPT"]
    B --> C["🔗 Remote Desktop Commander"]
    C --> D["🖥️ UbuntuミニPC"]
    D --> E["Git / ファイル / ターミナル"]
```

### 無料枠がかなり大きい

執筆時点では、Freeプランが**月10,000 tool calls**、Proが**月20ドルでtool call無制限**です。

最初は無料枠で使っていましたが、使う頻度が増えたので現在はProを契約しています。まず試すだけならFreeでも十分触れます。

料金は変わる可能性があるので、最新情報は[公式のPricingページ](https://desktopcommander.app/pricing/)を確認してください。

<!-- TODO: 料金画面のスクリーンショットを追加 -->

### 外出先からでも使える

スマホとPCが**同じWi-FiやLANにつながっている必要はありません**。それぞれがインターネットにつながっていれば、外出先のスマホからChatGPTに指示を出せます。

```mermaid
flowchart LR
    A["📱 外出先のスマホ"] --> B["💬 ChatGPT"]
    B --> C["🔗 Remote Desktop Commander"]
    C --> D["🏠 自宅のUbuntuミニPC"]
```

たとえば外出中に「開発中のサイトのGitの状態を確認して」と頼み、そのまま差分確認やテストまで進めてもらえます。

## 実際の使い方

使い方はかなりシンプルです。ChatGPTからRemote Desktop Commanderを使える状態にして、あとは普段どおり会話で指示します。

<!-- TODO: ChatGPTからGit確認・テストを頼んでいる実際の画面を追加 -->

```mermaid
flowchart LR
    A["Gitの状態を確認"] --> B["差分を見る"]
    B --> C["必要なら修正"]
    C --> D["テスト"]
    D --> E{"問題あり？"}
    E -- "あり" --> C
    E -- "なし" --> F["完了"]
```

全部を任せる必要はなく、「確認だけ」「原因調査まで」「修正とテストまで」のように区切って使えます。

感覚としては、**スマホのChatGPTから自宅PC上でCodex的な作業を頼む**のに近いです。

### ChatGPT Plusとの組み合わせ

ChatGPT側はPlusを使っています。執筆時点では月20ドルです。

APIのような従量課金ではなく月額プランなので、毎回トークン料金を気にせず使いやすいのが良いところです。ただし、モデルや機能ごとに利用上限が設定される場合はあります。

重めのコード調査ではGPT-5.6 Solを使うことが多いです。モデルや利用条件は変化が速いので、最新情報は[ChatGPT Plusの公式情報](https://help.openai.com/ja-jp/articles/6950777-what-is-chatgpt-plus)を確認してください。

### 常時稼働PCがあれば便利だが注意も必要

現在はUbuntuを入れたミニPCを自宅で常時稼働させています。外出先からでもすぐ使えるので便利ですが、ファイルやターミナルをAIから操作できる分、権限やデータの扱いには注意が必要です。

普段使いのPCと分ける、Gitやバックアップで戻せる状態にする、アクセス範囲を必要以上に広げない、といった対策はしておいた方が安心です。

自宅PCを常時動かしたくないなら、VPSに開発環境を用意する方法もあります。

```mermaid
flowchart TD
    A["Remote Desktop Commanderの接続先"] --> B["普段のPC"]
    A --> C["専用ミニPC"]
    A --> D["VPS"]
    B --> E["手軽 / 個人データに注意"]
    C --> F["環境を分けやすい"]
    D --> G["自宅PC不要 / サーバー管理が必要"]
```

どれが正解というわけではないので、コストやリスクに合わせて選ぶのがよいと思います。

## 実際に作ったもの

### DATLUME

[公共データ可視化サイト「DATLUME」](https://datlume.com/) は、日本の公共データをさまざまな角度から可視化するサイトです。

データ収集や加工、フロントエンド修正、テストなどで、ChatGPTとRemote Desktop Commanderを使っています。

### 無料Web便利ツール集

[無料Web便利ツール集](https://utility-tools-jp.com/) は、CSV/JSON変換など、ブラウザ上で使えるツールをまとめたサイトです。

こちらも機能追加、UI修正、テストなどをスマホから指示して進めることがあります。

<!-- TODO: DATLUMEと無料Web便利ツール集の画面を追加 -->

### この記事もこの構成で作っている

この記事自体も、スマホでChatGPTと話しながら作っています。

```mermaid
flowchart LR
    A["📱 スマホ"] --> B["💬 ChatGPTと相談"]
    B --> C["🖥️ 必要に応じてUbuntu環境を確認"]
    C --> D["📝 記事を修正"]
    D --> B
```

構成を相談し、必要ならPC側を確認し、記事ファイルを修正してGitへ反映するところまで、同じ仕組みを使っています。

## まとめ

Remote Desktop Commanderを使うと、スマホからChatGPTに話しかけて、自分のPC上の開発環境で作業を進められます。

便利なのは、単にPCを遠隔操作できることよりも、**PCの前にいなくても「やりたいこと」をChatGPTに伝えられること**だと感じています。

実際にDATLUMEや無料Web便利ツール集、この記事の制作でも使っています。

一方で、PCのファイルやターミナルを扱える強い仕組みでもあります。普段のPC、専用ミニPC、VPSなど、それぞれに合った環境を選びながら使うのがよさそうです。
