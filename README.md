# MimicX-protocol

**製品名: Mimic X**

USB-MIDIを利用した汎用HID制御通信プロトコルライブラリ。Dart（Flutter向け）とC/C++（マイコン向け）の実装を提供。

「Mimic X」プロジェクトの中核となる通信プロトコル仕様。

## 概要

スマートフォンとマイコン間のUSB-MIDI通信を抽象化し、キーボード・マウス・ジョイスティックなどのHIDデバイス制御メッセージを送受信するためのプロトコルを定義する。

## ドキュメント

- **[プロトコル仕様書](docs/protocol-spec.md)** — メッセージフォーマット、チャンネル割り当て、ネゴシエーション手順

## 設計方針

- **リアルタイム操作は素の MIDI メッセージ** (Note On/Off, Control Change) で最小レイテンシを実現
- **デバイス識別・設定は SysEx** で可変長の構造化データを送受信
- デバイスが自己申告するため、アプリ側は汎用的に対応可能

## 関連リポジトリ

- [MimicX-app](https://github.com/kunichiko/MimicX-app) - Flutterアプリ
- [MimicX-firmware](https://github.com/kunichiko/MimicX-firmware) - マイコンファームウェア
- [MimicX-hardware](https://github.com/kunichiko/MimicX-hardware) - 基板設計データ
