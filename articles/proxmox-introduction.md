---
title: "ジャンクPCにProxmox VEを導入した"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["proxmox", "linux"]
published: false
---

### はじめに
最近、コンテナとかVMとかを雑に立てられる環境が欲しいと感じるようになったので、ヤフオクで買ったジャンクのノートPCにProxmox VEを導入しました。
ある程度使えるようになるまでに行った設定を記録しておきたいと思います。

<!--![](https://storage.googleapis.com/zenn-user-upload/b46067e7b511-20241012.jpg)-->
![](https://storage.googleapis.com/zenn-user-upload/4dbabebb4250-20241012.jpg)*ヤフオクで買ったジャンクのLet's note（基板のみ）*

余談ですが、購入したのはLet's note CF-SV8 シリーズのマザーボードで、スペックは以下の通りです。
- CPU : Core i5-8365U
- RAM : 8GB
- SSD : 256GB（別途購入）

マザーボードをヤフオクにて3000円程度で落札し、これに加えてACアダプタ（2000円程度）とSSD（3000円程度）を別途用意しました。

### インストール
[公式ページ](https://www.proxmox.com/en/downloads)でisoイメージをダウンロードし、Rufusで適当なUSBメモリに書き込んで、インストールする。
https://dareblo.com/proxmoxve-install/
インストールが完了し、再起動後するとWebコンソールにアクセスするためのURLが表示される。(`https://<インストール時に指定したIPアドレス>:8006/`の形式になってるはず)

### aptリポジトリの変更
初期状態で設定されているaptリポジトリは、Enterprise subscription用のものであり、課金しなければ利用できない。
このままでは`apt install`等が行えないため、aptリポジトリを無償で使えるものに変更する。

`/etc/apt/sources.list`と`/etc/apt/sources.list.d/pve-enterprise.list`を以下のように変更する。

```diff:/etc/apt/sources.list
deb http://ftp.jp.debian.org/debian bookworm main contrib

deb http://ftp.jp.debian.org/debian bookworm-updates main contrib

# security updates
deb http://security.debian.org bookworm-security main contrib

+ deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription
```

```diff:/etc/apt/sources.list.d/pve-enterprise.list
- deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
+ # deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise
```

※改めて調べた感じ、Webコンソールからでも設定できるっぽい
`Datacenter → <nodeの名前> → Updates → Repositories`の順に押下すると、以下の画面が表示される。この画面でリポジトリの追加・無効化を行う。
![](https://storage.googleapis.com/zenn-user-upload/d17f0d6a4fcc-20241012.png)

### 管理者ユーザの追加
#### Linuxへのユーザ追加
ホームディレクトリの作成、ユーザ追加、パスワード設定を行う。
```
root@wasabi-1:~# mkdir /home/mikiken
root@wasabi-1:~# chown mikiken:mikiken /home/mikiken
root@wasabi-1:~# useradd mikiken -d /home/mikiken
root@wasabi-1:~# passwd mikiken
Enter new UNIX password:
Retype new UNIX password:
passwd: password updated successfully
```

sudoをインストールする
```
root@wasabi-1:~# apt update
root@wasabi-1:~# apt upgrade -y
root@wasabi-1:~# apt install -y sudo
```

上記で追加したユーザをsudoersに追加
```
root@wasabi-1:~# visudo
```
```
# User privilege specification
root    ALL=(ALL:ALL) ALL
mikiken ALL=(ALL:ALL) ALL
```

#### Proxmoxへのユーザ追加
上記で作成したLinuxのユーザ名と同じ名前で、管理者アカウントをProxmoxに追加する。
`Datacenter → Permissions → Groups → Create`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/0ae36d492993-20241012.png)

上記で作成したadminグループに対し、Administrator権限を付与する。
`Datacenter → Permissions → Add`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/b803700ba969-20241012.png)

adminグループに、上記で作成した管理者ユーザを追加する。
`Datacenter → Permissions → Users → Add`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/2541edd89f5a-20241013.png)
Realmを`Linux PAM standard authentication`にすることで、Linux自体の認証情報を用いてProxmoxのWebコンソールにログインできるようになる。

Webコンソールに、上記で設定したユーザ名とパスワードでログインできるかを確認する。

#### 作成した管理者ユーザにSSHできるか確認
まず、クライアント側（SSH接続する側）で、`ssh-keygen`して鍵を作成しておく。次に、`ssh-copy-id`コマンドで公開鍵をサーバに送る。
```
$ ssh-copy-id mikiken@192.168.10.112　-i wasabi-1.pub
```

作成したユーザでSSHする。
```
$ ssh mikiken@192.168.10.112 -i ~/.ssh/wasabi-1
```
もちろん上記のようなコマンドを毎回打つのは面倒なので、configを書いておくとよい。

デフォルトシェルがshだったので、bashに変更した。
```:@Proxmox
$ echo $0
sh
$ chsh
Password: 
Changing the login shell for ubuntu
Enter the new value, or press ENTER for the default
	Login Shell [/bin/sh]: /bin/bash
mikiken@wasabi-1:~$
```
#### `sshd_config`のバックアップ
この後`sshd_config`を編集し、rootログイン・パスワード認証を禁止するので、あらかじめバックアップを取っておく。
```bash
mikiken@wasabi-1:~$ sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config_original
```

#### rootログイン・パスワード認証の禁止
```bash
mikiken@wasabi-1:~$ sudo sed -i -e "s/PermitRootLogin yes/PermitRootLogin no/g" /etc/ssh/sshd_config
```
```bash
mikiken@wasabi-1:~$ sudo sed -i -e "s/#PasswordAuthentication yes/PasswordAuthentication no/g" /etc/ssh/sshd_config
```

上記の変更を反映させるために、sshdを再起動する。
```bash
mikiken@wasabi-1:~$ sudo systemctl restart sshd
```

### Tailscaleをインストール
外部からサーバにSSHしたり、Webコンソールに接続したりできるように、Proxmox(のホスト)にTailscaleを導入する。この辺を見るといいはず
https://tailscale.com/kb/1174/install-debian-bookworm

設定が完了すると、Tailscaleで割り当てられている`100.x.y.z`のIPアドレスやドメイン`<ホスト名>.XXXX.ts.net`を用いて、SSHやWebコンソールのログインができるようになる。

### 独自ドメインでアクセスできるようにする
Tailscaleで割り当てられているドメインが若干覚えにくいので、独自ドメインでアクセスできるように、DNSに以下のようなAレコードを追加した。
```
pve.mikiken.net.        300     IN      A       100.73.45.25
```
自分のTailnetに接続している端末に限り、`pve.mikiken.net`で接続できる。

※（備考）当初`pve.mikiken.net` → `<ホスト名>.XXXX.ts.net` のCNAMEレコードを設定したが、`<ホスト名>.XXXX.ts.net` → `100.x.y.z`の名前解決がうまくいかず接続できなかった。少し調べた感じ、`<ホスト名>.XXXX.ts.net` → `100.x.y.z`の部分のリクエストを、外部のDNSサーバに対して行おうとしてしまうのが原因っぽい。

### Webコンソールの証明書エラーを消す
`<ホスト名>.XXXX.ts.net`や上記で設定したドメインでWebコンソールに接続すると、Chrome等のブラウザでは証明書エラーが表示される。これはProxmoxがデフォルトで、自己証明書を利用しているためである。

#### `<ホスト名>.XXXX.ts.net` に対する証明書の発行
まず[TailscaleのコンソールのDNSの設定画面](https://login.tailscale.com/admin/dns)で、**Magic DNS**, **HTTPS Certificates**を有効にする。
次に、 Proxmoxのホストで、以下のコマンドを実行し、証明書を発行する。
- `$ tailscale cert <node名>.XXXX.ts.net`
    ```bash
    mikiken@wasabi-1:~$ tailscale cert wasabi-1.XXXX.ts.net
    ```
上記のコマンドを実行すると、カレントディレクトリに`<node名>.XXXX.ts.net.crt`と`<node名>.XXXX.ts.net.key`が出力されているはず。
 
ProxmoxのWebコンソールにアクセスし、発行した証明書の内容をProxmoxに登録する。
`(画面左のnodeを選択) → Certificates → Upload Custom Certificate`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/5620e81a499b-20241013.png)
 
Private Keyに`<node名>.XXXX.ts.net.key`の内容を、Certificate Chain　に`<node名>.XXXX.ts.net.crt`の内容をコピペする。
証明書の登録後に、Webコンソールを開き直すと、証明書が適用されていることが確認できる。

#### 独自ドメイン `pve.mikiken.net` に対する証明書の発行
独自ドメインでアクセスした際も証明書エラーが出るので、Let's encryptで証明書を発行する。また、Cloudflare DNS APIを通して証明書を自動更新できるように設定する。

##### Step0
まず、証明書を発行したいドメインのDNSサーバをCloudflareに変更しておく。（自分はCloudflare registerでドメインを買ったので、この作業は不要）

##### Step1
次に、Cloudflare APIの
- ゾーンID
- アカウントID
- Origin CA Key

を控えておく。また、
- APIトークン (ゾーンDNSの編集権限があるもの)

を発行し、控えておく。

ゾーンIDとアカウントIDは、Cloudflareのダッシュボードの概要の画面の右下に表示されている。（下記の画像で囲っている部分）
![](https://storage.googleapis.com/zenn-user-upload/a66da056e922-20241013.png)

また、上の画像の`APIトークンを取得`を押下すると、次の画面に遷移する。ここで、Origin CA Keyが確認できる。また、この画面の上部にある「**トークンを作成する**」から、APIトークンを発行できる。APIトークン発行の際、証明書を取得したいドメインゾーンのDNSレコードを編集できる権限を与えておく必要がある。
![](https://storage.googleapis.com/zenn-user-upload/3572bc8b35e9-20241013.png)

##### Step2
次に、ProxmoxのWebコンソールに**rootで**ログインし、SSL証明書の取得・更新通知に用いるメールアドレスを登録する。
`Datacenter → ACME → Accounts`の順に押下し、次のように入力する。
- `Accounts Name` : <適当な名前>
- `E-Mail` : SSL証明書の更新通知に用いたいメールアドレス
- `ACME Directory` : Let's Encrypt V2
- Accept TOSにチェック
![](https://storage.googleapis.com/zenn-user-upload/05e7fe9badcf-20241013.png)

さらに、証明書登録・更新を行うためのプラグインを登録する。同じ画面の下に「**Challenge Plugins**」という項目があるので**Add**をクリックし、次のように入力し、OKを押す。
- `Plugin ID` : cloudflare ※他の名前でも可
- `Validation Delay` : 30
- `CF_Account_ID` : Step1のアカウントID
- `CF_Email` : Cloudflareアカウントに登録しているメールアドレス
- `CF_Key` : Step1のOrigin CA Key
- `CF_Token` : Step1のAPIトークン
- `CF_Zone_ID` : Step1のゾーンID

![](https://storage.googleapis.com/zenn-user-upload/72053e12ebc7-20241013.png)

##### Step3
最後に、Step2までで設定したプラグインを用いて、証明書を発行する。
`Datacenter → <node名> → System → Certificates → ACME → Add`の順に押下すると、次のような画面が表示される。
![](https://storage.googleapis.com/zenn-user-upload/2f905f47ceba-20241013.png)
この画面で、
- `Challenge Type` : DNS
- `Plugin` : cloudflare (Step2で設定したPlugin IDが出てくるはず)
- `Domain` : 証明書を発行したいドメイン

のように入力し、`Create → Order Certificates Now`を押下すると、証明書が発行される。

ブラウザを立ち上げ直すと、証明書が適用されているはず。

### (Option) Webコンソールログイン時のメッセージを消す
デフォルトでは、Webコンソールにログインした際に、「有効なサブスクリプションがありません」みたいなメッセージが表示される。気になる場合は、以下の手順で消すこともできるっぽい。
https://qiita.com/flathill/items/01321c48bdf8022fa37e

### 今後やりたいことなど
- ジャンクPCとかミニPCを買い足してクラスタを組む
- VMを立てて、Kubernetesに入門する
- Terraformで宣言的に構成を管理できるようにする

### 参考資料
https://qiita.com/0ta2/items/eac2eb85b89da52c20c3

https://souiunogaii.hatenablog.com/entry/proxmox7-4-useradd

https://docs.redhat.com/ja/documentation/red_hat_enterprise_linux/8/html/configuring_basic_system_settings/managing-sudo-access_configuring-basic-system-settings#managing-sudo-access_configuring-basic-system-settings

https://tailscale.com/kb/1133/proxmox

https://scrapbox.io/dev-hihumikan/Proxmox_VE%E3%81%AESSL%2FTLS%E8%A8%BC%E6%98%8E%E6%9B%B8%E3%82%92Tailscale%E3%81%AEFQDN%E3%81%A7%E8%A7%A3%E6%B1%BA%E3%81%97%E3%81%A6%EF%BC%8C%E6%8C%81%E3%81%A3%E3%81%A6%E3%81%8F%E3%82%8B

https://qiita.com/katori_m/items/20ec2773000a89103774