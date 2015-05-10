# Vagrant と Chef による仮想環境構築の自動化 〜 TestKitchenによるテスト入門編〜

※見出しを記述する

## はじめに

本記事のシリーズでは Vagrant と Chef というツールを使い、仮想環境構築を自動化する方法を紹介してきました。環境構築の手順書にあたる Chef のレシピは専用の DSL （Domain Specific Language）を使った Ruby コードです。Chef のレシピをつくる行為は、コードを記述して期待するコンピュータの振る舞いを記述するという点で、通常のソフトウェア開発と同じであると言えます。このため、 Chef のレシピがある程度複雑になってくると、レシピ自身にバグを埋め込んでしまい、期待通りに仮想環境が構築できない現象が発生してしまいます。ここで、期待通りに仮想環境が構築されていないと、リリースされたサービス上で意図しない不具合につながる可能性があります。このため、通常のソフトウェア開発でテストして品質を確認してからリリースするのと同じように、Vagrant と Chef で本番環境を構築してリリースする前にテストで期待通りに仮想環境が構築されているか確認する必要があります。そこで、本記事では Chef のレシピに対してテストを行う方法を紹介します。

## 開発対象のレシピ git-daemon

本記事の目的は Chef レシピのテスト方法をお伝えるすることであるため、テスト対象のレシピそのものは極力簡単なものとします。そこで、本記事では題材として git-daemon （Git サーバ）のインストール用レシピを扱います。

git-daemon の [man](ftp://www.kernel.org/pub/software/scm/git/docs/git-daemon.html) ページ によると、受入れ基準は以下の通りです。この受入れ基準を満たすように Chef のレシピを開発します。

* プロセスは、Git デーモンのデフォルトポートである 9418 番ポートで待ち受けする。
* サービスの名前は "git-daemon"

## Chef によるレシピ開発で利用するテスト用ツール

Chef Development Kit（Chef の開発環境、以降 Chef DKと省略）には Chef のレシピをテストするためのツールが最初から梱包されています。本記事では Chef-DK に梱包されたツールおよび外部ツールを使い、仮想環境構築をテストする方法を紹介します。

<dl>
  <dt>Test Kitchen</dt>
  <dd>Chef-DK に梱包されている統合テスティングフレームワーク。Chef のレシピをテストする際のフロントエンドになります。Test Kitchen を通じて Vagrant による仮想環境の構築、テストツールを使ったレシピのテストを行うことができます。</dd>
  <dt>Serverspec</dt>
  <dd>元々は Chef とは無関係なツールです。RSpec 風のテストコードで環境構築結果を確認できるため、レシピの実行結果が期待通りか確認する結合テストツールとして利用します。</dd>
</dl>

## テスト駆動による Chef レシピの開発

本記事ではテスト駆動開発という手法を用いて、Chef のレシピを開発する手順を紹介します。テスト駆動開発では、従来のソフトウェア開発とは異なり、以下のような手順を通して安全に品質の良いレシピを開発していきます。

* 期待するレシピの結果テストコードで記述する。この時点ではレシピが作成されていないため、テストは失敗する。
* テストが成功するようなレシピを作成し、テストを通す。
* リファクタリング（外部の振る舞いは変更せずに内部の構造を整理する）を行い、テストが通ることを確認する。

![テスト駆動開発](http://www.techmatrix.co.jp/quality/concerto/hint/images/hint_agile02_1.jpg "テスト駆動開発")

## 開発環境の準備

ここから実際に Chef レシピ開発に必要な開発環境を用意していきます。本記事では以下のツールを使います。

* Chef DK 0.5.1
 * レシピ開発では Chef DK に梱包されている Chef、TestKitchen、Berksfhelf を利用 
* Vagrant 1.7.2
* VirtualBox 4.3.26

### Chef DK のインストール

以下の URL からインストーラをダウンロードして、インストールを行って下さい。2015 年 5 月時点の最新バージョンでは 0.5.1 です。

https://downloads.chef.io/chef-dk/

Chef のレシピ開発では、Chef DK に梱包されている Ruby に PATH を通す必要があります。以下のコマンドを使うと Chef を使うための初期設定が行われます。筆者の場合、シェルは zsh を使っていますが、別のシェルを使っている場合はシェル名部分を変更してください。

```
eval "$(chef shell-init zsh)"
```

### Vagrant のインストール

以下の URL からインストーラをダウンロードして、実行してください。2015 年 5 月時点の最新バージョンは 1.7.2 です。

https://www.vagrantup.com/downloads.html

### VirtualBox のインストール

以下の URL からインストーラをダウンロードして、実行してください。2015 年 5 月時点の最新バージョンは 4.3.26 です。

https://www.virtualbox.org/

