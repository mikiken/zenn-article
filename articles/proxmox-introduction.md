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

上記で作成したユーザ名と同じ名前で、管理者アカウントをProxmoxに追加する。
`Datacenter → Permissions → Groups → Create`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/0ae36d492993-20241012.png)

上記で作成したadminグループに対し、Administrator権限を付与する。
`Datacenter → Permissions → Add`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/b803700ba969-20241012.png)

adminグループに、上記で作成した管理者ユーザを追加する。
`Datacenter → Permissions → Users → Add`の順に押下する。
![](https://storage.googleapis.com/zenn-user-upload/2541edd89f5a-20241013.png)
Realmを`Linux PAM standard authentication`にすることで、Linux自体の認証情報を用いてProxmoxのWebコンソールにログインできるようになる。

### 作成した管理者ユーザにSSH接続できるか確認
- ログインシェルの変更

### Tailscaleをインストール

### 独自ドメインでアクセスできるようにする

### Webコンソールの証明書エラーを消す

### (Option) Webコンソールログイン時のメッセージを消す

### 今後やりたいことなど
- ジャンクPCとかミニPCを買い足してクラスタを組む
- VMを立てて、Kubernetesに入門する
- Terraformで宣言的に構成を管理できるようにする