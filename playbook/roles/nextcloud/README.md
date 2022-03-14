# Nextcloud

## What is Nextcloud?

Webブラウザやクライアントアプリからアクセスできるファイルサーバ。  
オンプレに導入することができる。

Official site: https://nextcloud.stylez.co.jp/

## What this role do?

Snap版のNextcloudをインストールする。  
指定ポートでアクセスを待ち受ける。

移行モードの場合は、データを移行する。バージョンは最新のものに変更される。  
移行ファイルの準備(下記参照)をしたうえで実行する。  
なお、バージョンが違いすぎると、データベースの構成が違ってくるため、エラーが発生する可能性がある。

---

# 確認すること

- `defaults/main.yaml`を確認して変数を設定する。

## データを移行するとき:

- `copied_datapath`に、移行元の`data`ディレクトリの内容を全てコピー
    - e.g. `user@new:~$ rsync -ae ssh old:/var/www/nextcloud/data/ /root/nextcloud_data/`
- `mysql_nextcloud_migration.sql.tar.bz2`を`files/`に入れる
    - `tar jcvf <移行元のDBをmysqldumpして得た .sql ファイル>`で作成したもの
    - サイズがでかすぎる場合は、dataディレクトリのように、移行先に予め配置しておく方式にタスクを書き換えたほうがいいかもしれない
- `migration_mode`を`yes`にする

## データを移行しないとき:

- `migration_mode`を`no`にする
