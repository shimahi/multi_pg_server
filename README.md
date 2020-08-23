# UbuntuにDocker, docker-composeをセットアップして、PostgreSQLサーバを複数台構築する

Ubuntu のバージョンは 18.04 (bionic)

vagrantでLinux環境を作りsshログインし、そこでDockerを起動する

ホストIPは `192.33.10`

※ 最初はDocker上のLinuxコンテナ内で行おうとしたが、コンテナ内でのDockerの起動がうまくいかなかった（デーモン周りの不具合）ため、vagrantで環境構築を行った。
```
$ vagrant up
$ vagrant ssh
```

```
パッケージリストの更新
# apt update
インストールされてるパッケージの更新
# apt -y upgrade

sudo, vimのインストール
# apt install -y sudo vim

作業ユーザー作成
# adduser inu
作業ユーザーにroot権限を与える
# usermod -aG sudo inu
作業ユーザーにログイン
# su inu
```

```
必要パッケージのインストール
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

作業ユーザーでDockerがつかえるように
$ sudo usermod -aG docker inu

/etc/docker ディレクトリを作成
$ sudo mkdir /etc/docker

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
```

```
docker-composeのインストール
$ sudo apt install -y docker-compose

docker-composeの実行

$ sudo docker-compose up -d

(しばらく待つ)
```

```
pg, psqlのインストール
$ sudo apt install -y gnupg
$ wget --quiet -O - https://www.postgresql.org/media/keys/ACCC4CF8.asc | sudo apt-key add -
$ echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" |sudo tee  /etc/apt/sources.list.d/pgdg.list
$ sudo apt update
$ sudo apt -y install postgresql-12 postgresql-client-12
```

```
<PostgreSQLの中に入る(例)>
$ psql -p 5433 -h localhost postgres_project_1 -U postgres_project_1
$ psql -p 5434 -h localhost postgres_project_2 -U postgres_project_2
```


## TODO: 
- 環境変数・ポートを一元管理できるようにしたい
- プロジェクトが増えても簡単にPostgreSQLサーバを作れるようなシェルを作る
(`ports.txt` みたいなポート管理のファイルと、シェル叩いたらポート自動生成 + プロジェクト名に応じたDB作成)
- コンテナ外部からの接続ができるように調査




memo:  
[Ubuntu 最低限抑えておきたい初期設定](https://qiita.com/kotarella1110/items/f638822d64a43824dfa4)  
[ConoHa VPS (ubuntu 18.04) 初期設定メモ](https://qiita.com/jqtype/items/126c33ea176f3ba506c3)
