# MimicX-protocol

**製品名: Mimic X**

MIDI 1.0 を利用した汎用HID制御通信プロトコルライブラリ。Dart（Flutter向け）とC/C++（マイコン向け）の実装を提供。下位トランスポートは USB-MIDI / BLE-MIDI に対応 (0.8.0+)。

「Mimic X」プロジェクトの中核となる通信プロトコル仕様。

## 概要

スマートフォンとマイコン間の MIDI 通信 (USB-MIDI / BLE-MIDI) を抽象化し、キーボード・マウス・ジョイスティックなどのHIDデバイス制御メッセージを送受信するためのプロトコルを定義する。リアルタイム / SysEx のワイヤフォーマットはトランスポート非依存で、タイムアウト等のパラメータのみトランスポートごとに調整する (仕様書 §2)。

## ドキュメント

- **[プロトコル仕様書](docs/protocol-spec.md)** — メッセージフォーマット、チャンネル割り当て、ネゴシエーション手順

## 設計方針

- **リアルタイム操作は素の MIDI メッセージ** (Note On/Off, Control Change) で最小レイテンシを実現
- **デバイス識別・設定は SysEx** で可変長の構造化データを送受信
- デバイスが自己申告するため、アプリ側は汎用的に対応可能

## バージョニング・リリース手順

プロトコルバージョンは Semantic Versioning (MAJOR.MINOR) を採用。
詳細は [プロトコル仕様書](docs/protocol-spec.md) の §9 / 変更履歴を参照。

- **MAJOR**: 後方互換性のない変更 (既存のコマンド形式やビット割り当ての破壊的変更)
- **MINOR**: 後方互換性のある機能追加 (新規コマンド・新規 SysEx 命令の追加など)

仕様変更を反映するときの流れ:

1. `docs/protocol-spec.md` の冒頭ヘッダ (`**Version:**`, `**Date:**`) と
   「変更履歴」セクションに新しいエントリを追記
2. 既存節を編集 / 新節を追加 (コマンド表・例も同期させる)
3. main に commit + push
4. タグを打つ場合は `git tag -a v<MAJOR>.<MINOR>.<PATCH> -m "..."` で注釈付きで

実装側 (firmware / app) はこの `MAJOR.MINOR` をハンドシェイク (`IDENTIFY_RESPONSE`)
で交換し、最低互換バージョンを満たさない組み合わせなら接続を拒否する。実装側で
プロトコル番号を上げるときも仕様書側の version と揃えること:

- firmware: `src/main.c` の `PROTOCOL_VERSION_MAJOR` / `_MINOR`
- app: `lib/protocol.dart` の `MinSupportedProtocol.knownLatestMajor` / `knownLatestMinor`

## 関連リポジトリ

- [MimicX-app](https://github.com/kunichiko/MimicX-app) - Flutterアプリ
- [MimicX-firmware](https://github.com/kunichiko/MimicX-firmware) - マイコンファームウェア
- [MimicX-hardware](https://github.com/kunichiko/MimicX-hardware) - 基板設計データ
