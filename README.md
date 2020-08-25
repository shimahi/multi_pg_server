# PostgreSQL on Docker

VPSを想定した仮想OSをVagrantで立ち上げ、Docker, docker-compose をセットアップして、PostgreSQL サーバを複数台構築する。

仮想OSは **Ubuntu 18.04** を使用


### 仮想OS立ち上げ
```
$ vagrant up
$ vagrant ssh
```

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

作業ユーザーにログイン
# su inu
# <パスワード入力>
```
その他  
[ConoHa VPS スタートアップガイド](https://support.conoha.jp/vps/guide/vpsstartup/?btn_id=top_guide-vpsstartup)  
[Ubuntu 最低限抑えておきたい初期設定](https://qiita.com/kotarella1110/items/f638822d64a43824dfa4)  


### Dockerのセットアップ
```
必要なパッケージのインストール
$ sudo apt install -y apt-transport-https ca-certificates curl software-properties-common gnupg2 wget

Docker公式のGPG鍵を追加
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -

Dockerのaptリポジトリを追加
$ sudo add-apt-repository \
  "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) \
  stable"

Docker CEのインストール
$ sudo apt-get install docker-ce docker-ce-cli containerd.io -y

作業ユーザーでDockerが使えるようにする
$ sudo usermod -aG docker inu

デーモンのセットアップ
$ sudo vim /etc/docker/daemon.json

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
$ sudo mkdir -p /etc/systemd/system/docker.service.d

デーモンの再起動
$ sudo systemctl daemon-reload

Dockerの再起動
$ sudo systemctl restart docker

docker-composeのインストール
$ sudo apt install -y docker-compose

PostgreSQLサーバーのコンテナを立ち上げるためのdocker-compose.yamlを作成する。
$ vim docker-compose.yaml

コンテナの起動
$ sudo docker-compose up -d

DBサーバの起動までしばらくかかるので待つ。ポートが開けているか確認する。
$ sudo docker ps
```

### 仮想OSの内部からDBサーバーにアクセスする
```
前準備: pg, psqlをインストールする
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
$ echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
$ sudo apt update
$ sudo apt -y install postgresql-12 postgresql-client-12
```
自身のホストIP `localhost` と docker-composeで記載したポート・DB名・ユーザ名を指定してDBにログインする
```
$ psql -p 5433 -h localhost project_1_db -U project_1_db_user
```

### 仮想OSの外部からDBサーバーにアクセスする
仮想OSのホストIP `192.168.33.10` (Vagrantfileに記載)を指定してDBにログインする。
```
$ psql -p 5433 -h 192.168.33.10 project_1_db -U project_1_db_user
```

<hr>
#### memo
- 本番運用する場合、安全のために固有IPアドレスからのみアクセスを許可する必要がある。その他セキュリティについて都度調べておく。できる限りdockerで設定したい  

[Certificate Authentication Recipe for PostgreSQL Docker Containers](https://info.crunchydata.com/blog/ssl-certificate-authentication-postgresql-docker-containers)  

[pg_hba.confファイルの設定方法](https://www.dbonline.jp/postgresql/ini/index2.html)  

[PostgreSQL にリモートホストから安全に接続するには](https://qiita.com/tom-sato/items/d5f722fd02ed76db5440)  
- ポートを一元管理できるようにしたい。プロジェクトが増えても簡単に PostgreSQL サーバを作れるようなシェルを作る (シェル叩いたらポート自動生成 + プロジェクト名に応じた コンテナ追加 + ポートが重複しないように `ports.txt` みたいな管理のファイルにポートを追加 )
