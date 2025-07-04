---
title: "PKIX-SSH を使って X.509 証明書を用いた公開鍵認証を行う"
emoji: "🔑"
type: "tech"
topics:
  - "ssh"
  - "x509"
published: false
---

SSH の公開鍵認証において、X.509 証明書の証明チェーンで公開鍵の信頼を行いたかったのだが、OpenSSH は [X.509 証明書に対応する予定がない](https://lists.mindrot.org/pipermail/openssh-unix-dev/2022-September/040423.html)らしい。
無理かと思っていたところ、Roumen Petrov 氏による、X.509 証明書のサポートを追加した OpenSSH のフォークを発見した。

https://roumenpetrov.info/secsh/

たった一人でメンテナンスを続け、upstream の変更にも対応している感じっぽい。
Roumen Petrov 氏に最大限の感謝をしつつ基本的な使い方を記す。

## 対象読者

- SSH に慣れ親しんでいる
- X.509 証明書について基本的な理解がある

## 環境

- PKIX-SSH v16.2
- クライアント
  - OpenSSL 3.2.2 4 Jun 2024
  - AlmaLinux 9.6
  - WSL 2.5.7.0
- サーバー
  - OpenSSL 3.2.2 4 Jun 2024
  - Almalinux 9.6

## インストール

### ダウンロード

[GitLab](https://gitlab.com/secsh/pkixssh/-/tree/master) から落としてきてもいいし、[ダウンロードページ](https://roumenpetrov.info/secsh/download.html)から autoreconf 済みの圧縮ファイルを落としてきてもいい。一応安定板ということでダウンロードページから取得することにする。

```sh
mkdir -p ~/softwares && cd softwares
wget https://roumenpetrov.info/secsh/src/pkixssh-16.2.tar.gz
tar -xvf pkixssh-16.2.tar.gz && cd pkixssh-16.2
```

### ビルド

autotools で作られているので、`./configure` を使ってインストール場所などの指定を行う。

今回サーバー側は systemd で自動起動させるようにするので、`--with-systemd` オプションを付けて `notify` タイプが利用できるようにする。

```sh:サーバー側
./configure --with-systemd
make
sudo make install
```

指定しなければ `/usr/local` 配下にインストールされる。
基本的に `/usr/local` 配下のものは `/usr` 配下のものより優先順位が高いので、`ssh` コマンドや man ページなどはすべて上書きされてしまうので注意。と言っても OpenSSH とは互換なのでほとんど気にする必要はない。

クライアント側は `~/.local/` 配下にインストールする。好み。

```sh:クライアント側
./configure --prefix=$HOME/.local/
make
make install
```

## サーバー側初期設定

### systemd ファイル作成

systemd での自動起動に必要な service ファイルを作成する。

`SYSTEMD-SYSTEM.CONF(5)` によると、

```text
In addition to the "main" configuration file, drop-in configuration snippets are read from /usr/lib/systemd/*.conf.d/,
/usr/local/lib/systemd/*.conf.d/, and /etc/systemd/*.conf.d/. Those drop-ins have higher precedence and override the main
configuration file.
```

らしいので、`/usr/local/lib/systemd/system/pkixssh-sshd.service` として作成する。

```sh
mkdir -p /usr/local/lib/systemd/system/
cat << 'EOF' > /usr/local/lib/systemd/system/pkixssh-sshd.service
[Unit]
Description=PKIXSSH server daemon
Documentation=man:sshd(8) man:sshd_config(5)
After=network.target

[Service]
Type=notify
ExecStart=/usr/local/sbin/sshd -D
ExecReload=/bin/kill -HUP $MAINPID
Restart=on-failure
RestartSec=10s

[Install]
WantedBy=multi-user.target
EOF
```

OpenSSH の service ファイルをベースに作成した。

### sshd_config の設定

```conf
Port 2222

X509KeyAlgorithm x509v3-ecdsa-sha2-nistp384,ssh-sha384,ecdsa-sha2-nistp384
X509KeyAlgorithm x509v3-ecdsa-sha2-nistp521,ssh-sha512,ecdsa-sha2-nistp521

AllowedCertPurpose any

PermitRootLogin no
PasswordAuthentication no

AuthorizedKeysFile      .ssh/authorized_keys

# override default of no subsystems
Subsystem       sftp    /usr/local/libexec/sftp-server
```

最低限のものだけ書いてみた。

通常の sshd_config と違うのは `X509KeyAlgorithm` と `AllowedCertPurpose` ぐらいだろう。

`X509KeyAlgorithm` はサーバーが受け入れる X.509 証明書認証方式の一覧を指定する。現時点でできるだけ強い暗号を使いたいと思ったので NIST P-384, P-521 系のみ有効化しているが、好きなものを使えばいい。

`AllowedCertPurpose` は X.509 証明書の keyUsage, extendedKeyUsage に基づいて決定される OpenSSL の Certificate Purpose を指定する。
[RFC6187 - 2.2 Certificate Extensions](https://datatracker.ietf.org/doc/html/rfc6187#section-2.2) によると、SSH クライアント証明書として使う場合は

- keyUsage: digitalSignature
- extendedKeyUsage: secureShellClient

を指定するはずなのだが、デフォルト値である `sshclient` を満たすためには

- keyUsage: digitalSignature
- extendedKeyUsage: serverAuth

となる必要があるらしい。

:::details さらに細かい話
Certificate Purpose の一覧は [openssl-verification-options - Certificate Extensions](https://docs.openssl.org/3.2/man1/openssl-verification-options/#checks-implied-by-specific-predefined-policies) に定義されている。
これは OpenSSL 独自の証明書識別子であり、採用している理由は不明である(OpenSSL には Extended Key Usage を取得できる [X509V3_get_d2i](https://docs.openssl.org/3.2/man3/X509V3_get_d2i/) という API が用意されている)。
:::

今回は RFC6187 を優先し、`AllowedCertPurpose any` としてチェックを回避するようにする。

### 証明書のインストール

デフォルトでは認証局証明書は `/usr/local/etc/ca/crt` 以下に、`[HASH].[NUM]` というファイル名で置くようになっている。
証明書のハッシュ値は以下のコマンドで取得できる。

```sh
openssl x509 -in certificate_file_name -noout -hash
```

NUM は基本 0 で OK。
また、サーバー側にはルート証明書しか置く必要はない。クライアント側が証明チェーン構築に必要なすべての証明書を提示する決まりになっているからだ。
ちなみに、ファイル名を最初からこのフォーマットにしてしまうとなんの証明書かわからなくなるので、わかりやすい名前にしてからシンボリックリンクを張るのがおすすめだ。

## クライアント側の準備

クライアント証明書を発行しておく。

クライアント側での証明書チェーンの構築には２つの方法がある。

1. 秘密鍵と証明書チェーンの構築に必要なすべての証明書を cat して SSH 鍵として使う
2. 秘密鍵とクライアント証明書のみ cat して、中間証明書・ルート証明書は `~/.ssh/crt` に置く

2 のほうがシンプルな感じするので、こっちを採用する。

```sh
cat ~/.ssh/ssh_testuser.key ~/.ssh/ssh_testuser_cert.pem > ~/.ssh/ssh_testuser_with_cert.key
```

`~/.ssh/crt` にはサーバ側と同様、`[HASH].[NUM]` のフォーマットで証明書を置く。
中間証明書も置かなくてはならないことに注意。

## authorized_keys への追加

サーバー側にルート証明書をインストールするだけではダメで、通常の公開鍵認証同様 `~/.ssh/authorized_keys` にエントリを追加する必要がある。
ただし、その書き方は少々異なる。

まずは鍵タイプを確認する。これは `ssh-keygen -f keyfile -y` で出力される公開鍵の一番最初に書いてある、`x509v3-ecdsa-sha2-nistp521` のような文字列のことだ。

次に、クライアント証明書の Distinguished Name (識別名) を確認する。

```sh
$ openssl x509 -noout -subject -in certfile -nameopt RFC2253
subject=CN=testuser,O=daima3629
```

これらをつなげると、

```text
x509v3-ecdsa-sha2-nistp521 subject=CN=testuser,O=daima3629
```

こうなる。これが公開鍵の識別子となる。これを `authorized_keys` に追記してあげればよいというわけだ。

## 動作確認

起動して、動作確認してみる。

```sh
$ sudo systemctl start pkixssh-sshd
$ journalctl -xeu pkixssh-sshd
...
Jul 04 16:32:37 certtest.home.daima3629.com sshd[149102]: Accepted publickey for testuser from x.x.x.x port 58358 ssh2: ECDSA+cert SHA256:eGO9VwZu7+IFPzQfwru9DKpYJMwpqSl3zQJwto3sPDs
...
```

やったね！！！

## 感想

あくまで OpenSSH のフォークなので楽に扱うことができた (証明書チェーンの構築方法など、詰まったところはあったが…)。
個人開発なので企業とかで採用するのは難しそうだが、個人のサーバーでは適切に利用すれば公開鍵インストールの手間など省けるので、積極的に活用したい。
