# ツールのインストール

## Chef Development Kitのインストール
執筆時点で最新はは 0.5.1
https://downloads.chef.io/chef-dk/
**インストール後に eval "$(chef shell-init zsh)" をやらない。これをやるとkitchen-vagrantが見つからなくなる。
Chef DK に付属のRubyを使う。
which ruby で確認。
/opt/chefdk/embedded/bin/ruby
この中にTest Kitchenも含まれている。
% kitchen -v
Test Kitchen version 1.3.1

## Vagrantのインストール
執筆時点で最新は 1.7.2。
https://www.vagrantup.com/downloads.html

## VirtualBoxのインストール
執筆時点で最新は 4.3.26
https://www.virtualbox.org/v

#クックブックの作成

$ git init git-cookbook
Initialized empty Git repository in /tmpt-cookbook/git-cookbook/.git/
$ cd git-cookbook
$ vim metadata.rb
name "git"
version "0.1.0"
$ mkdir recipes
$ touch recipes/default.rb

test kitchen用のファイルを初期化する。

$ kitchen init

.kitchen.yml を開いてみる。
centos 6.4は利用できない風のため、6.6に変更。

--
driver:
  name: vagrant

provisioner:
  name: chef_solo

platforms:
  - name: ubuntu-12.04
  - name: centos-6.6

suites:
  - name: default
    run_list:
      - recipe[git::default]
    attributes:

* driver Kitchenドライバの設定。テストに使うマシンを作成する。
　credentials, ssh usernames, sudo requirements, etc.なども設定する。
* provisioner:
  テスト対象のマシンに対してChefを実行するプロビジョナの設定。
* platforms: 
  コードを実行するOSのリスト。
* suites:
  何をテストするかを記述する。Chefのrun-listがある。attributesは設定していない。

kitchen list で作成可能なインスタンスのリストが表示される。

% kitchen list
Instance             Driver   Provisioner  Last Action
default-ubuntu-1204  Vagrant  ChefSolo     <Not Created>
default-centos-64    Vagrant  ChefSolo     <Not Created>

# インスタンスの作成

$ kitchen create default-ubuntu-1204

インスタンス名は全て指定しなくてもよい。今回は1204だけでも良い。
作成後にlistを表示すると、作成できたことが分かる。

% kitchen list
Instance             Driver   Provisioner  Last Action
default-ubuntu-1204  Vagrant  ChefSolo     Created
default-centos-64    Vagrant  ChefSolo     <Not Created>

# レシピの作成

% vim recipes/default.rb

package "git"

log "Well, that was too easy"

# kitchen convergeの実行
$ kitchen converge default-ubuntu-1204

% kitchen list
convergedになっている
Instance             Driver   Provisioner  Verifier  Transport  Last Action
default-ubuntu-1204  Vagrant  ChefSolo     Busser    Ssh        Converged

# ログインして結果を確認する

$ kitchen login default-ubuntu-1204
vagrant@default-ubuntu-1204:~$ which git
/usr/bin/git
vagrant@default-ubuntu-1204:~$ git --version
git version 1.7.9.5
vagrant@default-ubuntu-1204:~$ 

#最初のテストはbatsで書いてみる。
batsは手軽だが、プラットフォーム依存になるので、良くない。
test/integration/default/bats/git_installed.bats with the following:

#!/usr/bin/env bats

@test "git binary is found in PATH" {
  run which git
  [ "$status" -eq 0 ]
}

# Verify する

kitchen verify <platform>

を実行する。verifyを実行するごとに、テストをアップロードすること。
テストを変更して、再度、verifyすると、即座に新しいテストでverifyされる。

# Test する

・すでにインスタンスが作成済みの場合、インスタンスを一度破棄し、作りなおす。
・インスタンスを起動
・クックブックをアップロードし、実行する
・テストを実行し、期待通りの結果になっているか確認する

Destroys the instance if it exists (Cleaning up any prior instances of <default-ubuntu-1204>)
Creates the instance (Creating <default-ubuntu-1204>)
Converges the instance (Converging <default-ubuntu-1204>)
Sets up Busser and runner plugins on the instance (Setting up <default-ubuntu-1204>)
Verifies the instance by running Busser tests (Verifying <default-ubuntu-1204>)
Destroys the instance (Destroying <default-ubuntu-1204>)

# サーバのテスト
git のデーモンのテストを行う。
要件は
A process is listening on port 9418 (the default Git daemon port).
A service is running called "git-daemon". This name is arbitrary but shouldn't surprise anyone using this cookbook.

# suiteの追加

.kitchen.yml に以下を追記する。

 - name: server
    run_list:
      - recipe[git::server]
    attributes:

kitchen listをするとインスタンスが2つ増えていることを確認する。

# Serverのテストを書く
bats テスティングフレームワークは手軽であるものの、プラットフォーム依存なので、テストとしては良くない。
ディストリビューションごとにテスト方法を変える必要がある。
そこでここでは Serverspec というテスティングフレームワークを利用する。
serverspecはプラットフォーム非依存なテストをかける。RSpecのマッチャーの集合。

test/integration/server/serverspec/git_daemon_spec.rb
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

ひとまずverifyをかけて失敗することを確認する。
次にテストが通りはずのレシピを書く。

recipes/server.rb
include_recipe "git"
include_recipe "runit"

package "git-daemon-run"

runit_service "git-daemon" do
  sv_templates false
end

We include our default Git recipe so that Git is installed. Hey we already solved that one, right?
We include the default recipe from the runit cookbook which installs runit and provides the runit_service resource.
We install the git-daemon-run package which gives us the git-daemon program, on Ubuntu 12.04 at least.
Finally, we declare a runit service called git-daemon without generating the run and log scripts (they were provided by the Ubuntu package).

これではrunitのレシピがないため、テストが失敗する。

metadata.rbに依存関係を追記。これでレシピを実行する前にrunitが実行される。
depends "runit", "~> 1.4.0"

依存関係の解決にはBerkshelfというツールを使う。これはChef-DKに付いているので、あらためてインストールは不要。
Berkshelfを使うにはBerksfileというファイルを用意する必要あり。

source "https://api.berkshelf.com"

metadata

このファイルでは、metadata.rbで依存関係の解決を行う旨を伝えている。

これでverifyを行うと、テストが通ったことが分かる。




