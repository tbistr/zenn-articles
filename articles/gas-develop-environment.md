---
title: "Google Apps Script のローカル開発環境を整える"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gas", "devcontainer", "vscode", "nodejs", "typescript"]
published: false
---

## 概要

この記事では、Google Apps Script(GAS)のローカル開発環境について提案しています。
作成した環境は[GitHubのリポジトリ](https://github.com/tbistr/gas-devcontainer)
にテンプレートリポジトリとして公開されています。

GASのチュートリアルではないので、GAS自体の説明は省略します。
GASに触ったことのある方を対象読者としています。

エディタはVSCodeを前提としていますが、ほとんどの機能は他のエディタでも利用できると思います。

## 要件

GASはWebエディタをサポートしているため、ローカルにコードを置くことなく開発できます。
また、JavaScriptに加えてTypeScriptもサポートしており、TypeScriptのコードをそのまま実行できます。
しかし、好きなエディタを使いたい/コードをGitで管理したい場合、ローカルにコードを置いてGASにプッシュする必要があります。

いつも使っている環境と合わせて、要件を以下のように整理しました。

- ローカルで書いたコードをGASにプッシュできる
- TypeScript対応
- Linter, Formatterの自動実行
- GASの型が扱える

## 環境の概要

上記の要件から以下のような構成を考えました。

| 用途                  | ツール名 | 備考                            |
| --------------------- | -------- | ------------------------------- |
| GAS連携ツール         | clasp    | https://github.com/google/clasp |
| ビルド/バンドルツール | esbuild  | https://esbuild.github.io/      |
| Formatter             | prettier | https://prettier.io/            |
| Linter                | ESLint   | https://eslint.org/             |

Linter, Formatterについては、普段の開発環境と同じものの流用になるため説明しません。


## GAS連携ツールの選定

### clasp

### aside

## ビルドツール

## 課題

### claspの認証

### ビルドの必要性
