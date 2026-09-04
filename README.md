# pz80

Z80用のツールを作ってみました。CLIとして動作しますが、Pythonモジュールとしても利用できます。

## 概要

pz80は下記の機能を持ちます。

* **アセンブラ**: Z80アセンブリソースコードをバイナリに変換
* **逆アセンブラ**: バイナリファイルをZ80アセンブリニーモニックに変換
* **ウォーカー**: バイナリの制御フローグラフをトレースし、コードとして到達できないデータ領域を検出

## ドキュメント

| | 内容 |
| --- | --- |
| **[CLI の使い方](guide/cli.md)** | `pz80 asm` / `disasm` / `walk` の各オプションと、設定ファイル（`bins` / `data` / `chr` / `output` / `entry` / `m1_handler`）の書き方 |
| **[アセンブリ言語仕様](guide/language.md)** | `.asm` に書ける構文。数値リテラル・疑似命令・ラベル・式の評価 |
| **[Python API](guide/python-api.md)** | モジュールとして使う場合の公開 API、戻り値の形式、複数チャンクの連結アセンブル |

## 必要要件

* Python 3.10 以上

## インストール

現在ドキュメントのみの公開でプログラムはテスト中です。

## 使い方の概要

### コマンドラインから

```bash
# アセンブル
pz80 asm -f source.asm -o output.bin

# 逆アセンブル
pz80 disasm -i rom.bin

# データ領域の検出（結果は disasm の設定ファイルにそのまま使える）
pz80 walk -i rom.bin -e NMI
```

各オプションと設定ファイルの詳細は **[guide/cli.md](guide/cli.md)** を参照してください。

### Python モジュールとして

```python
from pz80 import assemble, disassemble, walk

binary = assemble("    ORG 0x100\n    LD A, 42\n    RET\n")
lines = disassemble(binary, start_address=0x100)
data_regions = walk(binary, extra_entries=["NMI"])
```

`Asm` クラスを直接使うと、行リストや複数チャンクからのアセンブル、シンボル表の参照、
アセンブル結果の一覧表示などができます。詳細は **[guide/python-api.md](guide/python-api.md)** を参照してください。

### アセンブリソース

```asm
ORG 0x0100          ; 開始アドレス設定
WIDTH:  EQU 8       ; 定数定義
START:              ; ラベル定義
    LD A, WIDTH
    JP START
```

書ける構文の一覧は **[guide/language.md](guide/language.md)** を参照してください。

## ライセンス

本プロジェクトは [MIT License](LICENSE) の下で公開されています。
