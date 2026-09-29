# AGENTS.md

> このリポジトリは NetSec 研究（暗号化トラフィック分類手法の実環境汎化性検証）で使う ET-BERT フォーク
> エージェントは作業前にこのファイル全体を読むこと

## 役割

- 上流: https://github.com/linwhitehat/ET-BERT（UER-py ベース、WWW 2022）
- 本フォーク: NetSec 研究向けの改造を行う作業リポジトリ（submodule として NetSec から参照される）
- 用途: 2×2マトリクス「payload込」列のベースライン -> 今後ペイロード除外オプション等の改造を実装する

## ブランチ運用

- 研究用の作業は `netsec` ブランチで行い NetSec 側から参照させる
- 上流追従は `git fetch origin && git rebase origin/main`（origin は上流を指す前提）
- 上流コードへの変更は最小限にし、研究用の新機能は新規ファイルとして追加する
- 上流バグ修正は上流PRの候補として NetSec 側 docs に記録する

## 研究上の制約（NetSec リポジトリの AGENTS.md に準拠）

- Time-aware Split（ランダムsplit禁止）: 配布TSVはランダム分割のため研究用データは NetSec 側で再構成する
- 学習データは正常トラフィックのみ（one-class）
- 本番pcap・モデル重み・大容量データはコミット禁止（.gitignore 登録済み）
- 攻撃スクリプトは NetSec 側で管理し本リポジトリには置かない

## コミット・コーディング規約

- コミット: `<prefix>: <要約>`（日本語可、簡潔に、diff自明な説明は書かない）
- Python: snake_case、Ruff（line length 100）、ロギングは loguru、型ヒント推奨

## セットアップ（Windows 想定）

1. `uv venv --python 3.11`
2. `uv pip install torch --index-url https://download.pytorch.org/whl/cu126`（PyPI 既定は CPU 版になるため注意）
3. `uv pip install six packaging gdown scapy loguru numpy`
4. 事前学習重み: `gdown 1r1yE34dU2W8zSqx1FkB8gCWri4DQWVtE -O models/pre-trained_model.bin`（716MB）
5. データセット: `gdown 1rZX1Y5v-4eTUX9S6VNEk3xD-UYv2MoC4 -O datasets/cstnet-tls1.3_packet.zip`（展開後 `datasets/cstnet-tls1.3_packet/packet/*.tsv`・120クラス）

## 入力形式（コード解析で確定済みの仕様）

- packet-level の `text_a`: フロー先頭パケット全体（Ethernetヘッダ込み）を hexlify し、先頭 76 文字（38バイト = Ethernet 14 + IPv4 20 + ポート 4）を除去
- 残りを1バイト単位で切り出し、隣接バイトが1バイト重なる bigram（`byte[i]+byte[i+1]`）を最大 128 トークン生成してスペース結合
- 語彙は `models/encryptd_vocab.txt`（60,005語）。実測トークン長は最大 78 程度のため `--seq_length 128` で欠落しない
- NetSec 側抽出器（flowwatch/preprocess/etbert_extract.py）の IPv6 対応では 58 バイト除去を使用
