# pz80

Z80用のツールを作ってみました。CLIとして動作しますが、Pythonモジュールとしても利用できます。

## 概要

pz80は下記の機能を持ちます。

* **アセンブラ**: Z80アセンブリソースコードをバイナリに変換
* **逆アセンブラ**: バイナリファイルをZ80アセンブリニーモニックに変換
* **ウォーカー**: バイナリの制御フローグラフをトレースし、コードとして到達できないデータ領域を検出

## 目次

* [必要要件](#必要要件)
* [インストール](#インストール)
* [使い方](#使い方)
  * [アセンブラ (asm)](#アセンブラ-asm)
  * [逆アセンブラ (disasm)](#逆アセンブラ-disasm)
  * [ウォーカー (walk)](#ウォーカー-walk)
* [設定ファイル詳細](#設定ファイル詳細)
* [アセンブリ言語仕様](#アセンブリ言語仕様)
* [Pythonモジュールとしての使用](#pythonモジュールとしての使用)
  * [公開API 一覧](#公開api-一覧)
* [ライセンス](#ライセンス)

## 必要要件

* Python 3.10 以上

## インストール

現在ドキュメントのみの公開でプログラムはテスト中です。

## 使い方

`pz80` コマンド（または `python -m pz80`）で使用します。

> **本 README のヘルプ出力は Python 3.10 / 端末幅 90 桁で生成しています。**
> `argparse` の出力形式は Python のバージョンで変わるため（3.13 以降は
> `-f FILE, --file FILE` が `-f, --file FILE` と短縮されます）、
> お使いの環境では表示が異なる場合があります。

```bash
C:\>pz80
usage: pz80 [-h] {disasm,walk,asm} ...

Z80 assembler & disassembler v0.4.28

positional arguments:
  {disasm,walk,asm}
    disasm           Z80 disassembler
    walk             Detect data regions in binary via CFG tracing
    asm              Z80 assembler

options:
  -h, --help         show this help message and exit

C:\>
```

### アセンブラ (asm)

```bash
C:\>pz80 asm --help
usage: pz80 asm [-h] -f FILE -o OUTPUT [-s SIZE] [-D DEFINE]

options:
  -h, --help            show this help message and exit
  -f FILE, --file FILE  asm file
  -o OUTPUT, --output OUTPUT
                        output file(bin)
  -s SIZE, --size SIZE  *option* : output file(bin) size
  -D DEFINE, --define DEFINE
                        symbol definition for conditional assembly (repeatable, merged
                        left to right). NAME=VALUE, NAME (=1), or a Python dict.
                        example: -D NOSCORE=1

C:\>
```

ソースファイルをアセンブルしてバイナリを出力します。

```bash
pz80 asm -f source.asm -o output.bin
```

**オプション:**

* `-f`, `--file`: 入力アセンブリファイル（必須）
* `-o`, `--output`: 出力バイナリファイル（必須）
* `-s`, `--size`: 出力ファイルサイズを指定（オプション）。指定サイズまで `0x00` でパディングします。
* `-D`, `--define`: 条件アセンブル用のシンボル定義（複数指定可）。`NAME=VALUE` または Python の辞書リテラルで渡します（後述）。

### 逆アセンブラ (disasm)

```bash
C:\>pz80 disasm --help
usage: pz80 disasm [-h] [-i INPUT] [-c CONFIG] [-s START] [-n] [-o OUTPUT]

options:
  -h, --help            show this help message and exit
  -i INPUT, --input INPUT
                        input image file (specify -i multiple times for multiple files)
  -c CONFIG, --config CONFIG
                        config file (Python module, see README)
  -s START, --start START
                        start address
  -n, --nodump          remove dump info
  -o OUTPUT, --output OUTPUT
                        output file

C:\>
```

バイナリファイルを0番地から順に配置して逆アセンブルします。

```bash
pz80 disasm -i prg0.bin -i prg1.bin -i prg2.bin
```

**オプション:**

* `-i`, `--input`: 入力バイナリファイル（複数指定可）。`-c` で `bins` を指定した場合は省略可。
* `-s`, `--start`: 逆アセンブル開始アドレス（デフォルト: `0x0000`、または `-c` の `start` 値）
* `-o`, `--output`: 出力ファイル（デフォルト: 標準出力）
* `-n`, `--nodump`: ダンプ情報を非表示にし、アセンブリのみ出力
* `-c`, `--config`: 設定ファイル（Python モジュール）（オプション）

#### 設定ファイルについて

`-c` オプションでPythonモジュール形式の設定ファイルを指定することで、作業対象のバイナリファイルの配置・逆アセンブルの挙動をカスタマイズできます。ファイルパスまたはモジュール名のどちらでも指定できます。設定項目はすべてオプションで、定義した項目のみが適用されます。

| 変数名          | 型        | 対象            | 説明                                                                                    |
| ------------ | -------- | ------------- | ------------------------------------------------------------------------------------- |
| `bins`       | list     | disasm / walk | バイナリファイル配置リスト `[(ファイルパス, ロードアドレス), ...]`。指定時は `-i` 不要。                                |
| `start`      | int      | disasm / walk | 逆アセンブル/ウォーク開始アドレス。CLI の `-s` が未指定の場合に使用。                                              |
| `data`       | list     | disasm        | `db` として扱うアドレス範囲 `[[開始, 終了], ...]`（両端含む）。                                             |
| `chr`        | tuple    | disasm        | バイト値→表示文字の256要素タプル。未指定時は標準ASCIIテーブル（0x20〜0x7E）を使用。                                    |
| `output`     | function | disasm        | カスタム出力関数。未指定時は `アドレス オペコード ラベル ニーモニック` 形式で標準出力。                                       |
| `entry`      | list     | walk / disasm | 追加エントリポイント。シンボル名または整数アドレスで指定。walk では CLI の `-e` とマージ、disasm ではラベル付与（`L_xxxx:`）に流用される。 |
| `m1_handler` | function | disasm / walk | M1サイクル復号ハンドラー `(address, byte) -> byte`。暗号化ROM対応。                                     |

各属性の詳細な使用例は「[設定ファイル詳細](#設定ファイル詳細)」を参照してください。

### ウォーカー (walk)

```bash
C:\>pz80 walk --help
usage: pz80 walk [-h] [-i INPUT] [-c CONFIG] [-s START] [-e ADDR_OR_SYMBOL]
                 [--auto-entry]

options:
  -h, --help            show this help message and exit
  -i INPUT, --input INPUT
                        input binary file (specify -i multiple times for multiple files)
  -c CONFIG, --config CONFIG
                        config file (Python module, see README)
  -s START, --start START
                        start address (default: 0x0000)
  -e ADDR_OR_SYMBOL, --entry ADDR_OR_SYMBOL
                        additional entry point (address or: RESET/RST0-7/IM1/NMI)
  --auto-entry          auto-detect entry points from dispatch idioms (jump tables etc.)

C:\>
```

バイナリファイルを制御フローグラフで解析し、コードとして到達できないアドレス範囲をデータ領域として出力します。出力は `disasm` の設定ファイルの `data` 変数として直接利用できる形式です。

```bash
pz80 walk -i rom.bin -e NMI -e IM1
```

出力例:

```python
data = [
    [0x1000, 0x12FF],
    [0x2000, 0x2FFF],
]
```

**オプション:**

* `-i`, `--input`: 入力バイナリファイル（複数指定可）。複数ファイルは先頭から順に連結して扱います。`-c` で `bins` を指定した場合は省略可。
* `-c`, `--config`: 設定ファイル（Python モジュール）。バイナリファイル配置（`bins`）・エントリポイント（`entry`・`start`）を記述できます（オプション）。
* `-s`, `--start`: メインエントリポイント（デフォルト: `0x0000`、または `-c` の `start` 値）。CLI 指定が `-c` より優先。
* `-e`, `--entry`: 追加エントリポイント（複数指定可）。シンボル名または16進数アドレスで指定。`-c` の `entry` とマージされます。
* `--auto-entry`: ジャンプテーブル等からエントリポイントを自動抽出します（後述）。

#### 設定ファイルを使ったバイナリファイル配置指定

複数のバイナリファイルを異なるアドレスに配置する場合は `-c` 設定ファイルの `bins` 属性を使います。ギャップ領域（バイナリファイル未配置のアドレス）は自動的に出力から除外されます。

```bash
pz80 walk -c rom_layout.py
```

```python
# rom_layout.py
bins = [
    ("smc1f",   0x0000),
    ("smc2f",   0x0800),
    ("smc3f",   0x1000),
    ("smc4f",   0x1800),
    ("e5",      0x2000),
    ("bepr199", 0x2800),
    ("e7",      0x3000),
    ("smc8f",   0x3800),
]
entry = ["NMI", "IM1"]  # 追加エントリポイント（-e と同等）
start = 0x0000          # メインエントリポイント（-s と同等、CLI が優先）
```

同じ設定ファイルを `disasm -c` に渡すと、`bins` を使って同じバイナリファイル配置で逆アセンブルできます。`data` や `output` など disasm 専用の属性も同じファイルにまとめて記述できます。

#### エントリポイントのシンボル名

Z80 の固定ベクタアドレスをシンボル名で指定できます。

| シンボル             | アドレス     | 説明                   |
| ---------------- | -------- | -------------------- |
| `RESET` / `RST0` | `0x0000` | リセットベクタ              |
| `RST1`           | `0x0008` | RST 1                |
| `RST2`           | `0x0010` | RST 2                |
| `RST3`           | `0x0018` | RST 3                |
| `RST4`           | `0x0020` | RST 4                |
| `RST5`           | `0x0028` | RST 5                |
| `RST6`           | `0x0030` | RST 6                |
| `RST7` / `IM1`   | `0x0038` | RST 7 / 割り込みモード1ハンドラ |
| `NMI`            | `0x0066` | 非マスカブル割り込みハンドラ       |

#### disasm との連携例

```bash
# 1. ウォーカーでデータ領域を検出して設定ファイルに保存
pz80 walk -i rom.bin -e NMI > config.py

# 2. 生成した設定ファイルを使って逆アセンブル
pz80 disasm -i rom.bin -c config.py
```

#### 追跡する分岐命令

制御フローグラフは以下の命令の分岐先を追跡します。

* `JP` / `JR` / `CALL` / `DJNZ`：直接アドレス指定の分岐・呼び出し。
* `RST nn`：固定ベクタ（`0x0000`〜`0x0038`）へのサブルーチン呼び出しとして分岐先を追跡しつつ、後続命令も継続します。

#### エントリポイントの自動抽出 (`--auto-entry`)

`JP (HL)` のような間接分岐でトレースは停止します。分岐先はジャンプテーブル内の値なので、通常は `-e` で手動指定が必要です。`--auto-entry` は、**手書きアセンブラのディスパッチは書き方の定型句が有限個しかない**という前提に立ち、到達済みコードから定型句を認識してテーブル基底を逆算します。

```bash
pz80 walk -i rom.bin --auto-entry
```

```python
# auto-entry: [sp-ret] @0x0278 タスク再開: 復帰先は実行時スタック依存のため静的解決不可
# auto-entry: [jp-indirect] @0x02B1 table=0x038B stride=2 -> 0x3800 0x1000 0x3000 0x2000 0x0A7C
# auto-entry: entry = -e 0x0000 -e 0x0066 -e 0x0A7C -e 0x1000 -e 0x2000 -e 0x3000 -e 0x3800
data = [
    [0x000B, 0x0065],
    [0x02C8, 0x02E7],
]
```

`# auto-entry:` 行は Python コメントなので、出力をそのまま設定ファイルに保存できます。抽出根拠が残るため、内容を確認したうえで `-e` に固定する使い方を想定しています。認識する定型句は以下のとおりです。

| 定型句 | 扱い |
| --- | --- |
| `LD HL,tbl` → 添字加算 → `JP (HL)` / `JP (IX)` / `JP (IY)` | テーブルを読む |
| テーブル基底が RAM 経由（`LD (ram),HL`、バイト単位の分割書き込み） | 書き込み元を逆引き |
| `PUSH HL` + `RET`（戻り番地の偽装） | テーブルを読む |
| `CALL disp` 直後にテーブルを埋め込み、`disp` 側が `POP HL` で取得 | 戻り番地をテーブル基底とする |
| テーブル本体が `JP nnnn` の並び | 各スロット先頭をエントリにする |
| `RST n`（到達コード中に実在する場合のみ） | ベクタを採用 |
| `LD SP,HL` + `RET` / `RETN`（タスク再開） | 静的解決不可のため報告のみ |

> **注意**: 誤ったエントリはデータをコードとして誤認させます。`--auto-entry` は既定では無効で、出力される抽出根拠を確認したうえで使ってください。

#### アルゴリズムの限界

* `JP (HL)` / `JP (IX)` / `JP (IY)` などの間接分岐は実行時の値が不明なため、分岐先を追跡できません。ジャンプテーブルやステートマシンで使われる場合、その先のコードを `-e` で手動指定するか、`--auto-entry` で抽出します。
* `LD SP,HL` + `RET` によるタスク再開など、復帰先が実行時のスタック内容に依存する分岐は静的に解決できません。`--auto-entry` は該当箇所を報告するのみです。
* IM2（割り込みモード2）のベクタテーブル経由の呼び出しは `-e` で手動指定が必要です。

## 設定ファイル詳細

`-c` で指定する設定ファイルはPythonソースで構成します。各属性について下記にまとめます。

### バイナリファイル配置 (`bins` / `start`)

複数のバイナリファイルをZ80アドレス空間の異なる位置に配置します。`disasm` と `walk` の両コマンドで使用されます。

```python
# rom_layout.py
bins = [
    ("smc1f",   0x0000),
    ("smc2f",   0x0800),
    ("smc3f",   0x1000),
    ("smc4f",   0x1800),
    ("e5",      0x2000),
    ("bepr199", 0x2800),
    ("e7",      0x3000),
    ("smc8f",   0x3800),
]

start = 0x0000  # 逆アセンブル/ウォーク開始アドレス（CLI の -s が優先）
```

```bash
pz80 disasm -c rom_layout.py
pz80 walk   -c rom_layout.py
```

### データ領域指定 (`data`)

指定したアドレス範囲を命令ではなくデータ（`db`）として出力します。`disasm` 専用。

```python
data = [
    [0x8000, 0x80FF],  # スプライトデータ
    [0x8100, 0x81FF],  # タイルマップ
]
```

出力形式：`db 0xXX ; [文字]`（`[文字]` は `chr` テーブルで決まる）

### 文字テーブル (`chr`)

`db` 行のコメント `; [文字]` に使われる256要素のタプルです。`disasm` 専用。

**デフォルトの動作:**

| アドレス範囲    | 表示                        |
| --------- | ------------------------- |
| 0x00〜0x1F | `.`（制御文字）                 |
| 0x20〜0x7E | ASCII印刷可能文字そのまま（スペース〜`~`） |
| 0x7F〜0xFF | `.`                       |

**カスタマイズ方法:**

全256要素を定義する必要があります。デフォルトテーブルを取得して一部変更するのが効率的です。

```python
from pz80 import Z80

# デフォルトテーブルをベースに一部変更
_chr = list(Z80().strmap)
_chr[0xC7] = "@"   # 0xC7 → '@'
_chr[0xF3] = "f"   # 0xF3 → 'f'
_chr[0xFB] = "h"   # 0xFB → 'h'
chr = tuple(_chr)
```

全256要素を定義すれば独自のコード体系にも対応できます。

```python
chr = (
    #  0    1    2    3    4    5    6    7    8    9    A    B    C    D    E    F
    "0", "1", "2", "3", "4", "5", "6", "7", "8", "9", "A", "B", "C", "D", "E", "F",  # 00
    "G", "H", "I", "J", "K", "L", "M", "N", "O", "P", "Q", "R", "S", "T", "U", "V",  # 10
    "W", "X", "Y", "Z", ".", ".", ".", ".", ".", ".", ".", ".", ".", ".", ".", ".",  # 20
    # ... 以下 0xFF まで256要素
)
```

### カスタム出力 (`output`)

逆アセンブル結果の出力形式を完全に制御できます。`disasm` 専用。

```python
def output(dis, sw):
    """
    dis: 逆アセンブルデータのリスト
    sw:  --nodump フラグ (True: アドレス・オペコードを非表示)
    """
    for p in dis:
        if p.get("label"):
            print(p["label"])
        if p.get("asm"):
            indent = "    " if p.get("opcode") else ""
            print(f'{indent}{p["asm"]}')
```

`dis` の各要素の構成：

| キー        | 型          | 説明                            |
| --------- | ---------- | ----------------------------- |
| `address` | int        | 命令のアドレス                       |
| `opcode`  | list\[int] | オペコードのバイト列（ORG行などでは存在しない場合あり） |
| `asm`     | str        | アセンブリ文字列（例: `ld a, 0x10`）     |
| `label`   | str        | ラベル文字列（存在する場合のみ。例: `L_0100:`） |

### エントリポイント (`entry`)

`walk` コマンドの追加エントリポイントを config に記述できます。CLI の `-e` とマージされます。

```python
entry = ["NMI", "IM1", 0x0018]  # シンボル名・整数アドレスの混在可
```

`disasm` コマンドでは、同じ `entry` がラベル付与（`L_xxxx:`）に流用されます。NMI など逆アセンブル結果のコード中に参照のないエントリポイントにもラベルが付き、`walk` と一貫したラベル付けになります。

### M1ハンドラー (`m1_handler`)

Z80オペコード読み込み時に呼び出されるハンドラーで config に記述できます。`disasm` と `walk` の両方で使用されます。使い方はピンとこないかもしれませんが、例えば暗号化されたバイナリーファイルの復号タイミングに利用できます。

```python
def m1_handler(address, byte):
    # アドレスとバイト値から復号キーを導出する例
    key = (address & 1) ^ ((byte & 0x80) >> 7)
    return byte ^ key
```

config に `m1_handler` を定義することで、`walk` によるデータ領域検出と `disasm` による逆アセンブルの両方に同じ復号ロジックが適用されます。

### 設定ファイルの総合例

バイナリファイル配置・データ領域・文字テーブル・出力・M1ハンドラーをまとめた例です。`disasm` と `walk` の両方に使用できます。

```python
# rom_layout.py

# バイナリファイル配置（walk / disasm 共通）
bins = [
    ("smc1f",   0x0000),
    ("smc2f",   0x0800),
    ("smc3f",   0x1000),
    ("smc4f",   0x1800),
    ("e5",      0x2000),
    ("bepr199", 0x2800),
    ("e7",      0x3000),
    ("smc8f",   0x3800),
]
start = 0x0000

# walk 用エントリポイント
entry = ["NMI", "IM1"]

# M1サイクル復号ハンドラー（walk / disasm 共通）
def m1_handler(address, byte):
    key = (address & 1) ^ ((byte & 0x80) >> 7)
    return byte ^ key

# データ領域（disasm 用）
data = [
    [0x3900, 0x3FFF],
]

# 文字テーブル（disasm 用）: デフォルトをベースに一部変更
from pz80 import Z80
_chr = list(Z80().strmap)
_chr[0xC7] = "@"
chr = tuple(_chr)

# カスタム出力（disasm 用）
def output(dis, sw):
    for p in dis:
        if p.get("label"):
            print(p["label"])
        if p.get("asm"):
            indent = "    " if p.get("opcode") else ""
            print(f'{indent}{p["asm"]}')
```

## アセンブリ言語仕様

### 基本構文

大文字・小文字は区別しません。`;` 以降はコメントとして扱います。

```asm
ORG 0x0100    ; 開始アドレス設定
ld a, 0x10    ; 16進数
ld b, 10      ; 10進数
ld c, 0b1010  ; 2進数
ld d, 0o17    ; 8進数
ld hl, 'AB'   ; 文字リテラル (0x4142)
ld a, -1      ; 負数（0xFF に変換）
```

### 数値リテラル

数値は Python の整数リテラル形式で記述します。

| 記法       | 例          | 説明          |
| -------- | ---------- | ----------- |
| 10進数     | `42`       | 接頭辞なし       |
| 16進数     | `0x2A`     | `0x` 接頭辞    |
| 2進数      | `0b101010` | `0b` 接頭辞    |
| 8進数      | `0o52`     | `0o` 接頭辞    |
| 文字リテラル   | `'A'` `'AB'` | 1〜2文字。2文字は上位=1文字目・下位=2文字目（`'AB'` → `0x4142`） |

* **負数**も指定できます。バイトオペランドは `-128`〜`255`、ワードオペランドは `-32768`〜`65535` の範囲で、内部で2の補数に変換されます（例: `db -1` → `0xFF`）。
* 文字リテラルは式の中でも使用できます（例: `db 'A' + 1`、`dw 'A' * 0x100`）。

### 疑似命令

| 命令        | 説明               | 例                    |
| --------- | ---------------- | -------------------- |
| ORG       | アドレス指定           | ORG 0x8000           |
| EQU       | 定数定義             | VAL: EQU 0xFF        |
| DB / DEFB | バイトデータ定義         | db 1, 2, "String", 0 |
| DW / DEFW | ワードデータ定義         | dw 0x1234, LABEL     |
| DS / DEFS | 指定バイト数をfill値で埋める | ds 16, 0xFF          |
| END       | アセンブル終了（以降の行を無視） | end                  |
| IF / ELSE / ENDIF | 条件アセンブル      | IF NOSCORE ... ENDIF |

> **`INCLUDE` / `INCBIN` / `MACRO` / `REPT` は疑似命令として実装しません。**
> これらは [複数チャンクの連結アセンブル](#複数チャンクの連結アセンブル) で Python 側から実現します。
>
> マクロ機構が必要とする機能（引数・条件分岐・繰り返し・文字列処理・ファイル入出力）は
> Python が既に備えています。アセンブラの中に弱い再実装を持ち込んでも表現力は増えず、
> ファイル探索・ネスト深度・循環検出・マクロ展開器といった複雑さだけが増えます。
> それなら Python の言語機能でそのまま書けた方が柔軟です。
>
> 引き換えに、`.asm` ファイル単体ではビルドできず Python 側の記述が必要になります。

**EQU の制約:** 定義値は `0`〜`65535` の範囲です。右辺では**既に定義済みの EQU 定数を参照できます**（後方参照）。前方参照とラベル参照はできません。

```asm
WIDTH:  EQU 8
HEIGHT: EQU 8
SIZE:   EQU 8 * 8           ; OK（数値リテラルと算術式）
AREA:   EQU WIDTH * HEIGHT  ; OK（定義済み EQU の参照）
BIG:    EQU AREA > 32       ; OK（比較演算子も使える）

AHEAD:  EQU LATER * 2       ; NG（前方参照）
LATER:  EQU 8
ADDR:   EQU LABEL_0980      ; NG（ラベルのアドレスは参照不可）
```

前方参照とラベル参照ができないのは、EQU の値が `DS` / `DB` のサイズに影響し、それがアドレスに影響するためです。前方参照を許すと「値を決めるためにアドレスが要り、アドレスを決めるために値が要る」という循環になり得ます。ラベルのアドレスも Pass 1 まで確定せず、EQU の置換はそれより前に行われます。

#### 条件アセンブル (IF / ELSE / ENDIF)

条件が偽のブロックは、アセンブル結果に一切含まれません。ラベル定義も ORG も無効になります。

```asm
IF NOSCORE
    LD  de, LABEL_NOSCORE
ELSE
    LD  de, LABEL_0D5F      ; score
ENDIF
```

フラグは `-D` でコマンドラインから渡します。複数指定した場合は左から順にマージされます（同じキーは後勝ち）。

```bash
pz80 asm -f mc.asm -o mc.bin -D NOSCORE=1
pz80 asm -f mc.asm -o mc.bin -D NOSCORE=1 -D TEXTTEST=0
pz80 asm -f mc.asm -o mc.bin -D NOSCORE            # 値を省略すると 1
```

値は `0x` / `0b` 表記も使えます（`-D ADDR=0x8000`）。

多数まとめて渡すときは Python の辞書リテラルも使えます。ただし `{`・`}`・空白をシェルが解釈するため、**引用符で囲む必要があります**。

```bash
pz80 asm -f mc.asm -o mc.bin -D '{"NOSCORE": 1, "TEXTTEST": 0}'   # bash
pz80 asm -f mc.asm -o mc.bin -D "{'NOSCORE': 1, 'TEXTTEST': 0}"   # cmd
```

> 引用符を付けずに `-D {"A":1,"B":2}` と書くとシェルのブレース展開が働き、まったく別の引数に化けます。手打ちでは `NAME=VALUE` 形式を使ってください。

`-D` で渡した値は、ソース先頭に `NOSCORE: EQU 1` を書いたのと同じ扱いになります。辞書リテラルは `ast.literal_eval` で解釈するため、コードは実行されません。値は整数のみです。

**条件式の制約:**

`IF` は行の有無を変えるため、それ以降のすべてのアドレスに影響します。したがって条件は**アドレスが確定する前**に評価できなければならず、参照できるのは既に定義済みの `EQU` 定数だけです。

```asm
VAL: EQU 8
IF VAL          ; OK（定義済み EQU）
IF LABEL_0980   ; NG（ラベルのアドレスは未確定）
IF UNDEFINED    ; NG（未定義シンボルはエラー。IFDEF は用意していません）
```

条件式には比較演算子・論理演算子が使えます（[式の評価](#式の評価)を参照）。

```asm
IF NOSCORE                  ; 0 以外が真
IF VAL == 8
IF VAL > 4 && !DEBUG
IF FLAGS & 0x04
```

到達しない枝の中の条件式は評価されないため、無効ブロック内で未定義シンボルを参照してもエラーにはなりません。

### ラベル

`:` で終わる識別子をラベルとして定義できます。行頭、または命令の前に記述可能です。ラベルも大文字・小文字を区別しません。

**ラベルの命名規則:**

* 先頭文字: 英字 (`A-Z`, `a-z`) または `@`
* 2文字目以降: 英数字・`_` など自由
* 予約語（レジスタ名・ニーモニック・疑似命令名）は使用不可

```asm
LOOP:      dec b       ; OK
SUB_01:    ret         ; OK (_は2文字目以降に使用可)
@START:    nop         ; OK (@ で始まるラベル)
_LABEL:    nop         ; NG (先頭が _ は不可)
HL:        nop         ; NG (予約語: レジスタ名)
ORG:       nop         ; NG (予約語: 疑似命令名)

LOOP: dec b
   jp nz, LOOP
```

### 式の評価

オペランドには数式を記述できます。演算子の優先順位はC言語準拠です（上ほど優先順位が低い）。

| 演算子                    | 説明        |
| ---------------------- | --------- |
| `\|\|`                  | 論理OR      |
| `&&`                   | 論理AND     |
| `\|`                    | ビットOR     |
| `^`                    | ビットXOR    |
| `&`                    | ビットAND    |
| `==` `!=`              | 等価比較      |
| `<` `<=` `>` `>=`      | 大小比較      |
| `<<` `>>`              | 左/右シフト    |
| `+` `-`                | 加減算       |
| `*` `/` `%`            | 乗除算・剰余    |

* `/` は**整数除算（切り捨て）**です（例: `7 / 2` → `3`）。ゼロ除算はアセンブルエラーになります。
* 比較演算子と論理演算子は **0 か 1** を返します。0 以外がすべて真です。
* 単項演算子として `-`（負号）・`~`（ビットNOT）・`!`（論理NOT）が使用できます。`~` と `!` は別物で、`~0` は `-1`、`!0` は `1` です。
* 論理演算子は定数式なので**短絡評価しません**（右辺も必ず評価されます）。
* 括弧 `()` で優先順位を明示できます。`$` は現在のアセンブルアドレス（PC）を表します。

```asm
ld a, (TABLE + 1)
dw END_ADDR - START_ADDR
ld a, 0xFF & ~0x80    ; ビット7をクリア
dw TABLE >> 8         ; 上位バイトを取り出す
jr $                  ; 無限ループ（$ は JR 命令自身のアドレス）
db $ - START          ; START からのバイト数を埋め込む
ds 16 - ($ & 0x0F)    ; 16バイト境界まで埋める
db SIZE > 0x100       ; 比較結果は 0 か 1
```

比較・論理演算子は条件アセンブルと組み合わせると効きます。

```asm
IF  VERSION == 2 && !DEBUG
    ...
ENDIF
```

## Pythonモジュールとしての使用

Pythonのソースから pz80 をインポートして使用する例です。

### 公開API 一覧

| シンボル                                                                                                        | 種別  | 概要                    |
| ----------------------------------------------------------------------------------------------------------- | --- | --------------------- |
| `assemble(source)`                                                                                          | 関数  | アセンブリソース文字列またはチャンクリストをバイト列に変換 |
| `to_bytes(result)`                                                                                          | 関数  | アセンブル済みリストをバイト列に変換    |
| `disassemble(data, start_address=0, data_regions=None, m1_handler=None, label_addresses=None, strmap=None)` | 関数  | バイト列をアセンブリ文字列リストに変換   |
| `read_chunks(source)`                                                                                       | 関数  | バイナリファイルを整数リストとして読み込む |
| `write_chunks(dest, data)`                                                                                  | 関数  | 整数リストをバイナリファイルに書き出す   |
| `walk(data, start=0, extra_entries=None, valid_ranges=None, m1_handler=None)`                               | 関数  | 制御フローグラフでデータ領域を検出     |
| `Asm`                                                                                                       | クラス | アセンブラ本体 (詳細制御用)       |
| `Disasm`                                                                                                    | クラス | 逆アセンブラ本体 (詳細制御用)      |
| `Z80`                                                                                                       | クラス | Z80命令テーブル・予約語の参照      |
| `__version__`                                                                                               | 文字列 | pz80 のバージョン           |

#### Asm クラスの主なメソッド

| メソッド                               | 概要                                  |
| ---------------------------------- | ----------------------------------- |
| `assemble_lines(lines, file=None)` | 行リスト (`list[str]`) からアセンブル          |
| `assemble_chunks(chunks)`          | 複数チャンク `[(識別子, 行リスト), ...]` からアセンブル |
| `exec(name, defines=None)`         | ソースファイルを読み込んでアセンブル                  |

`assemble_lines()` の `file` は**ソースを識別する任意の文字列**です。ファイルパスとして解釈されることはなく、ファイルシステムにアクセスもしません。用途は 2 つあります。

* エラーメッセージに `in <file>` として出る（省略時は `on line 1` の形）
* 戻り値の各要素の `"file"` キーに入る（リスティングを組むときの引き当てキー）

```python
Asm().assemble_lines(["    ld a, 300"])
# ValueError: Byte value 300 out of range on line 1 (expected -128 to 255)

Asm().assemble_lines(["    ld a, 300"], file="main_code")
# ValueError: Byte value 300 out of range on line 1 in main_code (expected -128 to 255)
```

`None` と空文字列はどちらも「識別子なし」として扱われます。`assemble_lines(lines, file=X)` は `assemble_chunks([(X, lines)])` と等価で、`exec(name)` は渡したファイル名が自動的に `file` になります。

`exec()` の `defines` は**条件アセンブル用のシンボル定義** `{名前: 値}` で、CLI の `-D` に対応します。ソース先頭に `名前: EQU 値` を前置したのと同じ扱いになるため、`IF` から参照でき、`labelmap` にも `equ` として現れます。

```python
Asm().exec("mc.asm", defines={"NOSCORE": 1})
Asm().exec("mc.asm", defines={"NOSCORE": "0x01"})   # 値は文字列でもよい
```

`IF` の条件に未定義のシンボルを書くとエラーになります（`0` とは扱われません）。バリアントを切り替えるソースでは、どの構成でも必ず定義を渡してください。

#### Asm クラスの主な属性

アセンブル実行後に参照します。

| 属性               | 概要                                                                       |
| ---------------- | ------------------------------------------------------------------------ |
| `labelmap`       | シンボル表 `[{"type": "equ"\|"label", "symbol": str, "value": str\|int}, ...]` |
| `label2address`  | ラベルと確定アドレスの対応 `[{"label": str, "address": int}, ...]`                     |

`labelmap` の `symbol` は大文字に正規化されます（シンボル名は大小文字を区別しません）。`value` は `equ` が文字列（評価後の10進）、`label` が整数です。

> `Asm` にはこのほか `cpu` / `encoder` / `preprocessor` / `directive_handler` / `defined_labels` という属性もありますが、**内部実装の都合で持っているだけ**で API ではありません。予告なく変わるため依存しないでください。

#### 戻り値の形式

`assemble_lines()` / `assemble_chunks()` / `exec()` の戻り値はアセンブル済みリストで、ソース上の並び順に次の3種類の辞書が入ります。

| 種類     | 判別方法           | キー                                                          |
| ------ | -------------- | ----------------------------------------------------------- |
| 命令行    | `"opcode"` を持つ | `line`, `file`, `asm`, `base`, `offset`, `opcode`, `fixups`  |
| ラベル定義行 | `"label"` を持つ  | `line`, `file`, `label`, `base`, `offset`                    |
| 消費された行 | `"kind"` を持つ   | `line`, `file`, `asm`, `kind`                                |

`kind` は `"equ"` / `"org"` / `"if"` / `"else"` / `"endif"` / `"skipped"` のいずれかです。**`"skipped"` は条件アセンブルで偽と判定されて捨てられた行**で、疑似命令として消費された行と区別できます。

消費された行は番地を占有しないため `base` / `offset` を持ちません。アドレスを求めるときは `"opcode"` か `"label"` を持つ要素だけを対象にしてください。

戻り値は**行の並び**であってメモリイメージではありません。バイト列が欲しいだけなら `assemble()` か `to_bytes()` を使ってください（「[アセンブル](#アセンブル)」を参照）。行ごとの情報を使ってリスティングを組む例は「[アセンブル結果の一覧表示](#アセンブル結果の一覧表示)」にあります。

#### Disasm クラスの主なメソッド・属性

| 名前                          | 種別    | 概要                                             |
| --------------------------- | ----- | ---------------------------------------------- |
| `exec(start, images, size)` | メソッド  | バイナリイメージを逆アセンブル                                |
| `op2asm(adr, opcode)`       | メソッド  | 1命令のオペコードを文字列化                                 |
| `m1_handler`                | 属性    | M1サイクル復号ハンドラー `(addr, byte) -> byte`（暗号化ROM対応） |
| `label_addresses`           | 属性    | 強制的にラベルを付与するアドレスのリスト（NMI 等の参照なしエントリ用）          |
| `datamap`                   | プロパティ | データ領域 `[[start, end], ...]` の設定                |
| `cpu.strmap`                | 属性    | バイト値 → 表示文字の256要素タプル                           |

`cpu` は `Disasm` が保持する `Z80` インスタンスです。上記以外のメンバは内部実装なので依存しないでください。

#### Z80 クラスの主な属性

| 名前         | 種別    | 概要                                          |
| ---------- | ----- | ------------------------------------------- |
| `reserved` | プロパティ | 予約語 (レジスタ・ニーモニック・疑似命令) のソート済みリスト            |
| `asm_map`  | プロパティ | アセンブラ用マップ (ニーモニック → 命令情報)                   |
| `op_map`   | プロパティ | 逆アセンブラ用マップ (オペコード → 命令情報)                    |
| `codetbl`  | プロパティ | 命令表そのもの (1138 件のリスト)。`asm_map` / `op_map` の元 |
| `strmap`   | プロパティ | バイト値 → 表示文字の256要素タプル (逆アセンブル時のダンプに使う)        |

#### walk のエントリポイント・シンボル名

`walk()` の `extra_entries` には以下のシンボル名 (文字列) または整数アドレスを指定できます。

| シンボル             | アドレス     | シンボル           | アドレス     |
| ---------------- | -------- | -------------- | -------- |
| `RESET` / `RST0` | `0x0000` | `RST4`         | `0x0020` |
| `RST1`           | `0x0008` | `RST5`         | `0x0028` |
| `RST2`           | `0x0010` | `RST6`         | `0x0030` |
| `RST3`           | `0x0018` | `RST7` / `IM1` | `0x0038` |
|                  |          | `NMI`          | `0x0066` |

### バイナリファイルの読み込み

`read_chunks()` はバイナリファイルを整数リストとして読み込む汎用ユーティリティです。

```python
from pz80 import read_chunks

# 単一ファイル
data = read_chunks("rom.bin")  # → list[int]

# 複数ファイルをアドレス指定で配置
data = read_chunks([
    ("smc1f",   0x0000),
    ("smc2f",   0x0800),
    ("smc3f",   0x1000),
    ("smc8f",   0x3800),
])  # → list[int] (アドレス間のギャップは 0x00 で埋められる)
```

読み込んだリストはPythonで直接解析・加工できます。

```python
# パターン探索の例: 'A''@' で始まり 'E''$' で終わるブロックを検出
data = read_chunks("rom.bin")
A, AT = ord('A'), ord('@')
E, DOLLAR = ord('E'), ord('$')

for i in range(len(data) - 1):
    if data[i] == A and data[i + 1] == AT:
        for j in range(i + 2, len(data) - 1):
            if data[j] == E and data[j + 1] == DOLLAR:
                print(f"0x{i:04X} - 0x{j + 1:04X}")
                break
```

`write_chunks()` と組み合わせると read → 加工 → write のワークフローが完結します。

```python
from pz80 import read_chunks, write_chunks

# ROM読み込み・加工・書き出し
data = read_chunks([("smc1f", 0x0000), ("smc2f", 0x0800)])
data[0x0123] = 0x00   # NOP に差し替え（パッチ）
write_chunks("patched.bin", data)

# 元のROM単位に分割して書き出し
write_chunks([
    ("smc1f_patched", 0x0000, 0x07FF),
    ("smc2f_patched", 0x0800, 0x0FFF),
], data)
```

### アセンブル

```python
from pz80 import assemble

source_code = """
    ORG 0x100
    LD A, 42
    RET
"""
# ソースコードをアセンブルしてバイト列を取得
binary_data = assemble(source_code)
```

`assemble()` はソース文字列のほか、後述する**チャンクリスト**も受け付けます。どちらの場合も配置先の最小アドレスから最大アドレスまでを返し、`ORG` で飛ばした範囲は `0x00` で埋まります。

```python
data = assemble("    ORG 0x0000\n    DB 0x11, 0x22\n    ORG 0x0008\n    DB 0x33\n")
# b'\x11\x22\x00\x00\x00\x00\x00\x00\x33'  (9 バイト)
```

#### Asm クラスを使った詳細制御

`Asm` クラスを直接使うと、行リストや複数チャンクからのアセンブルが可能です。

```python
from pz80 import Asm

# 行リストからアセンブル
lines_1 = ["ORG 0x100", "LD A, 42", "RET"]
lines_2 = ["ORG 0x200", "CALL 0x300", "HALT"]

result = Asm().assemble_lines(lines_1)

# file にソースを識別する文字列を渡しておくと、エラーメッセージが
# 「on line 3 in main_code」の形になり、どのソースの何行目か分かる
result = Asm().assemble_lines(lines_1, file="main_code")
result = Asm().assemble_lines(lines_2, file="sub_code")

# ファイル名指定で読み込み（file には "main.asm" が入る）
result = Asm().exec("main.asm")
```

`assemble_lines()` の呼び出しはそれぞれ独立していて、状態は持ち越されません（`ORG` もラベルも引き継がれない）。上の例のように 2 本のソースを扱う場合、`result` は最後の呼び出しの分だけになります。**両方を 1 つのバイナリにまとめたいなら `assemble_chunks()` を使ってください**（後述）。`file` はそのときのチャンク識別子と同じ役割です。

戻り値は行ごとの情報を持つリストです（形式は「[戻り値の形式](#戻り値の形式)」を参照）。バイト列にするには `to_bytes()` に渡します。

```python
from pz80 import Asm, to_bytes

result = Asm().assemble_lines(lines_1)
data = to_bytes(result)
```

戻り値は**行の並び**であってメモリイメージではありません。各行の配置先は `base + offset` で、`ORG` で飛ばした範囲は行として存在しないため、`opcode` を単純に連結すると隙間が詰まります。

```python
# NG: ORG の隙間が失われる（上の 9 バイトの例なら 3 バイトになる）
data = [b for item in result if item.get("opcode") for b in item["opcode"]]
```

#### 複数チャンクの連結アセンブル

`assemble_chunks()` は `(文字列, Python処理系)` のタプルリストを受け取り、すべてを連結してアセンブルします。Python 上で実装した関数をアセンブル時に動作させることでアセンブルの拡張機能をPythonで実現できます。

```python
from pz80 import Asm

def include(filename):
    with open(filename, encoding="utf-8") as f:
        return f.readlines()

def djnz_loop(count, body):
    """Python関数によるマクロ風記述"""
    return [
        f"    LD B, {count}",
        "LOOP:",
        *body,
        "    DJNZ LOOP",
    ]

def embed_binary(filename):
    """バイナリファイルを DB 行のリストに変換する"""
    from pz80 import read_chunks
    return [f"    DB 0x{b:02X}" for b in read_chunks(filename)]

chunks = [
    ("header.asm", include("header.asm")),  # 第1要素は第2要素（Python処理系）を識別するための文字列
                                            # 便宜上ファイル名と同じ文字列を使っているが、
                                            # ファイル名との依存関係はない。
    ("loop_macro", djnz_loop(10, ["    NOP"])),
    ("bootcode",   include("main.asm")),
    ("font_data",  embed_binary("font.bin")),
]
result = Asm().assemble_chunks(chunks)
```

各チャンクに含まれる第1要素の文字列はエラーメッセージ出力時に表示するため、各チャンクを連結後でもエラー箇所を特定できます。

```
Byte value 256 out of range on line 7 in bootcode (expected -128 to 255)
```

**連結のセマンティクス:**

チャンクはリストの順に**上から連結**されます。独立した翻訳単位ではなく、1 本のソースとして扱われるため、次の状態がチャンクを跨いで継続します。`INCLUDE` の代替として設計しているため、この「テキストを貼り付ける」意味づけになります。

* `ORG` で設定したベースアドレス
* ラベル（どのチャンクで定義しても、どのチャンクからも参照できる）
* `EQU` 定数（EQU 右辺での後方参照はチャンクを跨いでも有効。前方参照が不可なのは 1 本のソースの場合と同じ）
* `IF` / `ELSE` / `ENDIF` のネスト状態（`IF` がチャンクを跨いでもよい）

行番号は**チャンクごとに 1 から**振られます。そのためエラー位置の特定には識別子が必要です。識別子が重複した場合は 2 つ目以降に `#2`, `#3` … が付いて一意になります（同じファイルを 2 回取り込む使い方を妨げないため）。

```
Byte value 256 out of range on line 7 in macros.asm#2 (expected -128 to 255)
```

識別子は文字列か `None` のみ受け付けます。`None` と空文字列は「識別子なし」として扱われ、`line 7` の形になります。

第2要素は行の並び（リストやタプル）である必要があり、1 本の文字列を渡すとエラーになります（1 文字ずつイテレートされてしまうため。文字列から作る場合は `splitlines()` で分割してください）。

#### アセンブル結果の一覧表示

アセンブル結果を一覧表示する専用の機能はありません。ただし各行が `(識別子, 行番号)` を持つので、**入力チャンクを同じキーで引けるようにしておく**と、元のソース行（コメント込み）を添えたリスティングが作れます。

チャンクを Python 側で組み立てる使い方では、渡したソースが実際にどう解釈されたかを確かめる手段としてこれが効きます。命令行の `asm` は `EQU` 置換**後**なのでシンボル名が失われますが、元のソース行を添えれば残ります。

```python
from pz80 import Asm

chunks = [
    ("struct.def", ["PLAYER: EQU 0xC000", "PLAYER.hp: EQU 0xC002", "NOSCORE: EQU 1"]),
    ("main.asm",   include("main.asm")),
]
result = Asm().assemble_chunks(chunks)

# (識別子, 行番号) -> 元のソース行
source = {(n, i): ln.rstrip() for n, lines in chunks for i, ln in enumerate(lines, 1)}

for p in result:
    kind = p.get("kind", "label" if "label" in p else "")
    addr = f"{p['base'] + p['offset']:04X}" if "kind" not in p else "    "
    ops = " ".join(f"{b:02X}" for b in p.get("opcode", []))
    print(f"{addr}  {ops:<12s} {kind:<9s} {source[(p['file'], p['line'])]}")
```

```
ADDR  OPCODE       KIND      SOURCE
      equ                    PLAYER: EQU 0xC000
      equ                    PLAYER.hp: EQU 0xC002
      org                            ORG 0x8000
8000            label        START:
8000  21 02 C0                       LD HL,PLAYER.hp  ; メンバ
      if                             IF NOSCORE
8003  00                             NOP
      else                           ELSE
      skipped                        HALT
      endif                          ENDIF
8004  C3 00 80                       JP START
```

`kind` を持つ行はバイトを生成しません。**`"skipped"` だけが条件アセンブルで偽と判定されて捨てられた行**で、`equ` / `org` / `if` / `else` / `endif`（疑似命令として消費された行）と区別できます。`-D` で条件を切り替えるビルドで、どちらの枝が生きたかを確かめられます。

### 逆アセンブル

```python
from pz80 import disassemble

# バイト列を逆アセンブルして命令リストを取得
binary_data = b'\x3E\x2A\xC9'
instructions = disassemble(binary_data, start_address=0x100)
for line in instructions:
    print(line)

# データ領域を指定して逆アセンブル
instructions = disassemble(binary_data, start_address=0x100,
                           data_regions=[[0x8000, 0x80FF]])

# 暗号化ROMをM1ハンドラーで復号しながら逆アセンブル
def decrypt(address, byte):
    return byte ^ 0x55

instructions = disassemble(binary_data, m1_handler=decrypt)

# エントリポイントにラベルを強制付与（NMI など参照のないアドレス用）
# walk() の extra_entries と同じリストを渡すと一貫したラベル付けになる
# 整数アドレスのほか、シンボル名 (NMI 等) も指定できる
instructions = disassemble(binary_data, label_addresses=[0x0020, "NMI"])

# キャラクターコード表を指定（データ領域の db コメント [文字] に反映）
from pz80 import Z80
chr_table = list(Z80().strmap)
chr_table[0xC7] = "@"          # 0xC7 を '@' として表示
instructions = disassemble(binary_data, data_regions=[[0x8000, 0x80FF]],
                           strmap=tuple(chr_table))
```

### データ領域検出 (walk)

```python
from pz80 import walk

with open("rom.bin", "rb") as f:
    binary = f.read()

# CFGトレースでデータ領域を検出
regions = walk(binary, start=0x0000, extra_entries=["NMI", "IM1"])
print(regions)
# → [[0x1000, 0x12FF], [0x2000, 0x2FFF]]
```

`extra_entries` にはシンボル名（`"NMI"`, `"IM1"` など）と整数アドレスを混在して指定できます。

```python
# disasm と組み合わせてデータ領域を正しく逆アセンブル
from pz80 import walk, Disasm

with open("rom.bin", "rb") as f:
    binary = f.read()

regions = walk(binary, start=0x0000, extra_entries=["NMI"])

d = Disasm()
d.datamap = regions
result = d.exec(0x0000, list(binary), len(binary))
```

#### 複数バイナリファイルを異なるアドレスに配置する場合

`read_chunks()` と `valid_ranges` を組み合わせることで、ファイル間のギャップ領域を除外した正確な解析ができます。

```python
import os
from pz80 import read_chunks, walk, Disasm

bins = [
    ("smc1f",   0x0000),
    ("smc2f",   0x0800),
    ("smc3f",   0x1000),
    ("smc8f",   0x3800),
]

# 各ファイルを指定アドレスに配置した images を取得
images = read_chunks(bins)

# valid_ranges: 各ファイルのアドレス範囲 (ファイルサイズから算出)
valid_ranges = [[addr, addr + os.path.getsize(path) - 1] for path, addr in bins]

# ギャップを除いたデータ領域を検出
regions = walk(images, start=0x0000, extra_entries=["NMI", "IM1"],
               valid_ranges=valid_ranges)

# 逆アセンブルにも適用
d = Disasm()
d.datamap = regions
result = d.exec(0x0000, images, len(images))
```

### 暗号化バイナリーファイルの逆アセンブル (M1ハンドラー)

`m1_handler` を暗号化バイナリーファイルの復号化処理に利用できます（暗号化のロジックによる）。

設定した関数はオペコードバイト（プレフィックス含む）のフェッチ時のみ呼ばれ、即値やディスプレースメントなどのオペランドバイトには呼ばれません。

`walk()` と `disassemble()` の両方に `m1_handler` を渡すことで、データ領域検出から逆アセンブルまで一貫して復号できます。`label_addresses` に `walk()` と同じエントリポイントを渡せば、NMI など参照のないアドレスにもラベルが付き一貫します。

```python
from pz80 import Disasm, read_chunks, walk, disassemble

data = read_chunks("encrypted_rom.bin")

def decrypt(address, byte):
    # アドレスとバイト値から復号キーを導出する例
    key = (address & 1) ^ ((byte & 0x80) >> 7)
    return byte ^ key

# エントリポイント（walk と disassemble で共通、シンボル名と整数の混在可）
entries = ["NMI", 0x0020]

# CFGトレースで暗号化ROMのデータ領域を検出
regions = walk(data, start=0x0000, extra_entries=entries, m1_handler=decrypt)

# 同じ entries をラベル付与にも流用できる（解決不要）
lines = disassemble(data, data_regions=regions, m1_handler=decrypt,
                    label_addresses=entries)
```

M1ハンドラーは `(address: int, byte: int) -> int` の形式で定義します。Z80のM1フェッチ規則に従い、以下のバイトにのみ適用されます：

| 命令パターン                     | M1対象バイト                                  |
| -------------------------- | ---------------------------------------- |
| 通常命令                       | バイト0のみ                                   |
| プレフィックス命令 (CB/DD/FD/ED xx) | バイト0・1                                   |
| DDCB/FDCB命令 (DD CB d op)   | バイト0・1のみ（バイト2のディスプレースメント・バイト3のオペコードは対象外） |


> **補足: 元データは破壊されません**
> `m1_handler` の戻り値（復号結果）は逆アセンブル出力にのみ反映され、入力した `data`（`read_chunks()` で構築した images など）は書き換えられません。内部で常にコピーに対して復号を適用するため、同じ `data` を `walk()` と `disassemble()` に続けて渡しても、一方の M1 復号が他方に影響することはありません。

## ライセンス

本プロジェクトは [MIT License](LICENSE) の下で公開されています。
