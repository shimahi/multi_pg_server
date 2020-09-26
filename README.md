# PostgreSQL on Docker

Linuxサーバー環境にDocker, docker-compose をセットアップして、PostgreSQL サーバを複数台構築する。

仮想 OS は **Ubuntu 18.04** を使用

```
$・・・ローカルのターミナル
#・・・Ubuntuのrootユーザー
$$・・・Ubuntuの作業ユーザー
```

<hr>
### 仮想 OS 立ち上げ

```
$ vagrant up
$ vagrant ssh
```

<hr>

### 初期設定

```
パッケージリストの更新
# sudo apt update

インストールされてるパッケージの更新
# sudo apt -y upgrade

作業ユーザー作成
# sudo adduser inu
# <パスワード入力>

作業ユーザーにroot権限を与える
# sudo usermod -aG sudo inu

# sudo visudo
////
## Allows people in group wheel to run all commands
inu ALL=(ALL)       ALL
//////

作業ユーザーにログイン
# su inu
# <パスワード入力>
```

### Docker のセットアップ

```
必要なパッケージのインストール
$$ sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg2 wget

Docker公式のGPG鍵を追加
$$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

Dockerのaptリポジトリを登録
$$ sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

Docker CEのインストール
$$ sudo apt-get install docker-ce docker-ce-cli containerd.io -y

作業ユーザーでDockerが使えるようにする
$$ sudo usermod -aG docker inu

デーモンのセットアップ
$$ sudo vim /etc/docker/daemon.json

/// daemon.json ///
{
  "exec-opts": ["native.cgroupdriver=systemd"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m"
  },
  "storage-driver": "overlay2",
  "storage-opts": [
    "overlay2.override_kernel_check=true"
  ]
}
//////////////////

docker.service.dを作成
$$ sudo mkdir -p /etc/systemd/system/docker.service.d

デーモンの再起動
$$ sudo systemctl daemon-reload

Dockerの再起動
$$ sudo systemctl restart docker

docker-composeのインストール
$$ sudo apt install -y docker-compose

docker-compose置く用のディレクトリを作っておく
$$ mkdir ~/app
$$ cd ~/app

PostgreSQLサーバーのコンテナを立ち上げるためのdocker-compose.yamlを作成する。
$$ vim docker-compose.yaml

コンテナの起動
$$ sudo docker-compose up -d

DBサーバの起動までしばらくかかるので待つ。ポートが開けているか確認する。
$$ sudo docker ps
```

サーバー内部からDBサーバーにアクセスする

```
前準備: pg, psqlをインストールする
$$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
$$ echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
$$ sudo apt update
$$ sudo apt -y install postgresql-12 postgresql-client-12
```

自身のホスト IP `localhost` と docker-compose で記載したポート・DB 名・ユーザ名を指定して DB にログインする

```
$$ psql -p 5433 -h localhost project_1_db -U project_1_db_user
```

### サーバー外部からDBサーバーにアクセスする

サーバーのホスト IP `192.168.33.10` (vagrantの場合、Vagrantfile に記載してあるもの)を指定して DB にログインする。

```
$ psql -p 5433 -h 192.168.33.10 project_1_db -U project_1_db_user
```

<hr>

### ☆ConoHaでVPS借りてからSSHログインまで

IP アドレスにログイン

```
$ ssh root@xxx.ipaddress.xxx -p 22
```

エディタを nano から vim へ

```
# sudo update-alternatives --set editor /usr/bin/vim.basic


前述の <初期設定>の項目から作業ユーザー作成し、サーバーから抜ける
```

再度作業ユーザーでログイン

```
$ ssh inu@xxx.ipaddress.xxx -p 22
```

```
sshファイルの設定
$$ sudo vim /etc/ssh/sshd_config
/////

# ポート番号を22から他のに変更する
Port 2020

# 公開鍵認証を許可。コメントアウトを解除。
PubkeyAuthentication yes

# 公開鍵の場所を設定。コメントアウトを解除。 (☆)
AuthorizedKeysFile      .ssh/authorized_keys

# rootユーザーでのログインを禁止。
PermitRootLogin no

# パスワードログインを禁止。
PasswordAuthentication no

# PAM認証の禁止
UsePAM no
/////

完了したらsshdを再起動
$$ sudo service sshd restart
```

```
ファイアーウォールの設定

↑で設定したポート番号を解放する
$$ sudo ufw allow 2020/tcp
$$ sudo ufw allow 80
$$ sudo ufw allow 443

許可されたポート以外を閉鎖
$$ sudo ufw default deny

ufwの有効化とリロード
$$ sudo ufw enable
$$ sudo ufw reload

$$ sudo ufw status
でポートの状況が見れる

再度sshdを再起動
$$ sudo service sshd restart
```

```
localeの設定

日本語パッケージのインストール
$$ sudo apt -y install language-pack-ja

ja_JP.UTF-8の設定
$$ sudo update-locale LANG=ja_JP.UTF-8
```

```
鍵作る
$$ sudo mkdir ~/.ssh
## sudo chown inu .ssh
$$ cd ~/.ssh
$$ ssh-keygen -t rsa -b 4096
$$ $ ls # → id_rsa id_rsa.pub

! id_rsa.pub が公開鍵

id_rsaを (☆) で設定してある名前に変更する
$ mv id_rsa.pub authorized_keys

鍵の権限変える
$$ cd ~/.ssh
$$ chmod 700 ~/.ssh
$$ chmod 600 ~/.ssh/authorized_keys

秘密鍵をコピー
$$ cat id_rsa
```
ローカルに戻り、秘密鍵を登録する
```
$ cd ~/.ssh/conoha
$ vim id_rsa
// コピーした秘密鍵を貼り付け

権限設定
$ chmod 600 id_rsa  

$ vim ~/.ssh/config
<VPSのホストとポートとユーザーをローカルの秘密鍵の場所に紐づける>
///
Host my_first_conoha
  HostName xxx.xxx.xxx.xxx //リモートのIP
  User inu
  Port 2020
  IdentityFile  ~/.ssh/conoha/id_rsa
///

ssh-agentへの登録
$ sudo ssh-add -K ~/.ssh/conoha/id_rsa
```

ローカルからリモートVPSにSSHログインすることが可能になった
```
$ ssh my_first_conoha
```
[Ubuntu 最低限抑えておきたい初期設定](https://qiita.com/kotarella1110/items/f638822d64a43824dfa4)

<hr>


### 立ち上げたDBサーバとHasura Cloudを接続する

[Hasura Cloud](https://cloud.hasura.io/) で新規プロジェクトを立ち上げる  

`Enter a Postgres Database URL` に以下の形式でDB情報を記述する
```
postgresql://[user[:password]@][netloc][:port][/dbname][?param1=value1&...]

e.g. )
postgresql://project_1_db_user:password@xxx.ipaddress.xxx:5433/project_1_db
```

`Allow connections to your database from Hasura Cloud IP` のIPアドレスからのアクセスのみ許可するようにPostgresqlを設定する
```
$$ sudo vim  ~/app/my_first_conoha/<db_volume_name>/postgresql.conf

///
listen_addresses = 'HasuraのIPアドレス'
///

$$ sudo vim ~/app/my_first_conoha/<db_volume_name>/pg_hba.conf
///
host    all             all             HasuraのIPアドレス            trust
///
```


`Create project`でHasuraとDBサーバが接続できる。

<hr>

### Memo

[Postgres permissions](https://hasura.io/docs/1.0/graphql/cloud/projects/create.html#postgres-permissions)
[Postgres requirements](https://hasura.io/docs/1.0/graphql/core/deployment/postgres-requirements.html)

[Certificate Authentication Recipe for PostgreSQL Docker Containers](https://info.crunchydata.com/blog/ssl-certificate-authentication-postgresql-docker-containers)

[pg_hba.conf ファイルの設定方法](https://www.dbonline.jp/postgresql/ini/index2.html)

[PostgreSQLで外部許可](https://qiita.com/nao_yoshi/items/a5106e8f08547a54d5ea)

[PostgreSQL にリモートホストから安全に接続するには](https://qiita.com/tom-sato/items/d5f722fd02ed76db5440)



- ポートを一元管理できるようにしたい。プロジェクトが増えても簡単に PostgreSQL サーバを作れるようなシェルを作る (シェル叩いたらポート自動生成 + プロジェクト名に応じた コンテナ追加 + ポートが重複しないように `ports.txt` みたいな管理のファイルにポートを追加)
