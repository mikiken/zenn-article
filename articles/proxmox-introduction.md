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

### パッケージリストの更新

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