# Mimic X Protocol Specification

**Version:** 0.7.0 (Draft)
**Date:** 2026-06-01

## 変更履歴

- **0.7.0** (2026-06-01): 接続ライフサイクルとステータス LED の整理。
  - `HEART_BEAT` (0x08) を新設。ホストは 1 秒間隔で送信し、デバイスは ACK 応答 +
    内部状態を **CONNECTED** に保持する。3 秒間 HB が無いとデバイスは
    **WAITING** 状態に戻り、LED override も自動リセットされる
  - `DISCONNECT` (0x09) を新設。操作 / 編集画面から戻るときホストが明示送信する。
    デバイスは即座に `SCANNED` (緑) に戻り override をクリアする (HB タイムアウト
    の 3 秒を待たない)
  - ステータス LED (PB0 WS2812B) の状態モデルを定義:
    `WAITING`(黄) → `SCANNED`(緑) → `CONNECTED`(青) → `+activity`(青 4Hz 点滅)。
    遷移はデバイス側が自律的に行い (IDENTIFY 受信で SCANNED、HB 受信で CONNECTED、
    CONNECTED 中の Note/CC 受信で 500ms 青点滅)、ホストは `SET_LED` (0x20) と
    `SET_LED_BLINK` (0x21) で上書き (override) のみ行う
  - `SET_LED` で RGB=(255,255,255) を送ると **override reset** となり、LED は
    状態色に戻り点滅もクリアされる
  - `IDENTIFY_RESPONSE` に Chip UID 由来の **シリアル番号 (16 ASCII hex)** を
    追加。チャンネルマップ直後・デバイス名直前の固定 16 byte
- **0.6.0** (2026-05-24): `EMIT_REMOTE` (0x07) を追加。X68000 キーボードの REMOTE 端子から SHARP 12-bit リモコンコードをホストから発射可能に。SHIFT/OPT.2 + 特定キーによるリモコン発射はホスト側で検出し、本コマンドで送出する設計
- **0.5.0** (2026-05-20): 全 SysEx コマンドに **request ID** と **status** を導入。`SET_CONFIG`/`RESET` には新規 `ACK` (0x06) で応答する。`IDENTIFY_REQUEST`/`IDENTIFY_RESPONSE` のみ後方互換性のため旧フォーマットを維持 (ブートストラップ用途)。ホストはバージョン非対応を `IDENTIFY_RESPONSE` の `protocol_version` で判定する
- **0.4.0** (2026-05-05): ターゲット機からの受信バイトを SysEx TARGET_RX (0x05) で生バイト転送する方式に変更。Channel 14 の LED CC 通知 (5.1) は廃止し、解釈はアプリ側で行う
- **0.3.0** (2026-05-05): IDENTIFY_RESPONSE をチャンネルマップ方式に変更。1台のデバイスで複数 HID 種別 (各 MIDI チャンネルに別の HID を割り当て) をサポート
- **0.2.0** (2026-05-05): X68000 キーボード/マウスプロトコル詳細を追加 (Appendix A, C)、マウス CC 仕様確定、スキャンコード一覧拡充
- **0.1.0** (2026-05-01): 初版

## 1. 概要

Mimic X プロトコルは、USB-MIDI の上に構築されたレトロ PC 向け HID デバイス制御プロトコルである。
スマートフォン（ホスト）とマイコン（デバイス）間で、キーボード・ジョイスティック・マウスなどの HID 操作を双方向にやりとりする。

### 1.1. 設計原則

1. **USB-MIDI 標準準拠** — 標準的な USB-MIDI デバイスとして認識される。特殊なドライバ不要
2. **リアルタイム操作は素の MIDI メッセージ** — エンコード不要、最小レイテンシ
3. **ネゴシエーション・設定は SysEx** — 可変長の構造化データを送受信
4. **デバイス非依存** — ホスト（アプリ）はデバイスから自己申告された情報に基づいて UI を切り替える
5. **汎用性** — Mimic X 以外のプロジェクトでも USB-MIDI 制御プロトコルとして流用可能

### 1.2. 用語

| 用語 | 意味 |
|------|------|
| ホスト | スマートフォン（Flutter アプリ）。USB-MIDI ホスト側 |
| デバイス | マイコン（CH32X035 等）。USB-MIDI デバイス側 |
| ターゲット | デバイスが模倣する対象のレトロ PC |

## 2. トランスポート

### 2.1. USB-MIDI

- USB Full-Speed (12 Mbps)
- MIDI 1.0 over USB (USB-MIDI 1.0 spec)
- Bulk 転送（EP OUT: ホスト→デバイス、EP IN: デバイス→ホスト）
- Cable Number: 0（単一ポート）

### 2.2. メッセージの分類

| 分類 | MIDI メッセージ種別 | 用途 | 方向 |
|------|---------------------|------|------|
| リアルタイム入力 | Note On / Note Off | キー押下・解放、ボタン押下・解放 | ホスト→デバイス |
| リアルタイム入力 | Control Change | ジョイスティック軸、連続値 | ホスト→デバイス |
| 状態通知 | Control Change | LED 状態、デバイス状態 | デバイス→ホスト |
| ネゴシエーション | SysEx | デバイス識別、設定 | 双方向 |

## 3. MIDI チャンネル割り当て

| チャンネル | 用途 | 方向 |
|-----------|------|------|
| 0 (1) | ジョイスティック / ゲームパッド | ホスト→デバイス |
| 1 (2) | キーボード | ホスト→デバイス |
| 2 (3) | マウス | ホスト→デバイス |
| 3-13 | 予約（将来拡張） | — |
| 14 (15) | デバイス→ホスト: 状態通知 | デバイス→ホスト |
| 15 (16) | デバイス→ホスト: イベント通知 | デバイス→ホスト |

※ 括弧内は MIDI の慣習的な 1-origin 表記。本仕様では 0-origin を使用。

## 4. リアルタイムメッセージ

### 4.1. キーボード (Channel 1)

#### 4.1.1. キー押下

```
Note On, Channel 1
  note     = キーコード (0x00-0x7F, ターゲット機種のスキャンコード)
  velocity = 0x7F (固定)
```

#### 4.1.2. キー解放

```
Note Off, Channel 1
  note     = キーコード (0x00-0x7F)
  velocity = 0x00 (固定)
```

#### 4.1.3. 備考

- キーコードはターゲット機種固有のスキャンコードをそのまま使用する
- 例: X68000 では 0x01 = ESC, 0x02 = 1, ... (X68000 のキースキャンコード表に準拠)
- ホストアプリはネゴシエーションで取得したデバイスタイプに基づき、適切なキーマップを使用する
- 全キーのスキャンコードが 0x00-0x7F に収まらない機種は、将来 SysEx 拡張で対応する

### 4.2. ジョイスティック / ゲームパッド (Channel 0)

#### 4.2.1. デジタルボタン

```
Note On, Channel 0
  note     = ボタン番号 (0-127)
  velocity = 0x7F (固定)

Note Off, Channel 0
  note     = ボタン番号 (0-127)
  velocity = 0x00 (固定)
```

ボタン番号の標準割り当て:

| ボタン番号 | 名称 |
|-----------|------|
| 0 | 上 (Up) |
| 1 | 下 (Down) |
| 2 | 左 (Left) |
| 3 | 右 (Right) |
| 4 | ボタン 1 / A / トリガー |
| 5 | ボタン 2 / B |
| 6 | ボタン 3 / C |
| 7 | ボタン 4 / X |
| 8 | ボタン 5 / Y |
| 9 | ボタン 6 / Z |
| 10 | Start |
| 11 | Select |
| 12-127 | 拡張ボタン |

#### 4.2.2. アナログ軸（将来拡張用）

```
Control Change, Channel 0
  control = 軸番号 (下表参照)
  value   = 軸の値 (0-127, 64=中央)
```

| CC 番号 | 軸 |
|---------|-----|
| 0x30 | 左スティック X |
| 0x31 | 左スティック Y |
| 0x32 | 右スティック X |
| 0x33 | 右スティック Y |
| 0x34 | 左トリガー |
| 0x35 | 右トリガー |

### 4.3. マウス (Channel 2)

#### 4.3.1. ボタン

```
Note On, Channel 2
  note     = ボタン番号
  velocity = 0x7F

Note Off, Channel 2
  note     = ボタン番号
  velocity = 0x00
```

| ボタン番号 | 名称 |
|-----------|------|
| 0 | 左ボタン |
| 1 | 右ボタン |
| 2 | 中ボタン |
| 3-127 | 拡張 |

#### 4.3.2. 移動量 (差分)

マウス移動はホストが定期的に送信する 1 イベントあたりの差分。
MIDI の 7bit 制限のため、1 イベントあたり -64〜+63 の範囲で表現する。
連続的にイベントを送ることで大きな移動も表現できる。

```
Control Change, Channel 2
  control = 軸番号
  value   = 移動量 (オフセット表現: 64 = 0, 0 = -64, 127 = +63)
```

| CC 番号 | 軸 |
|---------|-----|
| 0x30 | dX (右が正) |
| 0x31 | dY (下が正) |
| 0x32 | スクロール (下スクロールが正) |

#### 4.3.3. 大きな移動量を送る場合

X68000 マウスは 1 レポートあたり -128〜+127 の符号付き 8bit。MIDI の 7bit を超える場合は、複数の CC を送ってデバイス側で累積する。

例: dX = +100 を送りたい場合
- CC 0x30 value=127 (= +63) を 1 回送信
- CC 0x30 value=101 (= +37) を 1 回送信
- 累積で +100 が反映される

## 5. 状態通知メッセージ (デバイス→ホスト)

### 5.1. ターゲット受信バイト通知 (SysEx TARGET_RX)

ターゲット機 → デバイス方向の制御線上で受信した生バイトを、デコードせずに
ホストへ転送する。LED 制御 / キーリピート設定 / LED 輝度など、ターゲット機が
規定するあらゆる制御コマンドの解釈はアプリ側に任せる。

詳細は §6 「SysEx メッセージ」の TARGET_RX (0x05) を参照。

> **設計メモ**: 旧版 (v0.3) では LED 状態を Channel 14 の Control Change で
> 個別に通知していたが、ターゲット機ごとに増えていく多様な制御コマンド
> (X68000 ならリピート遅延 0x60-0x6F, リピート間隔 0x70-0x7F, LED 輝度
> 0x54-0x57 など) に対応するため、生バイト転送に統一した。

### 5.2. デバイス状態通知 (Channel 15)

```
Control Change, Channel 15
  CC 0x00 = ターゲット接続状態 (0=未接続, 127=接続済み)
  CC 0x01 = エラーコード (0=正常)
```

## 6. SysEx メッセージ

### 6.1. フォーマット

プロトコル 0.5 以降、`IDENTIFY_REQUEST` / `IDENTIFY_RESPONSE` を除く全てのコマンド/応答は **request ID** (7bit) を含む。ホストがリクエストごとに 0x00〜0x7F の任意の値を採番し、デバイスは応答に同じ値をエコーする。複数リクエストが in-flight でも対応関係を一意に特定できる。

#### 6.1.1. 共通リクエスト形式 (ホスト→デバイス, 0.5+)

```
F0 7D 01 <command> <req_id> <payload...> F7
```

#### 6.1.2. 共通レスポンス形式 (デバイス→ホスト, 0.5+)

```
F0 7D 01 <rsp_cmd> <req_id> <status> <payload...> F7
```

- **Manufacturer ID `0x7D`**: MIDI 規格で非商用/教育用途に予約されている ID
- **Sub ID `0x01`**: Mimic X プロトコルの識別子
- **`req_id`**: 0x00-0x7F の任意の値。デバイスは応答に同じ値をエコーする
- **`status`**: 処理結果 (6.1.3 参照)

※ 将来商用化する場合は MMA (MIDI Manufacturers Association) に正式な Manufacturer ID を申請する。

#### 6.1.3. Status コード

| 値 | 名称 | 意味 |
|----|------|------|
| 0x00 | OK | 正常完了 |
| 0x01 | UNKNOWN_COMMAND | コマンド未対応 |
| 0x02 | UNKNOWN_KEY | SET_CONFIG/GET_CONFIG の key 未対応 |
| 0x03 | INVALID_VALUE | SET_CONFIG の value が範囲外/不正 |
| 0x7F | GENERIC_ERROR | その他のエラー |

#### 6.1.4. ACK の取り扱い

- ホストはリクエスト送信後、対応する応答 (ACK / 専用レスポンス) を `req_id` でマッチして待機する
- 推奨タイムアウト: **100 ms** (USB-MIDI Full-Speed)
- タイムアウト or `status != OK` の場合、ホストはユーザーにエラーを表示する
- `TARGET_RX` (デバイス発の非同期通知) は req_id を持たない

#### 6.1.5. 後方互換性

- 旧プロトコル (0.4 以下) のファームは req_id を解釈せず ACK も返さない。新アプリは `IDENTIFY_RESPONSE` で受信した `protocol_version` を `MIN_SUPPORTED_PROTOCOL` (= 0.5.0) と比較し、未満なら未対応として接続を拒否する
- `IDENTIFY_REQUEST` / `IDENTIFY_RESPONSE` だけはブートストラップ目的で旧フォーマット (req_id なし) のまま据え置き

### 6.2. コマンド一覧

| コマンド | 値 | 方向 | リクエスト形式 | レスポンス |
|---------|-----|------|---------------|-----------|
| IDENTIFY_REQUEST | 0x01 | ホスト→デバイス | レガシー (req_id なし) | IDENTIFY_RESPONSE |
| IDENTIFY_RESPONSE | 0x02 | デバイス→ホスト | レガシー (req_id なし) | — |
| CAPABILITY_REQUEST | 0x03 | ホスト→デバイス | `03 <req_id>` | CAPABILITY_RESPONSE |
| CAPABILITY_RESPONSE | 0x04 | デバイス→ホスト | `04 <req_id> <status> <tlv...>` | — |
| TARGET_RX | 0x05 | デバイス→ホスト | `05 <ch> <hi4> <lo4>` (非同期通知, req_id なし) | — |
| ACK | 0x06 | デバイス→ホスト | `06 <req_id> <status> <orig_cmd>` | — |
| EMIT_REMOTE | 0x07 | ホスト→デバイス | `07 <req_id> <code>` | ACK |
| HEART_BEAT | 0x08 | ホスト→デバイス | `08 <req_id>` | ACK |
| DISCONNECT | 0x09 | ホスト→デバイス | `09 <req_id>` | ACK |
| SET_CONFIG | 0x10 | ホスト→デバイス | `10 <req_id> <key> <value...>` | ACK |
| GET_CONFIG | 0x11 | ホスト→デバイス | `11 <req_id> <key>` | CONFIG_RESPONSE |
| CONFIG_RESPONSE | 0x12 | デバイス→ホスト | `12 <req_id> <status> <key> <value...>` | — |
| SET_LED | 0x20 | ホスト→デバイス | `20 <req_id> <R> <G> <B>` | ACK |
| SET_LED_BLINK | 0x21 | ホスト→デバイス | `21 <req_id> <speed>` | ACK |
| RESET | 0x7F | ホスト→デバイス | `7F <req_id>` | ACK |

### 6.3. IDENTIFY_REQUEST (0x01)

ホストがデバイスに対して自己申告を要求する。**ブートストラップ用途のため req_id は付与しない。**

```
F0 7D 01 01 F7
```

### 6.4. IDENTIFY_RESPONSE (0x02)

デバイスが自身の情報と各 MIDI チャンネルの割り当てを応答する。**req_id は付与しない (レガシーフォーマット)。**

ホストはここで受信した `protocol_version_major / minor` を、ホスト側の最低サポートバージョン (`MIN_SUPPORTED_PROTOCOL`) と比較する。未満なら以降の通信を打ち切り、ユーザーに「ファームウェアの更新が必要」と表示する。

```
F0 7D 01 02
  <protocol_version_major>    プロトコルバージョン (メジャー)
  <protocol_version_minor>    プロトコルバージョン (マイナー)
  <firmware_version_major>    ファームウェアバージョン (メジャー)
  <firmware_version_minor>    ファームウェアバージョン (マイナー)
  <firmware_version_patch>    ファームウェアバージョン (パッチ)
  <num_channels>              チャンネル割り当て数 (1〜16)
  <ch_a> <type_a> <target_a>  各チャンネルの割り当て (3 byte × N)
  <ch_b> <type_b> <target_b>
  ...
  <serial[16]>                Chip UID 由来のシリアル番号 (固定 16 byte ASCII hex 大文字)
  <device_name ...>           デバイス名 (ASCII, 可変長)
F7
```

#### 6.4.0. シリアル番号 (serial[16])

プロトコル 0.7 以降、チャンネルマップとデバイス名の間に **16 byte 固定** のシリアル番号フィールドが入る。MCU の Chip UID (CH32X035 では 64bit) をメモリ並びのバイト順で先頭から `[0-9A-F]` 大文字 16 文字に展開したもの。

- 例: Chip UID = `CD AB D8 2C 27 BD CC 95` → `"CDABD82C27BDCC95"`
- ホストアプリはこの値をアダプタ個体の **永続識別子** として使い、ユーザー定義の表示名 (ニックネーム) 等を端末ローカルに紐付けて保存する
- 値は不変。USB ID (`vendor:product:iSerial`) と異なり、ファーム再書込でも保持される
- プロトコル 0.6 以下のファームでは本フィールドが存在しないため、新アプリは `IDENTIFY_RESPONSE` の `protocol_version` を確認し、0.7 未満なら更新を促す



#### 6.4.1. チャンネル割り当て

`<ch> <type> <target>` の 3 byte ブロックを `num_channels` 個並べる。

- `<ch>`: MIDI チャンネル番号 (0-15)
- `<type>`: HID デバイスタイプ (下表)
- `<target>`: ターゲットシステム (下表)

#### 6.4.2. デバイスタイプ

| 値 | デバイスタイプ |
|----|---------------|
| 0x00 | Unknown |
| 0x01 | Keyboard |
| 0x02 | Joystick / Gamepad |
| 0x03 | Mouse |
| 0x10 | Custom / Generic I/O |

複合デバイス (例: キーボード+マウス) は **複数のチャンネル** に分けて割り当てる。

#### 6.4.3. ターゲットシステム

| 値 | ターゲットシステム |
|----|-------------------|
| 0x00 | Generic (汎用) |
| 0x01 | ATARI (2600/800/ST 互換ジョイスティック) |
| 0x02 | X68000 |
| 0x03 | PC-9801/PC-9821 |
| 0x04 | MSX |
| 0x05 | FM TOWNS |
| 0x06 | PC-8801 |
| 0x07 | Apple II |
| 0x08 | Commodore 64 |
| 0x09 | Amiga |
| 0x0A | ZX Spectrum |
| 0x10 | IBM PC/AT (PS/2) |
| 0x11 | IBM PC (XT) |
| 0x20-0x7F | 予約（将来拡張） |
| 0x40 | Mega Drive (6 ボタンファイティングパッド) |

#### 6.4.4. 例

**ATARI ジョイパッド単独 (Ch0):**
```
num_channels = 1
ch=0, type=2 (Joystick), target=1 (ATARI)
```

**X68000 キーボード+マウス (Ch0=KB, Ch1=Mouse):**
```
num_channels = 2
ch=0, type=1 (Keyboard), target=2 (X68000)
ch=1, type=3 (Mouse),    target=2 (X68000)
```

**複合デバイス (Ch0=ATARI Joystick, Ch1=X68000 Keyboard):**
```
num_channels = 2
ch=0, type=2 (Joystick), target=1 (ATARI)
ch=1, type=1 (Keyboard), target=2 (X68000)
```

### 6.5. CAPABILITY_REQUEST (0x03)

特定のデバイスタイプに応じた詳細な機能情報を要求する。

```
F0 7D 01 03 <req_id> F7
```

### 6.6. CAPABILITY_RESPONSE (0x04)

デバイスの詳細機能を応答する。TLV (Type-Length-Value) 形式で複数の機能を列挙する。

```
F0 7D 01 04 <req_id> <status>
  <cap_type> <cap_length> <cap_data ...>
  <cap_type> <cap_length> <cap_data ...>
  ...
F7
```

#### Capability Types

| Type | 説明 | Data |
|------|------|------|
| 0x01 | ボタン数 | 1 byte: ボタン数 |
| 0x02 | アナログ軸数 | 1 byte: 軸数 |
| 0x03 | LED 数 | 1 byte: LED 数 |
| 0x04 | LED 名称 | 1 byte: LED 番号 + ASCII 文字列 |
| 0x10 | キーコード範囲 | 2 bytes: min, max |
| 0x11 | キーマップ名 | ASCII 文字列 |
| 0x20 | 双方向通信対応 | 1 byte: 0=送信のみ, 1=双方向 |

### 6.6.1. ACK (0x06)

専用レスポンスを持たないリクエスト (`SET_CONFIG`, `RESET` 等) に対する汎用応答。

```
F0 7D 01 06 <req_id> <status> <orig_cmd> F7
```

- `<req_id>`: リクエストでホストが付与した値をエコー
- `<status>`: 6.1.3 の status コード
- `<orig_cmd>`: ACK 対象の元コマンド値 (例: `0x10` for SET_CONFIG)

### 6.6.2. TARGET_RX (0x05)

ターゲット機側からデバイスが受信した生の 1 byte をホストへ転送する。
LED 制御、キーリピート設定、LED 輝度など、ターゲット機固有の制御コマンド
の解釈はホスト (アプリ) 側で行う。

```
F0 7D 01 05
  <midi_channel>     送信元の MIDI チャンネル (0-indexed)
  <byte_high_nibble> データバイト上位 4 bit (0x00-0x0F)
  <byte_low_nibble>  データバイト下位 4 bit (0x00-0x0F)
F7
```

- 元のデータバイト = `(byte_high_nibble << 4) | byte_low_nibble`
- `midi_channel` は受信元の機能に対応するチャンネル (例: X68000 キーボードなら 0x01)
- 1 SysEx あたり 1 byte。連続する受信は連続する SysEx として送出される

**例: X68000 キーボードが LED 制御コマンド 0xC5 (`bit7=1`,
`bit6=1: 全角点灯`, `bit5..0: その他消灯`) を受信した場合**

```
F0 7D 01 05 01 0C 05 F7
```

ホスト側は `(0x0C << 4) | 0x05 = 0xC5` を復元し、`bit7=1` で LED コマンドと
判定し、各ビットから LED 状態を抽出する。

### 6.6.3. EMIT_REMOTE (0x07)

X68000 キーボードの REMOTE 端子から SHARP 12-bit 形式のリモコンコードを送出するようデバイスに要求する。応答は **ACK (0x06)**。

REMOTE 信号は X68000 純正カラーディスプレイテレビ (CZ-607D / CZ-614D 等) で受信され、TV チャンネル切替・音量・電源等の制御に用いられる。

実機キーボードでは SHIFT (および本体設定によっては OPT.2) + 特定キーで自動発射されるが、本コマンドにより任意のコードを発射できる。ホスト側で SHIFT/OPT.2 + キーを検出して本コマンドを呼び出すことを想定 (ファーム側に巨大な変換テーブルを置かないため)。X68000 本体からキーボードへの「専用ディスプレイ制御コマンド」(0x00-0x3F) 受信に対するファーム側自動発射とは独立した経路。

```
F0 7D 01 07 <req_id> <code> F7
```

| code | 意味 | 備考 |
|------|------|------|
| 0x01 | VOL_UP | 押し続けで最大音量 (リピート対象) |
| 0x02 | VOL_DOWN | 押し続けで最小音量 (リピート対象) |
| 0x03 | VOL_NORMAL | 標準音量 |
| 0x04 | CH_CALL | チャンネル番号表示 |
| 0x05 | RESET | テレビ画面リセット |
| 0x06 | MUTE | 音声ミュート (トグル) |
| 0x07 | POWER_ON | 電源 ON |
| 0x08 | TV_COMPUTER | テレビ ⇔ コンピュータ |
| 0x09 | VIDEO | テレビ ⇔ 外部入力 |
| 0x0A | CONTRAST_NORMAL | コントラスト標準 |
| 0x0B | CH_UP | チャンネル + (リピート対象, 12→1 巡回) |
| 0x0C | CH_DOWN | チャンネル − (リピート対象, 1→12 巡回) |
| 0x0D | POWER_OFF | 電源 OFF |
| 0x0E | POWER_TOGGLE | 電源 ON/OFF トグル |
| 0x0F | SUPERIMPOSE | スーパーインポーズ ⇔ 解除 |
| 0x10-0x1B | CH_1 - CH_12 | 直接選局 |
| 0x1C | TV | テレビ画面 |
| 0x1D | COMPUTER | コンピュータ画面 |
| 0x1E | SUPERIMPOSE_1 | スーパーインポーズ (コントラストダウン) |
| 0x1F | SUPERIMPOSE_2 | スーパーインポーズ (コントラストノーマル) |

範囲外コード (0x00, 0x20-0x7F) は `status=INVALID_VALUE` (0x03) で ACK を返す。

#### リピート挙動

1 リクエストにつきファームは 1 完全パケット (SHARP12 の data + 反転 data + トレーラー、約 100 ms) を 1 回送出する。

「押し続け」が必要なコード (VOL_UP/DOWN, CH_UP/DOWN) はホスト側でキーリピートタイマを動かして本コマンドを繰り返し送出すること。リピート間隔の目安は 100 ms 以上 (パケット長と同程度) を推奨。

### 6.6.4. HEART_BEAT (0x08)

ホストがアダプタとの接続維持を表明する。応答は **ACK (0x06)**。

```
F0 7D 01 08 <req_id> F7
```

#### 接続ライフサイクル

```
                   ┌──────────────┐
                   │   WAITING    │← (3s HB 無し / override も reset)
        ┌─────────→│ LED=黄       │
        │          └──────┬───────┘
        │                 │ IDENTIFY_REQUEST 受信
        │                 ↓
        │          ┌──────────────┐
        │          │   SCANNED    │
        │          │ LED=緑       │
        │          └──────┬───────┘
        │                 │ HEART_BEAT 受信
        │                 ↓
        │          ┌──────────────┐
        │          │  CONNECTED   │
        └──────────┤ LED=青       │
   3s HB 無し       │ (Note/CC 受信 │
                   │  時は青点滅)  │
                   └──────────────┘
```

- ホストは接続を維持したい間、**1 秒間隔** で HEART_BEAT を送り続ける
- ホストは ACK の応答を確認する。**3 回連続 (= 3 秒) で応答が無い場合**、
  ホストは接続失敗と判断してユーザーに通知し、デバイス一覧に戻る
- デバイスも同様に 3 秒間 HEART_BEAT を受信しなければ WAITING に戻る
- WAITING 復帰時、デバイスは LED override を自動的にクリアする (色は黄、点滅なし)
- IDENTIFY_REQUEST を任意の状態で受信すると SCANNED に戻る。ホストは
  「アダプタ利用中の rescan は接続を切る」前提で UI を構築する

### 6.6.5. DISCONNECT (0x09)

ホストがアダプタの選択を解除した (= 操作画面 / rename 画面から戻った) ことを明示する。応答は **ACK (0x06)**。

```
F0 7D 01 09 <req_id> F7
```

#### 動作

- デバイスは状態を即座に `SCANNED` (緑) に戻し、LED override (色 + 点滅) をクリアする
- HEART_BEAT タイムアウト (3 秒) で WAITING に落ちるのを待たずに済むため、UX 上「画面を抜けたらすぐ緑に戻る」見栄えになる
- ホストは本コマンドを送ったあと、HEART_BEAT 送信を停止する。次に同デバイスを選択するときは IDENTIFY → HEART_BEAT 再開でフローを再開する

#### 送信タイミング

- ジョイスティック/キーボード操作画面から戻るとき
- ニックネーム編集画面から戻るとき (override は自動 reset されるため、別途 `SET_LED(reset)` を送る必要はない)

### 6.7. SET_CONFIG (0x10)

デバイスの設定を変更する。応答は **ACK (0x06)**。

```
F0 7D 01 10 <req_id>
  <config_key>
  <config_value ...>
F7
```

| config_key | 説明 | Value |
|-----------|------|-------|
| 0x01 | キーリピート有効/無効 | 0=無効, 1=有効 |
| 0x02 | キーリピート速度 | 0-127 (遅い→速い) |
| 0x03 | ジョイスティック パッドモード | 0=ATARI, 1=MD 6B, 2=Libble Rabble (XPD-1LR) |

未知の key は `status=UNKNOWN_KEY` (0x02)、value 範囲外は `status=INVALID_VALUE` (0x03) で ACK を返す。

### 6.8. SET_LED (0x20) — 色 override

アダプタ基板上のステータス LED (PB0 接続の WS2812B-2020) の表示色を **override** する。応答は **ACK (0x06)**。

```
F0 7D 01 20 <req_id> <R> <G> <B> F7
```

| パラメータ | 値 | 説明 |
|-----------|------|------|
| `<R>` | 0x00-0x7F | 赤成分 (7bit)。デバイス側で `(v<<1) \| (v>>6)` により 8bit (0-255) に拡張 |
| `<G>` | 0x00-0x7F | 緑成分 (7bit) |
| `<B>` | 0x00-0x7F | 青成分 (7bit) |

#### 色 override セマンティクス

通常時、LED の色はアダプタの接続状態 (`WAITING`/`SCANNED`/`CONNECTED`) でデバイスが自律的に決める (6.6.4 のステートマシン)。本コマンドは「色を上書きしたい」状況 (例: ホストアプリでアダプタ個体に名前を付ける編集画面など) で送る。

- override は **`SET_LED` を最後に送った色** が維持される
- スケール後 RGB = `(255, 255, 255)` (= 全 7bit が `0x7F`) は **reset センチネル** として扱う。override が解除され、LED は状態色に戻り、`SET_LED_BLINK` も None にリセットされる
- HEART_BEAT が 3 秒途絶えてデバイスが `WAITING` に戻った際も、override は自動的に reset される (ホストは reset を意識せずに済む)
- 最大輝度の白を出したい場合は `(0x7E, 0x7E, 0x7E)` (= scaled 253) を送る。`0x7F` は reset 用に予約されているため使えない

### 6.9. SET_LED_BLINK (0x21) — 点滅速度 override

ステータス LED の点滅速度を **override** する。応答は **ACK (0x06)**。色は本コマンドでは変えない。

```
F0 7D 01 21 <req_id> <speed> F7
```

| `<speed>` | 名称 | 周期 |
|-----------|------|------|
| 0x00 | None | 点滅しない (色を常時点灯) |
| 0x01 | Slow | 1 Hz (500ms ON / 500ms OFF) |
| 0x02 | Mid  | 2 Hz (250ms ON / 250ms OFF) |
| 0x03 | High | 4 Hz (125ms ON / 125ms OFF) |

範囲外の値は `status=INVALID_VALUE` (0x03) で ACK が返る。

`SET_LED` で color override が掛かっているときのみ意味を持つ。color override が無い (状態色を表示中の) 状態では `SET_LED_BLINK` は値だけ保持されるが、画面には反映されない (CONNECTED + activity の自動青点滅が優先)。

#### ホスト側の運用ガイド

- 「編集中」「警告」など、ユーザーに注目させたい状態は `SET_LED(red)` + `SET_LED_BLINK(Slow)` の組合せ
- override を解除したいとき (編集終了など) は `SET_LED(255,255,255)` 1 発で十分。色も点滅もデバイス側が状態に応じて自動復帰する
- CONNECTED 状態の activity 点滅 (青 High) は **デバイスが Note/CC 受信を検知して自動的に出す**。ホストは何もする必要がない

### 6.10. RESET (0x7F)

デバイスを初期状態にリセットする。全キー/ボタンを解放状態にする。応答は **ACK (0x06)**。

```
F0 7D 01 7F <req_id> F7
```

## 7. 接続シーケンス

```
ホスト (スマホアプリ)                デバイス (マイコン)
  │                                    │
  │  ── USB-MIDI 接続確立 ──           │
  │                                    │
  │── SysEx: IDENTIFY_REQUEST ────────→│  (req_id なし: bootstrap)
  │                                    │
  │←── SysEx: IDENTIFY_RESPONSE ──────│  (req_id なし)
  │    protocol: 0.5                   │
  │    firmware: 0.6.0                 │
  │    num_channels: 2                 │
  │    [ch=0, Keyboard, X68000]        │
  │    [ch=1, Mouse, X68000]           │
  │    name: "mimic-x-x68k-kb"         │
  │                                    │
  │  (proto >= MIN_SUPPORTED_PROTOCOL  │
  │   を確認。未満なら未対応で中断)    │
  │                                    │
  │── CAPABILITY_REQUEST (req_id=1) ──→│
  │                                    │
  │←── CAPABILITY_RESPONSE ───────────│
  │    req_id=1, status=OK             │
  │    LED数: 7, LED名, キーコード範囲 │
  │                                    │
  │── SET_CONFIG (req_id=2, key=0x03,──→│  パッドモード = Libble Rabble
  │   value=2)                         │
  │                                    │
  │←── ACK (req_id=2, status=OK, ─────│
  │    orig_cmd=0x10)                  │
  │                                    │
  │  (アプリが該当 UI を表示)          │
  │                                    │
  │── Note On (CH1, note=0x01) ──────→│  ESC キー押下
  │── Note Off (CH1, note=0x01) ─────→│  ESC キー解放
  │                                    │
  │←── SysEx: TARGET_RX (0xF7) ───────│  本体からの LED コマンド (例: 0xF7)
  │    F0 7D 01 05 01 0F 07 F7         │  → アプリ側でデコード
  │                                    │
```

## 8. 7bit エンコーディング

SysEx データは MIDI の制約により各バイト 0x00-0x7F (7bit) でなければならない。
本プロトコルの現行仕様ではすべてのフィールドが 7bit に収まるよう設計されているが、
将来 8bit データ（バイナリ BLOB）を送る必要がある場合は以下のエンコーディングを使用する。

### 8.1. MIDI 標準 7bit エンコーディング

7 バイトの 8bit データを 8 バイトの 7bit データにエンコードする。

```
入力:  B0 B1 B2 B3 B4 B5 B6  (7 bytes, 8bit each)
出力:  H  L0 L1 L2 L3 L4 L5 L6  (8 bytes, 7bit each)

H = (B0[7] << 6) | (B1[7] << 5) | ... | (B6[7] << 0)
Ln = Bn & 0x7F
```

オーバーヘッド: 約 14.3% (7 bytes → 8 bytes)

## 9. バージョニング

- プロトコルバージョンは Semantic Versioning (MAJOR.MINOR) に従う
- MAJOR 変更: 後方互換性のない変更
- MINOR 変更: 後方互換性のある機能追加
- デバイスとホストは IDENTIFY_RESPONSE のバージョンを確認し、非互換の場合はユーザーに通知する

## Appendix A: X68000 キーボード スキャンコード一覧

参考: [taneken/USBKBD2X68K](https://github.com/taneken/USBKBD2X68K) 及び X68000 テクニカルリファレンス

### 上段 (記号・数字・BS)

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x01 | ESC | 0x02-0x0B | 1〜0 |
| 0x0C | - | 0x0D | ^ |
| 0x0E | ¥ | 0x0F | BS |

### 第二段 (TAB, QWERTY)

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x10 | TAB | 0x11-0x1A | Q W E R T Y U I O P |
| 0x1B | @ | 0x1C | [ |
| 0x1D | RETURN | | |

### 第三段 (ASDF...)

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x1E-0x26 | A S D F G H J K L | 0x27 | ; |
| 0x28 | : | 0x29 | ] |

### 第四段 (ZXCV...)

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x2A-0x33 | Z X C V B N M , . / | 0x34 | _ |
| 0x35 | Space | | |

### 編集・カーソルキー

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x36 | HOME | 0x37 | DEL |
| 0x38 | ROLLUP | 0x39 | ROLLDOWN |
| 0x3A | UNDO | 0x3B | ← |
| 0x3C | ↑ | 0x3D | → |
| 0x3E | ↓ | 0x3F | CLR |

### テンキー

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x40 | / (テンキー) | 0x41 | * |
| 0x42 | - (テンキー) | 0x43-0x45 | 7 8 9 |
| 0x46 | + | 0x47-0x49 | 4 5 6 |
| 0x4A | = | 0x4B-0x4D | 1 2 3 |
| 0x4E | ENTER (テンキー) | 0x4F | 0 |
| 0x50 | , (テンキー) | 0x51 | . |

### ファンクションキー

| コード | キー | コード | キー |
|--------|------|--------|------|
| 0x52 | 記号入力 | 0x53 | 登録 |
| 0x54 | HELP | 0x55-0x59 | XF1 XF2 XF3 XF4 XF5 |
| 0x5A | かな | 0x5B | ローマ字 |
| 0x5C | コード入力 | 0x5D | CAPS |
| 0x5E | INS | 0x5F | ひらがな |
| 0x60 | 全角 | 0x61 | BREAK |
| 0x62 | COPY | 0x63-0x6C | F1〜F10 |

### 修飾キー / 拡張

| コード | キー |
|--------|------|
| 0x70 | SHIFT |
| 0x71 | CTRL |
| 0x72 | OPT.1 |
| 0x73 | OPT.2 |

## Appendix B: ATARI ジョイスティック ボタンマッピング

ATARI 仕様 (DE-9) のジョイスティックのピンと、本プロトコルのボタン番号の対応:

| DE-9 ピン | 信号 | ボタン番号 |
|-----------|------|-----------|
| 1 | Up | 0 |
| 2 | Down | 1 |
| 3 | Left | 2 |
| 4 | Right | 3 |
| 6 | Button 1 (トリガー) | 4 |
| 9 | Button 2 (一部機種のみ) | 5 |
| 8 | GND | — |
| 7 | +5V | — |

## Appendix C: X68000 キーボード/マウスコネクタ

DIN 7pin (ミニ DIN) コネクタ。

| Pin | 信号 | 方向 (本体側) | 用途 |
|-----|------|---------------|------|
| 1 | Vcc2 (+5V) | OUT | キーボード/マウス電源 |
| 2 | MSDATA | IN | マウスデータ (4800bps 8N2) |
| 3 | KEY RxD | IN | キーボードへのコマンド (LED, リピート設定: 2400bps 8N1) |
| 4 | KEY TxD | OUT | キーコード送信 (2400bps 8N1) |
| 5 | READY | OUT | ホスト準備完了 (1=READY) |
| 6 | REMOTE | IN | TV リモコン (未使用) |
| 7 | GND | — | |

信号レベル: 5V TTL

### キーボード UART (Pin 3, 4)

- **2400 bps, 8N1**
- KBD → 本体: 1 byte/key
  - bit7 = 0:押下 (make), 1:解放 (break)
  - bit6-0 = スキャンコード (Appendix A 参照)
- 本体 → KBD: 制御コマンド (下表)

#### 本体 → キーボード制御コマンド一覧

| ビットパターン | 範囲 | 名称 | 意味 |
|---------------|------|------|------|
| `0b00xxxxxx` | 0x00-0x3F | TVCTRL | 専用ディスプレイ制御 (TV リモコンコード)。下位 6bit がコード。0x01-0x1F に定義あり (Appendix D) |
| `0b010010*X` | 0x48-0x4B | **KEY_EN** | キー送信許可。X=1 で送信可、X=0 で送信不可 (詳細下記) |
| `0b010100*X` | 0x50-0x53 | X68K_X1 | キー操作によるディスプレイ制御モード選択 (X68k / X1)。極性未検証 |
| `0b010101XX` | 0x54-0x57 | BRIGHT | キーボード LED 輝度。XX=00 最明 〜 XX=11 最暗 |
| `0b010110*X` | 0x58-0x5B | CTRL_EN | 本体発のディスプレイ制御コマンド (0x00-0x3F) の有効/無効。極性未検証 |
| `0b010111*X` | 0x5C-0x5F | OPT2_EN | OPT.2 キー + 特定キーによるディスプレイ制御の許可/禁止。X=0 で許可 (実機確認済み) |
| `0b0110dddd` | 0x60-0x6F | REPEAT_DELAY | キーリピート開始遅延 (200 + dddd × 100 ms) |
| `0b0111rrrr` | 0x70-0x7F | REPEAT_RATE | キーリピート間隔 (30 + rrrr² × 5 ms) |
| `0b1xxxxxxx` | 0x80-0xFF | LED | LED 制御。`1<全角><ひらがな><INS><CAPS><コード入力><ローマ字><かな>` (各 bit: 0=点灯, 1=消灯) |

`*` は don't care ビット。

#### KEY_EN (キー送信許可) の詳細

- **bit=1: キーデータ送信可** (通常状態)
- **bit=0: キーデータ送信不可**

KEY_EN=0 を受信したらキーボードは **キーコードの UART 送信を行ってはならない**。スキャンコードを内部キューに溜めるか、丸ごと破棄するかは実装依存。一方、**TV Control (REMOTE 端子) への信号送出は KEY_EN の影響を受けず継続して送出可能**。

運用上の理由: X68000 本体の CPU がキー受信を行わない状態 (例: 特定モード遷移中) に入ると、キーボード側 MCU の TX が `READY` 待ちで止まり、結果として同じ MCU が司る TV Control 信号の生成までブロックされてしまう。これを防ぐため、本体は事前に KEY_EN=0 をキーボードへ通知して TX をスキップさせ、TV Control だけを継続させる。本体 CPU が再び受け取り可能になったら KEY_EN=1 を送ってキー送信を再開させる。

### マウス UART (Pin 2)

- **4800 bps, 8N2** (ストップビット 2)
- 本体側からの **MSCTRL 信号 (HIGH→LOW エッジ)** をトリガに、3 byte 連続送信
- アイドル中の MSCTRL は HIGH。データ要求時に LOW にし、最終ビット送信後 ~2.56ms で HIGH に復帰
- 周期: 約 20 ms (50Hz)

#### 3 byte データフォーマット

| Byte | bit7 | bit6 | bit5 | bit4 | bit3 | bit2 | bit1 | bit0 |
|------|------|------|------|------|------|------|------|------|
| 0    | Y_UNF | Y_OVF | X_UNF | X_OVF | (予約) | (予約) | 右ボタン | 左ボタン |
| 1    | dX (signed 8bit, 右が正) |
| 2    | dY (signed 8bit, 下が正) |

- `X_OVF` / `Y_OVF`: 該当軸の移動量が +128 以上だった (= byte が +127 で飽和した)
- `X_UNF` / `Y_UNF`: 該当軸の移動量が -129 以下だった (= byte が -128 で飽和した)
- 移動量は前回送信からの差分。送信後リセット。
- ボタンビット割り当ては Inside X68000 正誤表 (<https://kg68k.github.io/InsideX68000-errata/>) に従い、bit0=左, bit1=右 とする。旧版の本文では逆になっているので注意。

#### MSCTRL 信号の入手

X68000 本体のキーボード端子では、MSCTRL は専用ピンとしては存在せず、**KEY RxD (Pin 3) のラインを兼用**するという情報がある（要実機検証）。

実装上は、KEY RxD を入力しつつ、立ち下がりエッジを EXTI で検知することで MSCTRL として機能させる。
