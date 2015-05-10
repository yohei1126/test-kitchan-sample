# Vagrant と Chef による仮想環境構築の自動化 〜Test Kitchen と Serverspec を使ったテスト駆動による仮想環境の構築〜

本記事のシリーズでは Vagrant と Chef というツールを使い、仮想環境構築を自動化する方法を紹介してきました。環境構築の手順書にあたる Chef のレシピは専用の DSL （Domain Specific Language）を使った Ruby コードです。Chef のレシピを作る行為は、期待するコンピュータの振る舞いをコードで記述するという点で、通常のソフトウェア開発と同じであると言えます。このため、 Chef のレシピ開発においても、レシピが複雑になるにつれて、バグを埋め込んでしまい、期待通りに仮想環境が構築できない現象が発生してしまう可能性があります。ここで、期待通りに仮想環境が構築されていないと、リリースされたサービス上で意図しない不具合につながる可能性があります。このため、通常のソフトウェア開発でテストして品質を確認してからリリースするのと同じように、Vagrant と Chef で本番環境を構築してリリースする前にテストで期待通りに仮想環境が構築されているか確認する必要があります。そこで、本記事では Chef のレシピに対してテストを行う方法を紹介します。

## 開発対象のレシピ

本記事の目的は Chef レシピのテスト方法をお伝えるすることであるため、テスト対象のレシピそのものは極力簡単なものとします。そこで、本記事では題材として git-daemon （Git サーバ）のインストール用レシピを扱います。

git-daemon の man ページ によると、受入れ基準は以下の通りです。この受入れ基準を満たすように Chef のレシピを開発します。

* プロセスは、Git デーモンのデフォルトポートである 9418 番ポートで待ち受けする。
* サービスの名前は "git-daemon"。

今回は以下の 2 つの OS をサポートします。

* Ubuntu 12.0.4
* CentOS 6.4

## Chef によるレシピ開発で利用するテスト用ツール

Chef Development Kit（Chef の開発環境、以降 Chef DKと省略）には Chef のレシピをテストするためのツールが最初から梱包されています。本記事では Chef-DK に梱包されたツールおよび外部ツールを使い、仮想環境構築をテストする方法を紹介します。

<dl>
  <dt>Test Kitchen</dt>
  <dd>Chef-DK に梱包されている統合テスティングフレームワーク。Chef のレシピをテストする際のフロントエンドになります。Test Kitchen を通じて Vagrant による仮想環境の構築、テストツールを使ったレシピのテストを行うことができます。</dd>
  <dt>Serverspec</dt>
  <dd>元々は Chef とは無関係なツールです。RSpec 風のテストコードで環境構築結果を確認できます。レシピの実行結果が期待通りか確認する統合テストのツールとして利用します。テストコードは OS 非依存であるため、1 回テストすれば色々な OS に対して利用できます。</dd>
</dl>

## テスト駆動による Chef レシピの開発

本記事ではテスト駆動開発という手法を用いて、Chef のレシピを開発する手順を紹介します。テスト駆動開発では、従来のソフトウェア開発とは異なり、以下のような手順を通して安全に品質の良いレシピを開発していきます。

* 期待するレシピの結果をテストコードで記述する。この時点ではレシピが作成されていないため、テストは失敗する。
* テストが成功するようなレシピを作成し、テストを通す。
* リファクタリング（外部の振る舞いは変更せずに内部の構造を整理する）を行い、テストが通ることを確認する。

得られる効果としては、最初にテストを書くことで、期待する振る舞いを明確に出来る点、テストで保護しながら安全にリファクタリング出来る点が挙げられます。

![テスト駆動開発](http://www.techmatrix.co.jp/quality/concerto/hint/images/hint_agile02_1.jpg "テスト駆動開発")

## 開発環境の準備

ここから実際に Chef レシピ開発に必要な開発環境を用意していきます。本記事では以下のツールを使います。

* Chef DK 0.5.1
 * レシピ開発では Chef DK に梱包されている Chef、Test Kitchen、Berksfhelf を利用 
* Vagrant 1.7.2
* VirtualBox 4.3.26
 * 今回は VirtualBox を使い、ローカルで仮想環境を構築します。

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
  * デフォルトでは default というスイートが用意されており、実行対象のレシピのリスト（run_list）と実行時にレシピへ渡される属性（attributes）はありません。

kitchen list コマンドを実行すると、実際に構築対象の仮想環境が確認できます。プラットフォームごとにテスト用のインスタンスが作成されるため、以下の通り、インスタンスの数は 2 つになります。

```
% kitchen list
Instance             Driver   Provisioner  Verifier  Transport  Last Action
default-ubuntu-1204  Vagrant  ChefSolo     Busser    Ssh        <Not Created>
default-centos-64    Vagrant  ChefSolo     Busser    Ssh        <Not Created>
```

これで Test Kitchen が動作していることが確認できました。実際にテスト駆動でレシピを開発していく前に、さらにいくつか準備が必要です。

プロビジョニングツールの Chef Solo がクックブック自体を認識できるように Chef のお約束事として metadata.rb を用意し、クックブックの名前とバージョンを書いておきます。

```
% vim metadata.rb
name "git"
version "0.1.0"
```

さらに、初期状態では実行対象のレシピがないため、default の run_list にレシピ default を追記しておきます。

```
suites:
  - name: default
    run_list:
      - recipe[git::default]
    attributes:
```

## serverspec によるテストコードを作成する

前述のテスト駆動開発の手順の通り、最初にテストコードを記述することで、期待する振る舞いを明確に定義します。

```
% mkdir -p test/integration/server/serverspec
% vim test/integration/server/serverspec/git_daemon_spec.rb
require 'serverspec'

# Required by serverspec
set :backend, :exec

describe "Git Daemon" do

  it "is listening on port 9418" do
    expect(port(9418)).to be_listening
  end

  it "has a running service of git-daemon" do
    expect(service("git-daemon")).to be_running
  end

end
```

テストコードでは、前述の要件 2 点について確認しています。

* プロセスは、Git デーモンのデフォルトポートである 9418 番ポートで待ち受けする。
* サービスの名前は "git-daemon"。

最初に default-ubuntu-1204 に対して、kitchen verify コマンドでテストを実行してみましょう。ここでは、コマンドの引数としてインスタンス名を省略して ubuntu を指定しています。インスタンス名は候補とパターンマッチングされるため、厳密にインスタンス名を書く必要はなく、他のインスタンス名と区別さえできれば名前の一部だけでも OK です。

```
kitchen verify ubuntu
（省略）
Vagrant instance <default-ubuntu-1204> created.
Finished creating <default-ubuntu-1204> (1m25.60s).
（省略）
Compiling Cookbooks...
       
       ================================================================================
       Recipe Compile Error
       ================================================================================
       
       Chef::Exceptions::RecipeNotFound
       --------------------------------
       could not find recipe default for cookbook git
```

インスタンスは作成されましたが、予想通り、git というクックブックの default レシピがないというエラーが出ています。kitchen list コマンドを実行すると、実際に default-ubuntu-1204 の「Last Action」が「Created」になっていることを確認できます。

```
% kitchen list                                                                                                    Instance             Driver   Provisioner  Verifier  Transport  Last Action
default-ubuntu-1204  Vagrant  ChefSolo     Busser    Ssh        Created
default-centos-64    Vagrant  ChefSolo     Busser    Ssh        <Not Created>
```

## テストが成功するレシピを作成する

次に実際にテストコードが成功するレシピを実際に作成します。.kitchen.yml に書いた通り、default という名前でレシピを作成します。

% mkdir recipes
% vim recipes/default.rb
```
include_recipe "runit"

package "git-daemon-run"

runit_service "git-daemon" do
  sv_templates false
end
```

レシピの内容は以下の通りです。
* runit クックブックをインストールしています。runit はデーモンを起動するためのツールです。
* git-daemon-run パッケージをインストールしています。こちらは git のデーモン用パッケージです。
* 最後に "git-daemon" という runit service を宣言しています。これでデーモンを起動できます。

次に実際にテストしてみると、この段階では、ローカルのリポジトリに runit クックブックが存在しないため、エラーが出ます。

```
% kitchen verify ubuntu
（省略）
       ================================================================================
       Recipe Compile Error in /tmp/kitchen/cookbooks/git/recipes/default.rb
       ================================================================================
       
       Chef::Exceptions::CookbookNotFound
       ----------------------------------
       Cookbook runit not found. If you're loading runit from another cookbook, make sure you configure the dependency in your metadata
       
       Cookbook Trace:
       ---------------
         /tmp/kitchen/cookbooks/git/recipes/default.rb:1:in `from_file'
       
       Relevant File Content:
       ----------------------
       /tmp/kitchen/cookbooks/git/recipes/default.rb:
       
         1>> include_recipe "runit"
         2:  
         3:  package "git-daemon-run"
         4:  
         5:  runit_service "git-daemon" do
         6:    sv_templates false
         7:  end
         8:  
（省略）
```

以前に書いた記事の通り、Chef ではクックブックの依存関係を自動的に解決するツールとして Berkshelf を利用します。以下のように Berkshelf の設定ファイル Berksfile を用意します。

ここでは以下のことを Berkshelf に指示しています。
* 依存先のクックブックを公開リポジトリである https://supermarket.chef.io metadata.rb から取得する。
* クックブックの依存関係を metadata.rb に記述する。

```
% vim Berksfile
source "https://supermarket.chef.io"

metadata
```

次に metadata.rb に git クックブックの依存関係を記述します。runit クックブックのバージョンは検証できている 1.4.0 以上と指定しました。

```
% vim metadata.rb
name "git"
version "0.1.0"

depends "runit", "~> 1.4.0"
```

次にテストを実行するとテストが成功する旨が表示されます。

```
% kitchen verify ubuntu
（省略）
Finished verifying <default-ubuntu-1204> 
（省略）
```

次に CentOS についてもテストが成功するか確認してみましょう。

```
% kitchen verify centos
（省略）
Chef::Exceptions::Package
           -------------------------
           No candidate version available for git-daemon-run
（省略）
```

どうやら Ubuntu と CentOS は git-daemon のパッケージ名が異なるため、同じパッケージ名で両方の OS に対してパッケージをインストールできないようです。以下のようにレシピ中でプラットフォームごとに適切なパッケージ名を使うように変更しましょう。

```
include_recipe "runit"
 
package "git-daemon" do
  case node[:platform]
  when "centos"
    package_name "git-daemon"
  when "ubuntu"
    package_name "git-daemon-run"
  end
  action :install
end
 
runit_service "git-daemon" do
  sv_templates false
end
```

テストを再度実行すると今度は問題ないことが確認できました。

```
% kitchen verify centos
```

## リファクタリングでレシピ中の条件分岐を排除する

前述の変更でテストが成功するようになりましたが、レシピ中にプラットフォームごとの条件分岐を行うやり方はあまり良い方法ではありません。現在、条件分岐は 1 箇所ですが、今後増えていくとレシピの保守性が悪くなっていきます。

そこで Chef の attribute （変数）という機能を使ってパッケージ名を外部設定ファイルから与えられるようにします。

まず以下のように attributes/default.rb というファイルを記述し、プラットフォームごとのパッケージを表す attributes の配列を用意します。今後はプラットフォームが増えていけば、attributes 配列を増やしていくことになります。

```
% mkdir attributes
% vim attributes/default.rb
default["gitdaemon"]["centos"] = "git-daemon"
default["gitdaemon"]["ubuntu"] = "git-daemon-run"
```

レシピ側では、例えば、node["gitdaemon"]["centos"]という書式で CentOS 上での git-daemon のパッケージ名にアクセスできます。プラットフォーム名の文字列は、デフォルトで node[:platform] で取得できるため、これを利用することによりプラットフォームごとの条件分岐を排除できます。

```
include_recipe "runit"
 
package node["gitdaemon"][node[:platform]]
 
runit_service "git-daemon" do
  sv_templates false
end
```

今後プラットフォームごとに値が異なる変数が増えた場合でも、レシピ中は変数で対処し、レシピの外部から attribute などの機能を使ってプラットフォームごとの値を変数に与えることができます。

以上でリファクタリングは終了です。一旦、作成済みのインスタンスを破棄し、作り直した上でレシピのテストを実行してみましょう。kitchen test コマンドを使うと、各インスタンスに対してインスタンスの破棄、レシピ実行、テスト実行を行ってくれます。

```
% kitchen test
```

無事リファクタリングができたことを確認したところで、今回のクックブックは完了です。

## おわりに

今回は、Test Kitchen と Serverspec というツールを使い、Chef のレシピをテスト駆動で開発する手順を紹介しました。テスト駆動で開発することで、レシピを書く前にレシピの要件を明確にでき、さらに機能拡張の際もテストコードがあることで安全にレシピをリファクタリング出来る点を示しました。

今回の題材として取り上げたレシピは、非常に小さいため、テストやリファクタリングをするありがたみは大きくありません。しかし、開発環境に必要なクックブックが大きく複雑になると、通常のソフトウェアと一緒で、開発が困難になります。その際、対象のクックブックを一気につくり上げるのではなく、小さい単位に小分けして、それぞれテストしながら順番に組み上げていくと、最初の段階から期待通りに動いていることを保証しながら、徐々に機能を増やし、クックブック群を大きく育てていくことができます。

## 次回予告

これまでは Vagrant と Chef のツールそのものの使い方に焦点を挙げてきましたが、次回からは AWS 上でプロビジョニングを効率的に行う方法を紹介していきます。
