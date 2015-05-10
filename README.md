# Vagrant と Chef による仮想環境構築の自動化 〜 TestKitchenによるテスト入門編〜

※見出しを記述する

## はじめに

本記事のシリーズでは Vagrant と Chef というツールを使い、仮想環境構築を自動化する方法を紹介してきました。環境構築の手順書にあたる Chef のレシピは専用の DSL （Domain Specific Language）を使った Ruby コードです。Chef のレシピをつくる行為は、コードを記述して期待するコンピュータの振る舞いを記述するという点で、通常のソフトウェア開発と同じであると言えます。このため、 Chef のレシピがある程度複雑になってくると、レシピ自身にバグを埋め込んでしまい、期待通りに仮想環境が構築できない現象が発生してしまいます。ここで、期待通りに仮想環境が構築されていないと、リリースされたサービス上で意図しない不具合につながる可能性があります。このため、通常のソフトウェア開発でテストして品質を確認してからリリースするのと同じように、Vagrant と Chef で本番環境を構築してリリースする前にテストで期待通りに仮想環境が構築されているか確認する必要があります。そこで、本記事では Chef のレシピに対してテストを行う方法を紹介します。

## 開発対象のレシピ git-daemon

本記事の目的は Chef レシピのテスト方法をお伝えるすることであるため、テスト対象のレシピそのものは極力簡単なものとします。そこで、本記事では題材として git-daemon （Git サーバ）のインストール用レシピを扱います。

git-daemon の man ページ によると、受入れ基準は以下の通りです。この受入れ基準を満たすように Chef のレシピを開発します。

* プロセスは、Git デーモンのデフォルトポートである 9418 番ポートで待ち受けする。
* サービスの名前は "git-daemon"

## Chef によるレシピ開発で利用するテスト用ツール

Chef Development Kit（Chef の開発環境、以降 Chef DKと省略）には Chef のレシピをテストするためのツールが最初から梱包されています。本記事では Chef-DK に梱包されたツールおよび外部ツールを使い、仮想環境構築をテストする方法を紹介します。

<dl>
  <dt>Test Kitchen</dt>
  <dd>Chef-DK に梱包されている統合テスティングフレームワーク。Chef のレシピをテストする際のフロントエンドになります。Test Kitchen を通じて Vagrant による仮想環境の構築、テストツールを使ったレシピのテストを行うことができます。</dd>
  <dt>Serverspec</dt>
  <dd>元々は Chef とは無関係なツールです。RSpec 風のテストコードで環境構築結果を確認できます。レシピの実行結果が期待通りか確認する結合テストツールとして利用します。テストコードは OS 非依存であるため、1回テストすれば色々な OS に対して利用できます。</dd>
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
 * レシピ開発では Chef DK に梱包されている Chef、Test Kitchen、Berksfhelf を利用 
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

以上で開発環境の構築は完了です。

## Test Kitchen 用設定の準備

ここから実際に前述のテスト駆動開発の流れにそってレシピを開発します。一番最初に Test Kitchen の各種設定を準備する必要があります。

まず、レシピやテストコードを配置するディレクトリを用意してください。

```
% mkdir git-cookbook
% cd git-cookbook
```

以下のコマンドで Test Kitchen の設定ファイルを用意します。

```
% kitchen init
      create  .kitchen.yml
      create  test/integration/default
Fetching: kitchen-vagrant-0.18.0.gem (100%)
Successfully installed kitchen-vagrant-0.18.0
```

標準出力にある通り、以下の2つが作成されています。

* .kitchen.yml
 * Test Kitchen の設定ファイル
* test/integration/default
 * 結合テストのテストコード置き場

さらに、 kitchen-vagrant-0.18.0.gem という Gem がインストールされています。これは Test Kitchen から Vagrant を呼び出し、仮想環境を操作するためのプラグインです。Test Kitchen を使った仮想環境構築のドライバのデフォルトは Vagrant のため、初期化時に kitchen-vagrant がインストールされていない場合は自動的にインストールされます。

作成された .kitchen.yml を開いてみると、内容は以下のようになっています。

```
driver:
  name: vagrant
 
provisioner:
  name: chef_solo
 
platforms:
  - name: ubuntu-12.04
  - name: centos-6.4
 
suites:
   - name: default
     run_list:
     attributes:
```

上から順番に説明していくと、
* driver
 * Test Kitchen が仮想環境構築を委託するドライバの名前です。デフォルトでは vagrant です。
* provisioner
 * Test Kitchen がプロビジョニングで使うツールの名前です。デフォルトは chef_solo です。
* platforms
  * 構築対象のプラットフォームを記述します。デフォルトでは Ubuntu 12.04 と Centos 6.4 が対象になっています。
  * 上記 2 つのプロジェクトの他に、Bento というプロジェクトで提供されている Vagrant 用 box が利用できます。
    * https://github.com/chef/bento
* suites
  * Test Kitchen が構築するテストスイートについて記述します。
  * デフォルトでは default というスイートが用意されており、実行対象のレシピ（run_list）と実行時にレシピに渡される属性（attributes）はありません。

初期状態では実行対象のレシピがないため、default の run_list に以下を追記しておきます。

```
suites:
  - name: default
    run_list:
      - recipe[git::default]
    attributes:
```

