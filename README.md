# Zenn コンテンツ管理リポジトリ

このリポジトリは、Zennの記事や本をローカルで管理・執筆するためのワークスペースです。

## 初期セットアップ

新しく環境を構築する場合（別のPCでクローンした時など）は、最初に依存パッケージをインストールしてください。

```bash
npm install
```

## 基本的な使い方

Zenn CLIを使用して、記事の作成やプレビューを行います。コマンドはすべてこのディレクトリ（`package.json` がある場所）で実行してください。

### ローカルプレビューの起動

```bash
npx zenn preview
```
実行後、ブラウザで `http://localhost:8000` にアクセスすると、執筆中の記事や本をリアルタイムでプレビューできます。

### 新規記事の作成

```bash
npx zenn new:article
```
`articles` ディレクトリに新しいMarkdownファイルが生成されます。

### 新規本の作成

```bash
npx zenn new:book
```
`books` ディレクトリに新しい本のディレクトリが生成されます。

## 参考リンク
* [📘 Zenn CLI ガイド (公式)](https://zenn.dev/zenn/articles/zenn-cli-guide)