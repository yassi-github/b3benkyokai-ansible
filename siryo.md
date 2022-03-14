# Ansible使おう！

Ansibleはいいぞ

資料のダウンロード  
用意していたLinuxマシンに入れておいてください。

```
git clone 
```

## Ansibleとは？

構成管理ツール。ミドルウェアのインストールや設定などなどを自動で行う。

- VMをどかどか建てる場合、手動でミドルウェアをインストールしたり設定を変更したりするのは時間と労力の浪費が激しい。
- クラウドでは、VMを作成しては消すことが想定される
    - モノリスのような構成では、ソフトウェア間の依存関係の問題(一方の設定がもう一方の設定とコンフリクトするなど)が発生しやすいし、なにより汚い。
        - 例: 授業で使っているPC
    - サーバを増やすなどの場合、構成のコピーをすることが考えられる
        - VMのコピーができるのは限られた場合のみ(センシティブな情報の扱い(別の顧客のVMをコピーしたろ！←だめ)や、そもVMのコピー機能が使えるのかなど)
- 手動だとヒューマンエラーは避けられない
- installationなど、やり方が決まっているなら自動化できるはず

-> よって、何回も行う処理やミスしたくない処理などを、自動化しよう！という考え。

## 類似するソフトウェアとの比較

||ベース言語|構築設定ファイル|クライアントへの操作|GitHub Star|
|-|-|-|-|-|
|Ansible|Python|Playbook(YAML)|SSH Connection|52.3k|
|Chef|Ruby|Recipe(Ruby like DSL)|Install Agent|6.8k|
|Puppet|Ruby|manifest(Ruby like DSL)|Install Agent|6.5k|

AnsibleはクライアントにAgentを入れなくても使える！  
構成の設定ファイルはRubyを知らなくても書ける！

## YAML?

- 読みやすい
- 書きやすい
- 分かりやすい

データシリアライズ言語。JSONみたいなやつ。

ちょっとｋｗｓｋ: https://circleci.com/ja/blog/what-is-yaml-a-beginner-s-guide/

## Ansibleの哲学

### **冪等性**

同じ操作を複数回行っても、結果は同じになる性質。  
例: エレベーターのボタン

タスクを複数回行っても、記述されたあるべき状態になるだけ。  
例: ユーザーを作成するタスクを何回行っても、ユーザーがすでに作成されているならば何もしない(新たに作成しようとしない)

### **Infrastructure as Code (IaC)**

サーバの構成をコード化しようという考え。

今までは、構成を変更する場合、サーバにログインし変更を加え、何かしらの文書に変更を追記、などとしてきた(と思われる)。  
その場合、サーバの状態を知るためには、難解な自然言語で書かれた文書を読む必要があり、  
しかも、変更の追記し忘れやtypoなどが十分考えられるため、文書に書いてあることが正しいとは限らない。  
サーバを増やす、あるいはリニューアルするなどのときは、その文書を読み、書いてあることは正しいか確認し、書いていないことはないかサーバの状態を調べ…そして、(おそらく適切と思われる)操作を行う。  
と、かなり大変な作業となる。文書がないなどの場合は本当に大変。まじで。

そこで、構成をコード化しそれを元に変更を適用する。

これにより、

1. コードが全てであるため、サーバの構成はどうなっているかが明確になる
1. 構成の変更や再適用を容易にすることで、システムを一定の望ましい状態に維持することができる

というメリットが生まれる。

## 実際どう使うのか

- Cloudのインスタンスにデータベースなどのミドルウェアを入れる
- 授業用のLAMPの環境を作る(playbookを配布する)
- 研究や実験用にVMにdockerやらデータベースやらを入れる

etc...

## Try it!

Nextcloud (ブラウザから利用可能なファイル共有サービス)を構築してみる

### 用意するもの

- Linuxマシン(Ubuntu 20.04.4 LTS Server を想定)
    - コントロールノード(Ansibleを動かしているサーバ)
    - リモートノード(Nextcloudを動かしたいサーバ)

ここでは、コントロールノードとリモートノードを分離したケースを解説している。

だが、勉強会では、コントロールノードにインストールなどを行う。  
つまり、**コントロールノードとリモートノードは同一のマシン**とする。適宜読み替えてほしい。

### 0. Ansible自体のインストール

`コントロールノード` に、Ansibleをインストールする

```bash
# パッケージリストをアップデートし、Pythonの仮想実行環境をインストール
sudo apt update && sudo apt install python3-venv
# Pythonの仮想実行環境をホームディレクトリ以下に ansiblenv という名前で作成
python3 -m venv ~/ansiblenv
# 作成したPythonの仮想実行環境に入る
source ~/ansiblenv/bin/activate
# pipをアップデート、依存パッケージもインストール
pip install -U pip setuptools wheel
# ansibleをインストール
pip install ansible
# コマンドが使えることを確認
ansible-playbook --version

# [Tips] Pythonの仮想実行環境を抜ける:
# deactivate
```

### 1. コントロールノードからリモートノードにSSHできるようにする

`コントロールノード`:

```bash
# ssh鍵を作成
ssh-keygen -t ed25519
# [Enter押しまくる]

# 公開鍵 コピーしておく
cat ~/.ssh/id_ed25519.pub
```

`リモートノード`:

```bash
# コントロールノードのSSH公開鍵をリモートノードに登録する
echo '[コピーした公開鍵]' >> ~/.ssh/authorized_keys
# 例: echo 'ssh-ed25519 AAAAC4NzaC2lZDI2NTE6AAAAIPe+7EGFuuCm3no9g8xc9UDAtb0gQWfXxyliJFQlnNX2 user@Гангут' >> ~/.ssh/authorized_keys
```

`コントロールノード`:

```bash
# 接続確認
ssh [リモートノードのユーザー名]@[リモートノードのアドレス]
# 例: ssh user@192.0.2.10
# yes/no 聞かれたら yes と答える
```

### 2. コントロールノードにPlaybookを用意する

ゴリゴリとYAMLを書いていく。

今回は、予め書いておいたPlaybookを使用する。`ansible`ディレクトリを参照されたい。  
ファイルを見てもらうと、Playbookとはどんなものかがよくわかるだろう。

用語の解説を記載する。

- Playbook:  
    roleとインベントリファイルなどからなる、ファイルのまとまりを指す。
- インベントリファイル:  
    リモートノードのアドレス、実行するroleの種類や順番などを記述するファイル。実行の起点となる。  
    例: `hosts`, `site.yaml` (どのファイルがインベントリファイルか定義があやしい。大体この2つのどちらかと思っておけば多分おｋ)
- role:  
    taskを記述し、役割ごとにまとまりを持たせたもの。構成するディレクトリやファイルの名前によって、それらに書かれたtaskなどをどう実行するかが決まっている。  
    例: `task/main.yaml` には実行するtaskが書かれているが、`defaults/main.yaml`にはtaskの実行に必要な変数を定義している。これらのファイル名・変数名を変えてしまうと、Playbookの実行に支障をきたす。
- task:  
    サーバ操作の実行単位。結局のところ、taskを実行しまくって構成管理を行う。taskはAnsibleのmoduleを使用して実行される。taskの定義方法は、[moduleのリファレンス](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/#modules)を見ながらYAMLファイルに書いていく。(リファレンスと言ってもExampleが豊富で非常に読みやすい)

さて、ファイルの読み方がわかったところで、自分用にPlaybookを書き換えてみる。

### 3. Playbookを各自環境に合うように書き換える

- vault passwordを変更:
    暗号化のためのパスワードを`.vault_password`に書き込む。
    - `ansible-vault`コマンドを使用することで、ファイルレベルや変数レベルで暗号化ができる。  
        playbookはGitなどのバージョン管理システムで管理されるべきだが、パスワードを記述した変数(`ansible_become_password`など)をそこで他人に見られるとまずいため。  
        暗号化した[ファイル/変数]は、playの際に自動で復号されたものが適用される。  
        decryptで復号もできるし、editで一時的に復号し編集することも可能。  
        よく使うのは以下の暗号化のコマンド。
        - ファイルレベルの暗号化: `ansible-vault encrypt [file path]`
        - 変数レベルの暗号化: `ansible-vault encrypt_string`
            - ターミナルに生の文字列が残ってしまうため、次のシェルコマンドを使用すべき:  
                ```
                (printf "encrypt string: ";read -s PASS;ansible-vault encrypt_string <<< ${PASS})
                ```

- アカウントを作成:
    `playbook/roles/nextcloud/defaults/main.yaml`の`admin_username`と`admin_password`を自分好みに書き換えてみる。

- リモートノードのアクセス情報を書き加える:
    `playbook/group_vars/server`の`ansible_user`, `ansible_become_password`をSSH接続先の`ユーザー名`、`パスワード`に書き換えてみる。

### 4. Playbookにエラーがないかチェック

```bash
# yamlの構文に間違いがないか
ansible-playbook --syntax-check -i hosts site.yaml
# 実行してみてエラーが発生しないか(実際の変更は加えられない。そのため変更された状態を前提とするtaskは失敗してしまう)
ansible-playbook --check -i hosts site.yaml
# 構文は汚くないか(しなくてもよい。実行には ansible-lint をインストールしている必要がある)
ansible-lint
```

### 5. 実行

```bash
ansible-playbook -i hosts site.yaml
```

### 6. 確認

ブラウザで`[リモートノードのアドレス]`にアクセスしてみる

Nextcloudの画面が表示されたら成功！

## おわりに

Ansibleを使えば、kubernetesのHAクラスターのような大規模なものでも、ヒューマンエラーの心配なく、何回でも、同じ状態に、サーバーを維持することができます。  
冪等性やIaCを実現する自動化ツールで、SSH接続ができれば使用できる(リモートノードはWindowsにも対応！)ため非常に強力。  
Ansibleすごい！

「docker image, Dockerfileでええやん」  
-> ~~なんだァ？てめェ......~~ dockerなどのコンテナではカバーできない領域はどうしても存在します。(コンテナでは満たせないスペック(速度やセキュリティ)の要求、ベアメタルサーバのユーザー・ミドルウェア・ネットワーク管理など)

今回使ったplaybookでは、リモートノードのwebルート(https://example.com/ のようなアドレス)にNextcloudが配置されています。  
URLを変更したい場合(https://example.com/nextcloud/ などでアクセスできるようにする)には、Nginxなどのロードバランサとなるミドルウェアのroleを追加し、いくつかの設定を変更する必要があります。  
Nextcloudのほうの設定は少し書いてコメントアウトしていますが、興味があれば、Nginxのroleを追加し、playbookを完成させてみてはいかかでしょう。

---

# ご参考まで

[Ansible Galaxy](https://galaxy.ansible.com/): 他人の作ったモジュールやroleを配布しているサイト。検索結果の画面はブラウザが停止するほど重い。  
[Ansible workshops](https://github.com/ansible/workshops/tree/devel/roles): サンプルのroleなどが書かれている。AnsibleはRedHat傘下なのでRHELやRedHatの営業がコンテンツに入っている。
[Ansible Documentation](https://docs.ansible.com/ansible/2.9_ja/)
