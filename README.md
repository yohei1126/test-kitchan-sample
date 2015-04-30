# ツールのインストール

## Chef Development Kitのインストール
執筆時点で最新はは 0.4.0
https://downloads.chef.io/chef-dk/
インストール後は eval "$(chef shell-init zsh)" を実行。
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

$ kitchen init --driver=kitchen-vagrant

Kitchen Vagrantがデフォルトのドライバのため、--driver=kitchen-vagrantはなくても良い。

.kitchen.yml を開いてみる。

--
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





