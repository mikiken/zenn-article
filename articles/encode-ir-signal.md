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
このデコード結果を見ると、前から8byte目と最後のbyteが変化していることが分かります。リモコン信号の終端付近のbyteは誤り訂正用であることが多いので、前から8byte目が温度を表していそうです。

8byte目の数値そのものに注目してみると、16℃のときに`00`であり、1℃上げるごとに1ずつ増え、31℃のときに`0f`になっていることが分かります。よって、8byte目にセットすべき値は、`(設定温度) - 16`であると推測できます。

### 同じbyteに複数のパラメータが含まれている場合
基本的には上記で説明したように、エアコンの運転状態のパラメータを1つだけ変化させた際に、どのbyteが変化しているかに注目することで、信号の解析を進めることができます。

ただし、パラメータの中には、データ長が1byte以下のものも存在しえます。上位4bitと下位4bitにそれぞれ別のパラメータが入っているような場合であれば、まだ分かりやすいですが[^2]、1byteの中に複数のパラメータが含まれている場合、一見しただけではどこにパラメータが含まれているのか分かりづらいです。

[^2]: 16進数の1桁は、2進数の4桁に相当するため

そのようなパラメータを特定するには、**変化しているバイトを2進数表記してみると判別しやすい**です。

例えば、今回解析したリモコン[^3]の場合、風向(上下)と風量を変化させると、前から10byte目が変化していましたが、どのような規則でパラメータが配置されているかは、一見しただけでは分かりませんでした。

[^3]: 三菱電機の霧ヶ峰のリモコン(型番:NA053)。詳細は[前回の記事](https://zenn.dev/mikiken/articles/decode-ir-signal)を参照

このような場合の解析の方法を実際に見てみましょう。

エアコンのリモコンの設定を、

- 冷房16℃
- 風速:自動
- 風向(左右):スイング

にセットし、風向(上下)を`自動`→`一番上`→`上`→`真ん中`→`下`→`一番下`→`スイング`と変化させた際のパース結果を以下に示します。

```
23 cb 26 01 00 20 58 00 c6 40 00 00 00 00 10 00 00 a3
23 cb 26 01 00 20 58 00 c0 48 00 00 00 00 10 00 00 a5
23 cb 26 01 00 20 58 00 c0 50 00 00 00 00 10 00 00 ad
23 cb 26 01 00 20 58 00 c0 58 00 00 00 00 10 00 00 b5
23 cb 26 01 00 20 58 00 c0 60 00 00 00 00 10 00 00 bd
23 cb 26 01 00 20 58 00 c0 78 00 00 00 00 10 00 00 d5
```

10byte目が変化していることは分かりますが、数字の規則性が見えてきません。そこで、10byte目を2進数表記してみます。

| 風向(上下) | hex | bin |
| :---: | :---: | :---: |
| 一番上 | 40 | 01**000**000 |
| 上 | 48 | 01**001**000 |
| 真ん中 | 50 | 01**010**000 |
| 下 | 58 | 01**011**000 |
| 一番下 | 60 | 01**100**000 |
| スイング | 78 | 01**111**000 |

これを見ると、中央の3bitのみが変化していることが分かります。そのため、この3bitが風向(上下)を表していると推測できます。

同様に、風量について調べてみると10byte目の下位3bitが風量を表していることが分かりました。[^4]

[^4]: 具体的には、`風量自動`: `0b000`, `風量弱`: `0b001`, `風量中`: `0b010`, `風量強`: `0b011` となっていた

### 誤り訂正用byteの計算
大体のリモコンでは、送信信号の最後に誤り訂正用のbyteを付与していることが多いです。しかし、このbyteにどのような値がセットされるかは、特に規格が決まっているわけではなく、実装依存です。そのため、複数の信号を比較しながら推測するしかありません。

誤り訂正の代表的な手法として、

- パリティチェック
- チェックサム
- CRC

などがありますが、いくつかのリモコンを解析してみた経験では、チェックサムが採用されていることが多かったです。具体的には、**最後のbyteを除いた送信信号のhexを1byteずつ加算し、和の下位1byteをチェックサムとしている場合が多かった**です。

## 最終的な解析結果
以上のような手順で信号を解析していたのですが、途中で調べていると、ほぼ同じリモコンの信号を解析している方を見つけました。
https://qiita.com/Hiroki_Kawakami/items/37cdb412a4e511a58103

~~最初からこの記事の結果をお借りすればよかった😇~~
この記事の結果も参考にしつつ解析を進めた結果は、以下のようになりました。

| byte | パラメータ | 値 |
| :---: | :---: | :---: |
| Data1 | 固定 | `0x23`|
| Data2 | 固定 | `0xcb`|
| Data3 | 固定 | `0x26`|
| Data4 | 固定 | `0x01`|
| Data5 | 電源 | ON: `0x20`, OFF: `0x00` |
| Data6 | 運転モード | 冷房: `0x18`, 暖房: `0x08`,<br>除湿: `0x10`, 送風: `0x38`,<br>冷房MoveEye: `0x58`,<br>暖房MoveEye: `0x48`,<br>除湿MoveEye: `0x50`|
| Data7 | 温度 | `(設定温度) - 16` |
| Data8<br>(上位4bit) | 風向(左右) | 左端: `0x1`, 左: `0x2`, 中央: `0x3`,<br> 右: `0x4`, 右端: `0x5`, スイング: `0xc` |
| Data8<br>(下位4bit) | 除湿強さ | 強: `0x0`, 中: `0x2`, 弱: `0x4`<br>※冷房のときは`0x6`, 暖房/送風のときは`0x0` |
| Data9<br>(上位2bit) | 固定 | `0b01` |
| Data9<br>(中央3bit) | 風向(上下) | 自動: `0b000`, 一番上: `0b001`,<br>上: `0b010`, 真ん中: `0b011`,<br>下: `0b100`, 一番下: `0b101`,<br>スイング: `0b111` |
| Data9<br>(下位3bit) | 風量 | 自動: `0b000`, 弱: `0b001`,<br>中: `0b010`, 強: `0b011` |
| Data10 | 固定 | `0x00` |
| Data11 | 固定 | `0x00` |
| Data12 | 固定 | `0x00` |
| Data13 | 風向(左右) | スイング: `0x00`,<br>全体: `0x80`, 左: `0x40`, 右: `0xc0` |
| Data14 | 固定 | `0x10` |
| Data15 | 固定 | `0x00` |
| Data16 | 固定 | `0x00` |
| Data17 | チェックサム | Data1 ~ Data16の1byte単位の和の下位1byte |



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

    # byte 9 (upper 2bit) : fixed
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

## 送信信号のhexから赤外線LEDのON/OFFパターンを生成する
最後に、送信信号のhexをバイナリデータにエンコードし、それを赤外線LEDのON/OFFパターンに変換する関数を作成します。[^5]

[^5]: [前回の記事](https://zenn.dev/mikiken/articles/decode-ir-signal)で説明した内容の真逆の操作に相当する

```python
# リモコン信号のhexからAEHAフォーマット準拠のバイナリデータを生成する
def encode_aeha_hex_to_bin(encoded_hex):
    bin_data = ""
    while len(encoded_hex) > 0:
        byte = "{:08b}".format(int(encoded_hex[:2], 16))
        bin_data += byte[::-1]
        encoded_hex = encoded_hex[2:]
    return bin_data

# バイナリデータから赤外線LEDのON/OFFパターンを生成する
def encode_ir_signal(
    format: str,
    encoded_hex: str,
    unit_time: int,
    repeat: int,
):
    if format == "AEHA":
        bin_data = encode_aeha_to_bin(encoded_hex)
        unit_frame = [unit_time * 8, unit_time * 4]
        for b in bin_data:
            if b == "0":
                unit_frame.extend([unit_time, unit_time])
            else:
                unit_frame.extend([unit_time, unit_time * 3])
        frame = []
        for i in range(repeat):
            frame += unit_frame
            frame += [unit_time, unit_time * 30]
        return frame
```

## 最後に
[前回の記事](https://zenn.dev/mikiken/articles/decode-ir-signal)と合わせて、エアコンのリモコン信号を解析し、それを再現する方法について説明しました。普段、何気なく使っているリモコンが、案外複雑なことを行っていることが分かったのではないでしょうか。

今回の記事の内容を応用することで、スマートリモコンのようなものを作成することができます。SwitchBotやNature Remoといった既製品を利用するより手間はかかりますが、自作することでよりきめ細かな操作が可能になります。皆さんも是非オレオレスマートリモコンを作ってみてください。
