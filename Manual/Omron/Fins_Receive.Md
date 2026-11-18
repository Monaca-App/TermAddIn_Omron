# コマンドリファレンス：Fins_Receive

## 概要

**Fins_Receive** は、`Fins_Send` により送信された要求電文に対する PLC からの **FINS 応答フレーム（UDP / TCP）または HostLink 応答電文を受信し、終了コード（および FCS）を検証した上で応答データ部を抽出格納** する汎用受信用コマンドです。

---

## 💡 書式・パラメータ仕様

### コマンド構文

```csv
,SEQUENCE, [ステップキー], Fins_Receive, "[DestKey]", "[Format]"
```

### パラメータ（引数）一覧

| 引数位置 | パラメータ名 | 必須/任意 | 型・制約 | 説明 |
| :--- | :--- | :---: | :--- | :--- |
| 第 1 引数 | `DestKey` | 必須 | 文字列（例: `"@RAW_RES"` または `"PLC_MODEL"`） | 抽出された応答データ部を格納する結果キー／変数名。 |
| 第 2 引数 | `Format` | 任意 | `"HEX"`, `"DEC"`, `"ASCII"`（既定: `"HEX"`） | 抽出データの整形方式。`"HEX"` は連続16進文字列、`"DEC"` は Big-Endian 16ビット10進カンマ区切り、`"ASCII"` は形名等の文字列変換。 |

---

## 🔄 特殊置換規約・分岐規約の適用

本コマンドの引数における特殊記法および解釈ルールです。
* **動的置換（`$`）**: 引数内で `$PARAMキー$` のように記述すると、実行時に変数プール（PARAM）の値へ動的置換されます。（適用可能な引数: 全引数対応）
* **ジャンプ先指定（`#`）**: 本コマンドはジャンプ制御を行わないため適用されません。（適用可能な引数: 非適用）
* **共通コマンド指定（`@`）**: 本コマンドは外部マクロを呼び出さないため、共通コマンド呼び出し記法は適用されません。（適用可能な引数: 非適用）
* **カンマのエスケープ**: 通信条件文字列（例: `"9600,E,7,1"`）や複数データのカンマ区切り値を単一引数として渡す場合は、必ず `"..."` と二重引用符で囲んで記述します。
* **エイリアス（別名）**: `Fins_R`, `Fins_Res_R`, `Analysis_Fins_Receive`

---

## 💡 動作例（正しい構文での記述例）

### スクリプト記述例

※COMMAND ブロック配下の子要素行（PARAM, SEQUENCE）は必ず行頭カンマから開始し、スペースインデントや行末コメント（`;`）は記述しないでください。

```csv
COMMAND, "SAMPLE_FINS_RECEIVE", ""
; 1. 接続開始
,SEQUENCE, 010, Fins_Open, "192.168.250.1", "9600", "UDP"
; 2. CPU ステータス読出（0601）を送受信
,SEQUENCE, 020, Fins_Send, "0601"
,SEQUENCE, 030, Fins_Receive, "@CPU_STATUS", "HEX"
; 3. 切断
,SEQUENCE, 040, Fins_Close
,SEQUENCE, 050, Exit
```

### 実行時の挙動・変数変化

* **実行前の状態**: Term 通信エンジンが初期化され、必要な通信パラメータおよび変数プール（`PARAM`）が準備されている状態。
* **実行後の状態**: ステップ実行により `Fins_Receive` の通信制御・プロトコル処理が正常に遂行され、結果または回線状態が更新されます。

---

## 🔄 動作フロー（シーケンス図）

```mermaid
sequenceDiagram
    autonumber
    participant Script as "シーケンス実行エンジン"
    participant Cmd as "Fins_ReceiveCommand"
    participant Trans as "Omron 通信トランスポート"
    participant Buf as "受信バッファ / 変数プール"

    Script->>Cmd: Func(parent, param)
    activate Cmd
    Cmd->>Cmd: ライセンス・パラメータ範囲検証 ($PARAM$ 展開)
    alt 引数および通信状態が正常な場合
        Cmd->>Trans: プロトコル電文送受信 / 回線制御実行
        Trans-->>Cmd: 正常応答フレーム / ステータス返却
        Cmd->>Buf: 結果格納・状態更新 (IsCommandGo = true)
        Cmd-->>Script: 正常完了 (次ステップへ進行)
    else パラメータ不正・タイムアウト・異常応答の場合
        Cmd->>Cmd: エラーログ出力 (_log.LogError)
        Cmd-->>Script: 安全停止またはエラー結果格納
    end
    deactivate Cmd
```

---

## 🚫 エラーハンドリング・安全設計（仕様制限）

入力データの異常や実行時エラーに対し、解析・実行エンジンが実行するセーフティ規約です。

1. **引数範囲・プロトコル書式バリデーション保護**:
   - 必須引数の欠落、デバイス表記・数値範囲の不正、または余剰引数が指定された場合は `_log.LogError` に詳細を記録し、`SetErrorAndHalt` により異常電文の送出を未然に遮断します。
2. **通信タイムアウト・異常応答・排他ドメイン保護**:
   - 相手機器の無応答タイムアウト、NAK／異常終了コード応答、または他通信ドメインとの排他競合が発生した場合は、例外クラッシュを起こさず安全にエラーハンドリングを行います。

---

## 🧪 ユニットテスト仕様

本コマンドクラス（`Fins_ReceiveCommand.cs`）に実装されている `RunUnitTest()` メソッドが検証するユニットテストの仕様です。

### テスト実行前提条件

* **実行トリガー**: `RunUnitTest()` メソッドの呼び出し
* **有効化条件**: システム全体の最小ログレベル（`ErrorLogger.MinimumLevel`）が `LogLevel.Test` であること（それ以外の場合は無効化ガードにより早期リターン）
* **検証結果出力**: `_log.LogTest` によるログ出力。検証失敗時は `_log.LogError`、予期せぬ例外発生時は `_log.LogCritical` で記録し、`finally` ブロックで `FinsTransportClient.Shared.IsSimulationMode` を開始前の状態へ復元します。

### テストケース一覧

| No | テストカテゴリ | テスト条件（入力値） | 期待される結果（出力値） | 検証の目的 / 観点 |
| :---: | :--- | :--- | :--- | :--- |
| **1** | 正常系 | `IsSimulationMode = true` 環境下で引数 `"MODEL_STR"`, `"ASCII"` および擬似 FINS `0501` 形名応答（ヘッダ `C0 00 02 00 01 00 00 00 00 01 05 01 00 00` ＋ `"CJ2M-CPU31          "`）を与えて実行 | `!param.HasError` かつ `param.IsCommandGo == true`、格納先 `SBuf["MODEL_STR"]` が末尾空白除去済みの `"CJ2M-CPU31"` と一致する | FINS 正常終了コード検証および ASCII ペイロード抽出・トリム格納の検証 |
| **2** | 異常系 | 必須引数 `DestKey` 欠落・引数範囲外（`ValidateParamRange(1, 3)` 違反）、空応答受信、または FINS／HostLink 異常終了コード応答 | 空応答時は `DestKey` に `"ERROR_TIMEOUT"`、異常終了コード時はエラー文字列を格納して `IsCommandGo = false` で安全停止する | 引数境界検証および無応答・FINS エラー応答時のフェイルセーフ検証 |
