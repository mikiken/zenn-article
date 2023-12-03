---
title: "Pythonでエアコンのリモコン信号を解析し自在に操作できるようにする"
emoji: "👋"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["HomeAutomation", "Python"]
published: false
---

この記事は、[CAMPHOR- Advent Calendar 2023](https://advent.camph.net)の5日目の記事です。

## はじめに
この記事は、以下の記事の続きです。
https://zenn.dev/mikiken/articles/decode-ir-signal

前回は、赤外線リモコンの信号をバイナリデータにデコードし、それをパースして以下のように16進数として表記する方法までを説明しました。

```
23cb260100205800c64000000000100000a3
```

この記事では、エアコンのリモコン信号を解析した上で、エアコンの運転状態からリモコン信号を生成する方法について説明します。