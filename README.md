Ubuntu のバージョンは 18.04 (bionic)を使う

```
# ローカル環境はdockerで作成する

まず。systemdのパッケージをコンテナにインストールするためにコンテナを起動し中に入る
$ docker run -it container_name ubuntu:bionic /bin/bash

Systemdをインストール
# apt install systemd -y

コンテナの外へ
# exit

このコンテナをイメージ化する
$ docker commit container_name <イメージ名>:<tag>

このイメージを--privileged /sbin/initで繋げてコンテナ起動
$ docker run --privileged -d -p 22 --name new_container_name <イメージ名>:tag /sbin/init

新しく作ったコンテナに入る
$ docker exec -it new_container_name bash

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
→→→ここでエラー！！！！！
Job for docker.service failed because the control process exited with error code.
See "systemctl status docker.service" and "journalctl -xe" for details.

```

dindで動かないだけ？ConoHaで動けばそれでいいんだが...


memo:  
[Ubuntu 最低限抑えておきたい初期設定](https://qiita.com/kotarella1110/items/f638822d64a43824dfa4)
[ConoHa VPS (ubuntu 18.04) ](https://qiita.com/jqtype/items/126c33ea176f3ba506c3)
