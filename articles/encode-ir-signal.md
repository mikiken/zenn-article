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

## エアコンのリモコン信号解析の難しさ
[前回の記事で説明した方法](https://zenn.dev/mikiken/articles/decode-ir-signal)で、リモコンの各ボタンを押した際の信号を上記のように記録しておき、その信号をそのまま再現して送信すれば、自由にエアコンを操作できると思うかもしれません。

実際、この方法でテレビやシーリングライトなどのリモコン信号を再現することができます。こうした機器のリモコンは、各ボタンに対して決められた信号を送信しているだけだからです。

それに対し、エアコンのリモコン信号は、この方法では再現できません。なぜなら、エアコンのリモコンは、各ボタンに対して決められた信号を送信しているわけではなく、**エアコンの運転状態に応じて毎回異なる信号を送信しているからです**。

端的にいえば、テレビの音量を上げるボタンを押した際に「音量を上げたい」という信号が送信されるのに対し、エアコンの温度を上げるボタンを押すと、「温度を25℃にしたい」「温度を26℃にしたい」のように、その時点でのエアコンの運転状態に応じて異なる信号が送信されます。

さらに詳しく説明すると、エアコンのリモコン信号には、運転モード(冷房, 暖房, ...)、温度、風量、風向といった、**エアコンの運転状態を指定するパラメータが全て含まれています**。「温度を上げる」ボタンを押した際には、風向や風量といったパラメータも送信されているのです。[^1]

[^1]: 昔、「エアコンのリモコンの信号が正しく本体に届かなかった場合、リモコンに表示されている状態と本体側の状態がズレてしまうのではないか?」と思っていました。実際には、次に本体がリモコンの信号を受信したときに、リモコンと本体の状態は正しく同期されていたのです。

すなわち、エアコンのリモコン信号を再現するには、**送信信号のどのフィールドがどのパラメータに対応しているのか**を全て調べる必要があります。

## 実際に信号を解析してみる
上記で述べた通り、送信信号のどのフィールドがどのパラメータの情報が含まれているかを解析していきます。

解析する際のコツは、**解析したいパラメータ(温度,風向, 風量など)1つのみを変更し、他のパラメータは固定しておく**ことです。

こうすることで、デコードされた信号のどの部分が、解析したいパラメータの情報を含んでいるのかを特定しやすくなります。

### 温度を表すフィールドの特定
まず、エアコンのリモコンの設定を、

- 冷房
- 風速:自動
- 風向(上下):自動
- 風向(左右):スイング

にした状態で、温度を16℃(最低温度)から31℃(最高温度)まで順に1℃ずつ上げてみます。
すると、デコードした16進数は次のようになりました。(見やすさのために空白を挿入しました)

```
23 cb 26 01 00 20 58 00 c6 40 00 00 00 00 10 00 00 a3
23 cb 26 01 00 20 58 01 c6 40 00 00 00 00 10 00 00 a4
23 cb 26 01 00 20 58 02 c6 40 00 00 00 00 10 00 00 a5
23 cb 26 01 00 20 58 03 c6 40 00 00 00 00 10 00 00 a6
23 cb 26 01 00 20 58 04 c6 40 00 00 00 00 10 00 00 a7
23 cb 26 01 00 20 58 05 c6 40 00 00 00 00 10 00 00 a8
23 cb 26 01 00 20 58 06 c6 40 00 00 00 00 10 00 00 a9
23 cb 26 01 00 20 58 07 c6 40 00 00 00 00 10 00 00 aa
23 cb 26 01 00 20 58 08 c6 40 00 00 00 00 10 00 00 ab
23 cb 26 01 00 20 58 09 c6 40 00 00 00 00 10 00 00 ac
23 cb 26 01 00 20 58 0a c6 40 00 00 00 00 10 00 00 ad
23 cb 26 01 00 20 58 0b c6 40 00 00 00 00 10 00 00 ae
23 cb 26 01 00 20 58 0c c6 40 00 00 00 00 10 00 00 af
23 cb 26 01 00 20 58 0d c6 40 00 00 00 00 10 00 00 b0
23 cb 26 01 00 20 58 0e c6 40 00 00 00 00 10 00 00 b1
23 cb 26 01 00 20 58 0f c6 40 00 00 00 00 10 00 00 b2
```
このデコード結果を見ると、前から8byte目と最後のbyteが変化していることが分かります。リモコン信号の終端付近のbyteはパリティであることが多いので、前から8byte目が温度を表していそうです。

8byte目の数値そのものに注目してみると、16℃のときに`00`であり、1℃上げるごとに1ずつ増え、31℃のときに`0f`になっていることが分かります。よって、8byte目にセットすべき値は、`(設定温度) - 16`であると推測できます。

### データ長が1byte以下のフィールドの特定
基本的には上記で説明したように、エアコンの運転状態のパラメータを1つだけ変化させた際に、どのbyteの値が変化しているかに注目することで、信号の解析を進めることができます。

ただし、パラメータの中には、データ長が1byte以下のパラメータも存在する場合があります。上位4bitと下位4bitにそれぞれ別のパラメータが入っているような場合であれば、まだ分かりやすいですが[^2]、1byteの真ん中3bitにパラメータが入っているような場合は、一見しただけではパラメータが含まれている位置が分かりづらいです。

[^2]: 16進数の1桁は、2進数の4桁に相当するため

そのようなパラメータを特定するには、**まず変化しているバイトを特定し、そのバイトを2進数表記してみると判別しやすい**です。

実際、エアコンの風向(上下)のパラメータは、10バイト目の真ん中3bitに含まれていました。
除湿強度 high→normal→low
```
23 cb 26 01 00 20 50 08 c0 40 00 00 00 00 10 00 00 9d
23 cb 26 01 00 20 50 08 c2 40 00 00 00 00 10 00 00 9f
23 cb 26 01 00 20 50 08 c4 40 00 00 00 00 10 00 00 a1
```

風上下 auto→一番上から下→スイング
```
23 cb 26 01 00 20 58 00 c6 40 00 00 00 00 10 00 00 a3
23 cb 26 01 00 20 48 00 c0 48 00 00 00 00 10 00 00 95
23 cb 26 01 00 20 48 00 c0 50 00 00 00 00 10 00 00 9d
23 cb 26 01 00 20 48 00 c0 58 00 00 00 00 10 00 00 a5
23 cb 26 01 00 20 48 00 c0 60 00 00 00 00 10 00 00 ad
23 cb 26 01 00 20 48 00 c0 78 00 00 00 00 10 00 00 c5
```
40 0100 0000
48 0100 1000
50 0101 0000
58 0101 1000
60 0110 0000
78 0111 1000
真ん中3bit

### パリティの計算
大体のリモコンでは、送信信号の終端あたりにパリティを付与していることが多いです。しかし、パリティの計算方法は、特に規格が決まっているわけではなく、実装依存です。そのため、パリティの計算は、複数の信号を比較しながらエスパーするしかありません。

典型的なパリティの計算方法としては、

- 送信信号を4bit or 8bitずつ足す
    - 足した結果の下位4bitをパリティとする
    - 上位4bitと下位4bitをXORする
- hoge

あたりがあります。

### 他のパラメータの特定
とか調べていたら、ほぼ同様のリモコンで信号を解析している方がいた
https://qiita.com/Hiroki_Kawakami/items/37cdb412a4e511a58103

~~最初からこの記事の結果をお借りすればよかった~~

## 運転状態から送信信号のhexを生成する
以上の内容を踏まえ、以下のようにエアコンのパラメータから送信信号のhexを生成する関数を作成しました。

```python
def encode_mitsubishi_aircon(
    power: str,
    mode: str,
    temp: int,
    strength: str = "auto",
    direction_h: str = "swing",
    direction_v: str = "auto",
    wind_area: str = "swing",
    dry_strength: str = "high",
):
    # byte 0 ~ 4 : customer_code1, customer_code2, data0 + parity, data1, data2 (fixed)
    # NOTE: when parsing to hexadecimal notation, the order of data0 and parity, which are 4-bit fields, is reversed
    data = "23cb260100"

    # byte 5 : power
    match power:
        case "on":
            data += "20"
        case "off":
            data += "00"
        case _:
            raise (f"'{power}' is Invalid argument for 'power'.")

    # byte 6 : mode
    match mode:
        case "ac_move_eye":
            data += "58"
        case "ac":
            data += "18"
        case "heat_move_eye":
            data += "48"
        case "heat":
            data += "08"
        case "dry_move_eye":
            data += "50"
        case "dry":
            data += "10"
        case "fan":
            data += "38"
        case _:
            raise (f"'{mode}' is Invalid argument for 'mode'.")

    # byte 7 : temprature
    if 16 <= temp <= 31:
        data += "{:02x}".format(temp - 16)
    else:
        raise (f"'{temp}' is Invalid argument for 'temp'.")

    # byte 8 (upper 4bit) : wind direction (horizontal)
    match direction_h:
        case "leftmost":
            data += "1"
        case "left":
            data += "2"
        case "center":
            data += "3"
        case "right":
            data += "4"
        case "rightmost":
            data += "5"
        case "swing":
            data += "c"
        case _:
            raise (f"'{direction_h}' is Invalid argument for 'direction_h'.")

    # byte 8 (lower 4bit) : dry strength
    match mode:
        case "ac" | "ac_move_eye":
            data += "6"
        case "heat" | "heat_move_eye" | "fan":
            data += "0"
        case "dry" | "dry_move_eye":
            match dry_strength:
                case "high":
                    data += "0"
                case "middle":
                    data += "2"
                case "low":
                    data += "4"

    # byte 9 (upper 2bit) : beep (once : 0b01, twice: 0b10)
    byte9_bin = "01"

    # byte 9 (middle 3bit) : wind direction (vertical)
    match direction_v:
        case "auto":
            byte9_bin += "000"
        case "upmost":
            byte9_bin += "001"
        case "up":
            byte9_bin += "010"
        case "middle":
            byte9_bin += "011"
        case "down":
            byte9_bin += "100"
        case "downmost":
            byte9_bin += "101"
        case "swing":
            byte9_bin += "111"
        case _:
            raise (f"'{direction_v}' is Invalid argument for 'direction_v'.")

    # byte 9 (lower 3bit) : wind strength
    match strength:
        case "auto":
            byte9_bin += "000"
        case "low":
            byte9_bin += "001"
        case "middle":
            byte9_bin += "010"
        case "high":
            byte9_bin += "011"
        case _:
            raise (f"'{strength}' is Invalid argument for 'strength'.")

    data += "{:02x}".format(int(byte9_bin, 2), "x")

    # byte 10 ~ 12 (fixed)
    data += "000000"

    # byte 13 : wind area
    match wind_area:
        case "swing":
            data += "00"
        case "wide":
            data += "80"
        case "left":
            data += "40"
        case "right":
            data += "c0"

    # byte 14 ~ 16 (fixed)
    data += "100000"

    # byte 17 : check sum (256 remainders of the sum of 0~16 bytes)
    byte_sum = 0
    for i in range(0, len(data), 2):
        byte_sum += int(data[i : i + 2], 16)
    data += "{:02x}".format(byte_sum % 256)

    return data
```