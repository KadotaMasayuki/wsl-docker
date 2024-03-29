# proxy環境のwindowsで、wsl2にdockerを入れる！

proxy環境下で windows-wsl に docker を導入しようとすると認証エラーで非常に手間取るため、記録がてら残す。


## 環境

Windows10 Pro ver 22H2 (OS build 19045.3693)

WSL2

```
PS > wsl -v
WSL バージョン: 2.0.9.0
カーネル バージョン: 5.15.133.1-1
WSLg バージョン: 1.0.59
MSRDC バージョン: 1.2.4677
Direct3D バージョン: 1.611.1-81528511
DXCore バージョン: 10.0.25131.1002-220531-1700.rs-onecore-base2-hyp
Windows バージョン: 10.0.19045.3693
```

Ubuntu-22.04.3 LTS

```
wsl $ uname -a
Linux PC_NAME 5.15.133.1-microsoft-standard-WSL2 #1 SMP Thu Oct 5 21:02:42 UTC 2023 x86_64 x86_64 x86_64 GNU/Linux
wsl $ docker -v
Docker version 24.0.7, build afdd53b
```

> [!NOTE]
> Debianでの差異も記述してあります。
> 
> $ cat /etc/os-release
> 
> PRETTY_NAME="Debian GNU/Linux 12 (bookworm)"
>   ;


## 作業ダイジェスト

- powershellからwslをインストールする。
- wslに好みのlinuxディストリビューションをインストールする。
- linux上のcurlコマンドが、dockerの暗号キーを取得するためhttpsアクセスをするが、proxy下では認証できないため、`~/.profile`にproxy環境変数を設定する。
- linux上のaptコマンドが、dockerリポジトリにアクセスするためにhttpsアクセスするが、proxy下では認証できないため、`/etc/apt/apt.conf.d/proxy.conf`にproxy環境変数を設定する。
- linuxにdockerをインストールする。
- linux上のdockerdサービスが、dockerのレジストリサーバーからイメージをダウンロードするためにhttpsアクセスするが、proxy下では認証できないため、`/etc/systemd/system/docker.service.d`にproxy環境変数を設定する。
- linux上のdockerクライアントが、Dockerfile中またはコンテナ内でhttps接続する際、proxy下では認証できないため、`~/.docker/config.json`にproxy環境変数を設定する。
- linux上の一般ユーザー権限でdockerコマンドを使うために、`dockerグループ`に所属させる。
- linuxディストリビューションを再起動する。


***


# express : 取り急ぎコマンドのみ


## powershellでの作業　wslとlinuxディストリビューションをインストールする

```
# インストールできるlinuxディストリビューションの一覧を表示
wsl -l -o

# インストール済みのディストリビューションとその状態を確認する
wsl -l -v

# Ubuntu-22.04が入っていて、それを削除したい場合
wsl --unregister Ubuntu-22.04

# Ubuntu-22.04をインストール（好きなのを入れる）
wsl --install -d Ubuntu-22.04

# 名前とパスワード、そしてもう一度パスワードを聞かれるので、入力すれば終了
```

## wsl上のlinuxでの作業　proxy環境変数を設定する（proxy環境下ではない場合は不要）

proxyのアドレスとポートは自分の環境に合わせること。

```
# .profileにproxyを設定
cat << EOS >> ~/.profile

export http_proxy=http://12.3.45.67:9999
export https_proxy=http://1.23.45.67.8901
EOS

# .profile読み直し
source .profile

# apt用のproxy設定
cat << EOS | sudo tee -a /etc/apt/apt.conf.d/proxy.conf
Acquire::http::Proxy "http://12.34.56.78:9999";
Acquire::https::Proxy "http://1.23.45.67:8901";
EOS

# ファイルの権限設定
sudo chmod 0644 /etc/apt/apt.conf.d/proxy.conf
```

## wsl上のlinuxでの作業　dockerをインストールする

docker公式の手順に従う。

https://docs.docker.com/engine/install/ubuntu/

```
# dockerの関連ファイルを削除しておく
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# official GPG keyを取得するためにいろいろインストール
sudo apt-get update
sudo apt-get install ca-certificates curl gnupg

# official GPS keyを取得する
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg


> [!NOTE]
> Debianのときは上記gpgの入手先が異なり、上記のままでやると`apt-get update`が失敗するので、公式のDebianのページを確認。https://docs.docker.com/engine/install/debian/


# dockerのリポジトリをaptに追加する
echo \
    "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
    sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# aptリポジトリをアップデートする
sudo apt-get update

# dockerをインストールする
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

## wsl上のlinuxでの作業　dockerのためのproxyを設定する（proxy環境下ではない場合は不要）

```
# dockerd用のproxy設定
mkdir /etc/systemd/system/docker.service.d
cat << EOS | sudo tee -a /etc/systemd/system/docker.service.d/override.conf
[Service]
Environment="HTTP_PROXY=http://12.34.56.78:9999" "HTTPS_PROXY=http://1.23.45.67:8901"
EOS

# ファイルの権限設定
sudo chmod 0644 /etc/systemd/system/docker.service.d/override.conf

# docker client用のproxy設定
cat << EOS >> ~/.docker/config.json

{
  "proxies": {
    "default": {
      "httpProxy": "http://12.34.56.78:9999",
      "httpsProxy": "http://1.23.45.67:8901"
    }
  }
}
EOS
```

## wsl上のlinuxでの作業　dockerを動かしてみる

```
# hello-worldイメージが動くことを確認する
sudo docker run hello-world
```

## dockerをsudo無しで利用するための設定を行う

### wsl上のlinuxでの作業

sudo無しではdockerを利用できないため、dockerグループに自分を所属させて実行権限をつける

```
# sudo無しでdockerを利用するために、自分をdockerグループに所属させる
sudo groupadd docker
sudo usermod -aG docker $USER

# ubuntuを終了する
exit
```

### powershellでの作業

```
# 起動していたwsl側のディストリビューション名を確認する
wsl -l -v

# 対象のディストリビューションを終了する
wsl -t Ubuntu-22.04
```

### windowsでの作業

Ubuntu-22.04のアイコンをクリックするなどして起動すると、再起動完了。

### wsl上のlinuxでの作業

```
# hello-worldイメージをsudo無しで動かしてみる（成功する）
docker run hello-world
```

以上


***


# step by step で解説

`PS >`で始まる行はpowershellのコマンドラインを示す。
`wsl $`で始まる行はwsl上のlinuxのコマンドラインを示す。


## wslをインストールする

wslをmicrosoft storeなどでインストールする。

## linuxディストリビューションをインストールする

### インストールできるlinuxディストリビューションの一覧を表示

linuxディストリビューションはUbuntu-22.04で動作確認済み。Debianや他のディストリビューションでも同じやり方で動くはず。

```
PS > wsl -l -o
インストールできる有効なディストリビューションの一覧を次に示します。
'wsl.exe --install <Distro>' を使用してインストールします。

NAME                                   FRIENDLY NAME
Ubuntu                                 Ubuntu
Debian                                 Debian GNU/Linux
kali-linux                             Kali Linux Rolling
Ubuntu-18.04                           Ubuntu 18.04 LTS
Ubuntu-20.04                           Ubuntu 20.04 LTS
Ubuntu-22.04                           Ubuntu 22.04 LTS
OracleLinux_7_9                        Oracle Linux 7.9
OracleLinux_8_7                        Oracle Linux 8.7
OracleLinux_9_1                        Oracle Linux 9.1
openSUSE-Leap-15.5                     openSUSE Leap 15.5
SUSE-Linux-Enterprise-Server-15-SP4    SUSE Linux Enterprise Server 15 SP4
SUSE-Linux-Enterprise-15-SP5           SUSE Linux Enterprise 15 SP5
openSUSE-Tumbleweed                    openSUSE Tumbleweed
```

### インストール済みのディストリビューションの状態

すでにインストール済みのものは入れられないので、`wsl -l -v`でインストール済みか確認。

ここでは`Ubuntu-22.04`と`Debian`がインストールされていることが分かる。

また、`Ubuntu-22.04`は停止中、`Debian`は起動中であることが分かる。

```
PS > wsl -l -v
  NAME            STATE           VERSION
  Ubuntu-22.04    Stopped         2
  Debian          Running         2
PS >
```

### Ubuntu-22.04を登録解除する

インストール済みだけど入れなおしたいときは、windowsメニューのアプリと機能からアンインストール後、`wsl --unregister ナントカ`で登録削除してから再度インストールする。

たとえば、今まで使っていた Ubuntu-22.04 を捨てて新たに同じものを入れなおしたい、という場合、windows上でアプリと機能からアンインストール後、powershellにて次のようにする。

```
PS > wsl --unregister Ubuntu-22.04
登録解除。
この操作を正しく終了しました。
PS > wsl -l -v
  NAME      STATE           VERSION
  Debian    Running         2
PS >
```

### Ubuntu-22.04をインストールする

```
PS > wsl --install Ubuntu-22.04
インストール中: Ubuntu 22.04 LTS
Ubuntu 22.04 LTS がインストールされました。
Ubuntu 22.04 LTS を起動しています...
Installing, this may take a few minutes...
Please create a default UNIX user account. The username does not need to match your Windows username.
For more information visit: https://aka.ms/wslusers
Enter new UNIX username: XXXXXXX
New password:
Retype new password:
passwd: password updated successfully
Installation successful!
To run a command as administrator (user "root"), use "sudo <command>".
See "man sudo_root" for details.

Welcome to Ubuntu 22.04.3 LTS (GNU/Linux 5.15.133.1-microsoft-standard-WSL2 x86_64)

 * Documentation:  https://help.ubuntu.com
 * Management:     https://landscape.canonical.com
 * Support:        https://ubuntu.com/advantage


This message is shown once a day. To disable it please create the
/home/XXXXXXX/.hushlogin file.
wsl $
```


### とにかくまずはproxyを設定しておく（proxy環境下でなければ不要）

proxy環境下では`curl`コマンドでhttpsのURIからダウンロードしようとすると認証エラーを出したり、`apt update`コマンドがdockerのリポジトリに接続できなかったり、dockerがimageをダウンロードするときに失敗する、などがある（後述）。
そのため、環境変数等にproxyを設定しておく。

windowsが以下のproxy環境のネットワークに接続していると前提する。

```
http-proxy = http://12.3.45.67:9999
https-proxy = http://1.23.45.67.8901
```

#### 環境変数にproxyを設定する

コマンドラインで次のように書き、`~/.profile`にプロキシの環境変数設定を追加する。
んで、sourceしておく。

```
wsl $ cat << EOS >> ~/.profile

export http_proxy=http://12.3.45.67:9999
export https_proxy=http://1.23.45.67.8901
EOS
wsl $ source
wsl $
```

設定されたか確認してみる。以下のようになればOK。

```
wsl $ echo $http_proxy
http://12.3.45.67:9999

wsl $ echo $https_proxy
http://1.23.45.67.8901

wsl $
```

これで、curlでhttpsアクセスできるようになる。


##### proxy環境下でcurlコマンドでhttps接続するとエラーになる

proxy下でcurlコマンドを使う時、proxy環境変数を設定しておかないと、https接続でエラーが出る

```
wsl $ curl https://google.com
curl: (60) SSL certificate problem: unable to get local issuer certificate
More details here: https://curl.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.
```

以下のようにコマンドライン内でproxyを指定してあげることでダウンロードできるが、都度入力する必要があるため、環境変数に設定してしまったほうが良い。

```
wsl $ curl --proxy 12.34.56.78:9999 https://google.com
```


#### aptのためのproxy設定をする

コマンドラインで次のように書き、`/etc/apt/apt.config.d/proxy.conf`を作る。

```
wsl $ cat << EOS | sudo tee -a /etc/apt/apt.conf.d/proxy.conf
Acquire::http::Proxy "http://12.34.56.78:9999";
Acquire::https::Proxy "http://1.23.45.67:8901";
EOS
wsl $ sudo chmod 0644 /etc/apt/apt.conf.d/proxy.conf
wsl $
```

これで、aptでdockerリポジトリのupdateができるようになる。


##### proxy環境下でaptコマンドでhttps接続するとエラーになる

proxy下でaptコマンドを使う時、proxy環境変数を設定しておかないと、dockerのaptリポジトリ(https接続)でエラーが出る

```
wsl $ sudo apt-get update
Hit:1 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:2 http://security.ubuntu.com/ubuntu jammy-security InRelease
Hit:3 http://archive.ubuntu.com/ubuntu jammy-updates InRelease
Ign:4 https://download.docker.com/linux/ubuntu jammy InRelease
Hit:5 http://archive.ubuntu.com/ubuntu jammy-backports InRelease
Ign:4 https://download.docker.com/linux/ubuntu jammy InRelease
Ign:4 https://download.docker.com/linux/ubuntu jammy InRelease
Err:4 https://download.docker.com/linux/ubuntu jammy InRelease
  Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification. [IP: 13.32.50.110 443]
Reading package lists... Done
W: Failed to fetch https://download.docker.com/linux/ubuntu/dists/jammy/InRelease  Certificate verification failed: The certificate is NOT trusted. The certificate issuer is unknown.  Could not handshake: Error in the certificate verification. [IP: 13.32.50.110 443]
W: Some index files failed to download. They have been ignored, or old ones used instead.
```

このとき、直接`apt update`にproxyを指定するとapt updateが成功する。

```
wsl $ sudo apt update  -y -o Acquire::http::Proxy=http://12.34.56.78:9999 -o Acquire::https::Proxy=http://1.23.45.67.8901
Get:1 https://download.docker.com/linux/ubuntu jammy InRelease [48.8 kB]
Hit:2 http://archive.ubuntu.com/ubuntu jammy InRelease
Get:3 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages [23.0 kB]
Get:4 http://security.ubuntu.com/ubuntu jammy-security InRelease [110 kB]
Get:5 http://archive.ubuntu.com/ubuntu jammy-updates InRelease [119 kB]
Hit:6 http://archive.ubuntu.com/ubuntu jammy-backports InRelease
Get:7 http://archive.ubuntu.com/ubuntu jammy-updates/main amd64 Packages [1263 kB]
Get:8 http://archive.ubuntu.com/ubuntu jammy-updates/universe amd64 Packages [1019 kB]
Fetched 2583 kB in 15s (175 kB/s)
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
114 packages can be upgraded. Run 'apt list --upgradable' to see them.
wsl $
```

指定するproxyの書式は、http://xx.xx.xx.xx:yyyyの形式でなければいけない。
http://を除いて、`xx.xx.xx.xx:yyyy`の形式で指定すると以下のように失敗する。

```
wsl $ sudo apt update  -y -o Acquire::http::Proxy=12.34.56.78:9999 -o Acquire::https::Proxy=1.23.45.67.8901
Ign:1 http://security.ubuntu.com/ubuntu jammy-security InRelease
Ign:2 https://download.docker.com/linux/ubuntu jammy InRelease
Err:3 http://security.ubuntu.com/ubuntu jammy-security Release
  Unsupported proxy configured: 1.23.45.67://8901
Err:4 https://download.docker.com/linux/ubuntu jammy Release
  Unsupported proxy configured: 1.23.45.67://8901
Ign:5 http://archive.ubuntu.com/ubuntu jammy InRelease
Ign:6 http://archive.ubuntu.com/ubuntu jammy-updates InRelease
Ign:7 http://archive.ubuntu.com/ubuntu jammy-backports InRelease
Err:8 http://archive.ubuntu.com/ubuntu jammy Release
  Unsupported proxy configured: 1.23.45.67://8901
Err:9 http://archive.ubuntu.com/ubuntu jammy-updates Release
  Unsupported proxy configured: 1.23.45.67://8901
Err:10 http://archive.ubuntu.com/ubuntu jammy-backports Release
  Unsupported proxy configured: 1.23.45.67://8901
Reading package lists... Done
E: The repository 'http://security.ubuntu.com/ubuntu jammy-security Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'https://download.docker.com/linux/ubuntu jammy Release' does not have a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'http://archive.ubuntu.com/ubuntu jammy Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'http://archive.ubuntu.com/ubuntu jammy-updates Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
E: The repository 'http://archive.ubuntu.com/ubuntu jammy-backports Release' no longer has a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
wsl $
```


## docker を ubuntu にインストールする

docker公式の手順に従う。

https://docs.docker.com/engine/install/ubuntu/


### dockerの関連ファイルを削除しておく

```
wsl $ for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
        ;
wsl $
```

### dockerを入れる準備（暗号や認証など）

official GPG keyを取得。

```
wsl $ sudo apt-get update
        ;
wsl $ sudo apt-get install ca-certificates curl gnupg
        ;
wsl $ sudo install -m 0755 -d /etc/apt/keyrings
wsl $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
     # proxy下では.profileにhttps-proxyを設定しないと認証エラー(60)が出る
wsl $ sudo chmod a+r /etc/apt/keyrings/docker.gpg
wsl $
```

#### UbuntuではなくDebianの場合

もし`Ubuntu`ではなく`Debian`の場合、`wsl $ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg`とすると、apt-get updateで下記のようなエラーが出る。

```
wsl $ sudo apt-get update
Hit:1 http://security.debian.org/debian-security bookworm-security InRelease
Hit:2 http://deb.debian.org/debian bookworm InRelease
Hit:3 http://deb.debian.org/debian bookworm-updates InRelease
Hit:4 http://ftp.debian.org/debian bookworm-backports InRelease
Ign:5 https://download.docker.com/linux/ubuntu bookworm InRelease
Err:6 https://download.docker.com/linux/ubuntu bookworm Release
  404  Not Found [IP: 1.23.45.67 8901]
Reading package lists... Done
E: The repository 'https://download.docker.com/linux/ubuntu bookworm Release' does not have a Release file.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
N: See apt-secure(8) manpage for repository creation and user configuration details.
wsl $
```

Debianの場合は https://docs.docker.com/engine/install/debian/ を参考にする。

### dockerのリポジトリをaptに追加しておく。

```
wsl $ echo \
      "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
      $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
      sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
wsl $
```

んで、aptリポジトリをアップデートする。

```
wsl $ sudo apt-get update
Get:1 https://download.docker.com/linux/ubuntu jammy InRelease [48.8 kB]
Get:2 https://download.docker.com/linux/ubuntu jammy/stable amd64 Packages [23.0 kB]
Hit:3 http://archive.ubuntu.com/ubuntu jammy InRelease
Hit:4 http://security.ubuntu.com/ubuntu jammy-security InRelease
Hit:5 http://archive.ubuntu.com/ubuntu jammy-updates InRelease
Hit:6 http://archive.ubuntu.com/ubuntu jammy-backports InRelease
Fetched 71.8 kB in 1s (50.3 kB/s)
Reading package lists... Done
wsl $
```


### dockerをインストールする

```
wsl $ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
       ;
wsl $
```

#### 【ヒント】apt updateが失敗した状態でapt installするとこんなエラーが出る

dockerのリポジトリを追加したがapt updateが成功していない状態だと、ダウンロード先が分からないため、docker-ceなんてどこにあるの？　と言われる。

```
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
Reading package lists... Done
Building dependency tree... Done
Reading state information... Done
Package docker-ce is not available, but is referred to by another package.
This may mean that the package is missing, has been obsoleted, or
is only available from another source

E: Package 'docker-ce' has no installation candidate
E: Unable to locate package docker-ce-cli
E: Unable to locate package containerd.io
E: Couldn't find any package by glob 'containerd.io'
E: Couldn't find any package by regex 'containerd.io'
E: Unable to locate package docker-buildx-plugin
E: Unable to locate package docker-compose-plugin
```

### dockerでhello worldしてみる

```
wsl $ sudo docker run hello-world
docker: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?.
See 'docker run --help'.
```

docker daemonが起動していないかも？　というエラーが出た。
`docker info`してみると、

```
wsl $ docker info
Client: Docker Engine - Community
  ;
  ;
Server:
ERROR: Cannot connect to the Docker daemon at unix:///var/run/docker.sock. Is the docker daemon running?
errors pretty printing info
```

となっている。この場合、

```
wsl $ sudo service docker start
```

してからもう一度やると良い。

```
wsl $ docker info
  ;
  ;
Server:
ERROR: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Get "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/info": dial unix /var/run/docker.sock: connect: permission denied
errors pretty printing info
```

あれ？　Server部分の情報へのアクセスは不許可になっているので、sudoでもう一度。

```
wsl $ sudo docker info
Client: Docker Engine - Community
  ;
  ;
Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: .....
```

起動できているようだ。

### dockerdのためのproxy設定を行う

wsl/linuxへのproxy設定だけでは、dockerdがリポジトリ取得を失敗するため、dockerd用にもproxy設定する必要がある。

#### dockerdへのproxy設定なしで実行すると失敗する

試しに、hello-worldイメージをダウンロードして実行してみる。

```
wsl $ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
docker: Error response from daemon: Get "https://registry-1.docker.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority.
See 'docker run --help'.
wsl $
```

docker daemonが認証エラーしたっぽい。
認証エラーと言えばproxyだろうということで、proxyをコマンドに直書きしてみてもダメ。

```
wsl $ sudo docker run --env HTTPS_PROXY="http://12.34.56.78:9999" hello-world
Unable to find image 'hello-world:latest' locally
docker: Error response from daemon: Get "https://registry-1.docker.io/v2/": tls: failed to verify certificate: x509: certificate signed by unknown authority.
See 'docker run --help'.
wsl $
```

知らない署名がされている。とあるので、知っていることにすれば良いのだが、そのためにsslでpemファイルなどをいっぱい取得して入れるのも違うような気がする。
proxy環境ではない状態では普通に動くので、wsl/linux上での一般的なproxy設定だけでなく、dockerd用のproxy設定も必要な気がする。

念のため`docker info`してみる

```
wsl $ docker info
Client: Docker Engine - Community
 Version:    24.0.7
 Context:    default
 Debug Mode: false
 Plugins:
  buildx: Docker Buildx (Docker Inc.)
    Version:  v0.11.2
    Path:     /usr/libexec/docker/cli-plugins/docker-buildx
  compose: Docker Compose (Docker Inc.)
    Version:  v2.21.0
    Path:     /usr/libexec/docker/cli-plugins/docker-compose

Server:
 Containers: 0
  Running: 0
  Paused: 0
  Stopped: 0
 Images: 0
 Server Version: 24.0.7
 Storage Driver: overlay2
  Backing Filesystem: extfs
  Supports d_type: true
  Using metacopy: false
  Native Overlay Diff: true
  userxattr: false
 Logging Driver: json-file
 Cgroup Driver: cgroupfs
 Cgroup Version: 1
 Plugins:
  Volume: local
  Network: bridge host ipvlan macvlan null overlay
  Log: awslogs fluentd gcplogs gelf journald json-file local logentries splunk syslog
 Swarm: inactive
 Runtimes: io.containerd.runc.v2 runc
 Default Runtime: runc
 Init Binary: docker-init
 containerd version: 3dd1e886e55dd695541fdcd67420c2888645a495
 runc version: v1.1.10-0-g18a0cb0
 init version: de40ad0
 Security Options:
  seccomp
   Profile: builtin
 Kernel Version: 5.15.133.1-microsoft-standard-WSL2
 Operating System: Ubuntu 22.04.3 LTS
 OSType: linux
 Architecture: x86_64
 CPUs: 8
 Total Memory: 7.719GiB
 Name: PC_NAME
 ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false

WARNING: No blkio throttle.read_bps_device support
WARNING: No blkio throttle.write_bps_device support
WARNING: No blkio throttle.read_iops_device support
WARNING: No blkio throttle.write_iops_device support
```

dockerdにproxyの設定がされていないことがわかる。

#### dockerdへproxy設定するために、管理しているプログラムを探す

dockerdを管理しているプログラムに応じてその設定先が異なるため、dockerdを起動しているプログラムを確認する。
systemdで動いている場合はsystemctlコマンドに表示されるはず。

```
wsl $ sudo systemctl status docker
● docker.service - Docker Application Container Engine
     Loaded: loaded (/lib/systemd/system/docker.service; enabled; vendor preset: enabled)
     Active: active (running) since Wed 2023-12-20 08:58:16 JST; 26min ago
TriggeredBy: ● docker.socket
       Docs: https://docs.docker.com
   Main PID: 241 (dockerd)
      Tasks: 13
     Memory: 110.4M
     CGroup: /system.slice/docker.service
             └─241 /usr/bin/dockerd -H fd:// --containerd=/run/containerd/containerd.sock

Dec 20 08:58:15 PC_NAME dockerd[241]: time="2023-12-20T08:58:15.992984900+09:00" level=info msg="Default bridge (dock>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.054277400+09:00" level=info msg="Loading containers: >
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.121718700+09:00" level=warning msg="WARNING: No blkio>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.121760000+09:00" level=warning msg="WARNING: No blkio>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.121775200+09:00" level=warning msg="WARNING: No blkio>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.121783000+09:00" level=warning msg="WARNING: No blkio>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.121818100+09:00" level=info msg="Docker daemon" commi>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.122200200+09:00" level=info msg="Daemon has completed>
Dec 20 08:58:16 PC_NAME dockerd[241]: time="2023-12-20T08:58:16.165194500+09:00" level=info msg="API listen on /run/d>
Dec 20 08:58:16 PC_NAME systemd[1]: Started Docker Application Container Engine.
```

表示されたので、systemd配下で動いていることがわかった。

#### dockerdのproxy設定をsystemd配下に設定を作る

`/etc/systemd/system/docker.serivce`ファイルに書く場合は、すべての設定も記載しなければならない。それを無視してdocker.serviceファイルにproxyだけ書いた場合、dockerが起動しない。
proxyだけ追加などの用途の場合は、`/etc/systemd/system/docker.service.d/override.conf`に記載する。

```
wsl $ mkdir /etc/systemd/system/docker.service.d
wsl $ cat << EOS | sudo tee -a /etc/systemd/system/docker.service.d/override.conf
[Service]
Environment="HTTP_PROXY=http://12.34.56.78:9999" "HTTPS_PROXY=http://1.23.45.67:8901"
EOS
wsl $ sudo chmod 0644 /etc/systemd/system/docker.service.d/override.conf
```

docker daemonを再起動する。

```
wsl $ sudo systemctl restart docker
Warning: The unit file, source configuration file or drop-ins of docker.service changed on disk. Run 'systemctl daemon-reload' to reload units.
```

設定が変わったから設定ファイルを読み直せ、と警告された。
そのとおりにする。

```
wsl $ sudo systemctl daemon-reload
wsl $ sudo systemctl restart docker
```

`docker info`してみる。

```
 ;
 OSType: linux
 Architecture: x86_64
 CPUs: 8
 Total Memory: 7.719GiB
 Name: PC_NAME
 ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
 Docker Root Dir: /var/lib/docker
 Debug Mode: false
 HTTP Proxy: http://12.34.56.78:9999
 HTTPS Proxy: http://1.23.45.67:8901
 Experimental: false
 Insecure Registries:
  127.0.0.0/8
 Live Restore Enabled: false
 ;
```
 
無事proxyの設定がされたようだ。

#### Debianの場合の管理プログラムはserviceらしい

```
wsl $ service --status-all
 [ - ]  apparmor
 [ - ]  cron
 [ - ]  dbus
 [ - ]  docker
 [ ? ]  hwclock.sh
 [ ? ]  kmod
 [ ? ]  networking
 [ - ]  procps
 [ - ]  sudo
 [ + ]  udev
```

`/etc/default/docker`に以下を追加して、`sudo service docker restart`すれば良い。

```
export http_proxy="http://10.0.201.201:8080"
export https_proxy="http://10.0.201.201:8080"
```


### 実行してみる（成功）

いよいよ起動してみる。

```
wsl $ sudo docker run hello-world
Unable to find image 'hello-world:latest' locally
latest: Pulling from library/hello-world
c1ec31eb5944: Pull complete
Digest: sha256:ac69084025c660510933cca701f615283cdbb3aa0963188770b54c31c8962493
Status: Downloaded newer image for hello-world:latest

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
wsl $
```

よし、成功！


### dockerクライアントをsudo無しで利用するための設定を行う

hello-worldイメージをsudo無しで動かしてみる（失敗する）

```
wsl $ docker run hello-world
docker: permission denied while trying to connect to the Docker daemon socket at unix:///var/run/docker.sock: Post "http://%2Fvar%2Frun%2Fdocker.sock/v1.24/containers/create": dial unix /var/run/docker.sock: connect: permission denied.
See 'docker run --help'.
```

dockerを利用できる権限が無いと言われる。
sudoを付けずに利用するためには、dockerグループに自分を所属させる必要がある。
`getent`コマンドでdockerグループの所属員を見てみると、自分が含まれていないことが分かる。

```
wsl $ getent group docker
docker:x:999:
```

もし、dockerグループが無ければ作る。

```
wsl $ sudo groupadd docker
```

dockerグループに自分を追加する。

```
wsl $ sudo usermod -aG docker $USER
```

設定できたか確認する

```
wsl $ getent group docker
docker:x:999:自分のユーザー名
```

または、


```
wsl $ cat /etc/group
root:x:0:
daemon:x:1:
bin:x:2:
sys:x:3:
 ;
 ;
netdev:x:116:自分のユーザー名
自分のユーザー名:x:1000:
docker:x:999:自分のユーザー名
```

設定できているので、ubuntuを再起動するために、いったんexitする。
※ wslでexitコマンドを打って終了して、改めてUbuntuのアイコンをクリックして起動する、というだけでも動くので、これで良いような気がする。

```
wsl $ exit
```

powershellで、wslコマンドを使って再起動する。

もしも、起動していたwsl側のディストリビューション名を忘れた場合は以下で確認。

```
PS > wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-22.04    Running         2
  Debian          Running         2
```

対象のディストリビューションを`wsl -t`で終了する

```
PS > wsl -t Ubuntu-22.04
PS > wsl -l -v
  NAME            STATE           VERSION
* Ubuntu-22.04    Stopped         2
  Debian          Running         2
```

windowsで、Ubuntu-22.04のアイコンをクリックするなどして起動すると、再起動完了。
hello-worldイメージをsudo無しで動かしてみる。

```
wsl $ docker run hello-world

Hello from Docker!
This message shows that your installation appears to be working correctly.

To generate this message, Docker took the following steps:
 1. The Docker client contacted the Docker daemon.
 2. The Docker daemon pulled the "hello-world" image from the Docker Hub.
    (amd64)
 3. The Docker daemon created a new container from that image which runs the
    executable that produces the output you are currently reading.
 4. The Docker daemon streamed that output to the Docker client, which sent it
    to your terminal.

To try something more ambitious, you can run an Ubuntu container with:
 $ docker run -it ubuntu bash

Share images, automate workflows, and more with a free Docker ID:
 https://hub.docker.com/

For more examples and ideas, visit:
 https://docs.docker.com/get-started/
```

成功！


### docker clientにもproxyを設定する

Dockerfile中でhttpsからaptやpipで取得しようとしたり、コンテナ内でhttps接続しようとすると、エラーが出る。
またしてもproxyか。

```
wsl $ docker build -t jupyter .
 ;
 ;
9.262 Could not fetch URL https://pypi.org/simple/jupyterlab/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host='pypi.org', port=443): Max retries exceeded with url: /simple/jupyterlab/ (Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1006)'))) - skipping
9.275 ERROR: Could not find a version that satisfies the requirement jupyterlab (from versions: none)
9.275 ERROR: No matching distribution found for jupyterlab
9.384 Could not fetch URL https://pypi.org/simple/pip/: There was a problem confirming the ssl certificate: HTTPSConnectionPool(host='pypi.org', port=443): Max retries exceeded with url: /simple/pip/ (Caused by SSLError(SSLCertVerificationError(1, '[SSL: CERTIFICATE_VERIFY_FAILED] certificate verify failed: unable to get local issuer certificate (_ssl.c:1006)'))) - skipping
------
Dockerfile:13
--------------------
  11 |     # pip install
  12 |     RUN pip3 install --upgrade pip
  13 | >>> RUN pip3 install jupyterlab
  14 |     RUN pip3 install pandas
  15 |     RUN pip3 install mecab-python3
--------------------
ERROR: failed to solve: process "/bin/sh -c pip3 install jupyterlab" did not complete successfully: exit code: 1
```

docker clientにもproxy設定が必要なので、`~/.docker/config.json`にproxyを設定する。

```
wsl $ cat << EOS >> ~/.docker/config.json

{
  "proxies": {
    "default": {
      "httpProxy": "http://192.168.0.10:8080",
      "httpsProxy": "http://192.168.0.10:8080"
    }
  }
}
EOS
```

これでDockerfileからのビルドが出来るようになる。

以上

