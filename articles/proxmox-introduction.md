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

### 一般ユーザーの追加
- ログインシェルの変更

### Tailscaleをインストール

### 独自ドメインでアクセスできるようにする

### Webコンソールの証明書エラーを消す

### (Option) Webコンソールログイン時のメッセージを消す

### 今後やりたいことなど
- ジャンクPCとかミニPCを買い足してクラスタを組む
- VMを立てて、Kubernetesに入門する
- Terraformで宣言的に構成を管理できるようにする