# コマンドリファレンス：Fins_Receive

## 概要

**Fins_Receive** は、`Fins_Send` により送信された要求電文に対する PLC からの **FINS 応答フレーム（UDP / TCP）または HostLink 応答電文を受信し、終了コード（および FCS）を検証した上で応答データ部を抽出格納** する汎用受信用コマンドです。
抽出したペイロードは、第 2 引数 `Format` の指定に従い `"HEX"`（既定）、`"DEC"`、または `"ASCII"` 形式へ変換して `DestKey` に格納します。

---

## 💡 書式・パラメータ仕様

### コマンド構文

```csv
,SEQUENCE, [ステップキー], Fins_Receive, [DestKey], "[Format: HEX]", "[Reserved]"
```

### パラメータ（引数）一覧

| 引数位置 | 引数名 | 必須/任意 | 説明・制約 |
| :--- | :--- | :---: | :--- |
| 第 1 引数 | `DestKey` | **必須** | 抽出された応答データ部を格納するパラメータキー名（`"@"` や `"$"` を付けない素のキー名、例: `"CPU_STATUS"` または `"PLC_MODEL"`）。 |
| 第 2 引数 | `Format` | 任意 | 抽出データの整形方式（`"DEC"`: Big-Endian 16 ビット符号なし 10 進カンマ区切り、`"ASCII"`: 末尾の NUL・空白・改行を除去した ASCII 文字列、それ以外［既定 `"HEX"`］: 連続大文字 16 進文字列）。 |
| 第 3 引数 | `Reserved` | 任意 | 互換用の予備引数（`ValidateParamRange(1, 3)` により 3 引数まで許容されますが、指定された値は参照されません）。 |

---

## 🔄 特殊置換規約・分岐規約の適用

本コマンドの引数における特殊記法および解釈ルールです。
* **動的置換（`$`）**: 引数内で `$PARAMキー$` のように記述すると、実行時に変数プール（PARAM）の値へ動的置換されます。（適用可能な引数: 全引数対応）
* **ジャンプ先指定（`#`）**: 本コマンドはジャンプ制御を行わないため適用されません。（適用可能な引数: 非適用）
* **共通コマンド指定（`@`）**: 本コマンドは外部マクロを呼び出さないため、共通コマンド呼び出し記法は適用されません。（適用可能な引数: 非適用）
* **カンマのエスケープ**: 引数自体にカンマを含める場合は、必ず `"..."` と二重引用符で囲んで記述します。
* **エイリアス（別名）**: `Fins_R`, `Fins_Res_R`, `Analysis_Fins_Receive`

---

## 💡 動作例（正しい構文での記述例）

### スクリプト記述例

※COMMAND ブロック配下の子要素行（PARAM, SEQUENCE）は必ず行頭カンマから開始し、スペースインデントや行末コメント（`;`）は記述しないでください。

```csv
COMMAND, "SAMPLE_FINS_RECEIVE", "CPU_STATUS"
,PARAM, "CPU_STATUS", Work, "CPUステータス結果", ""
; 1. 接続開始
,SEQUENCE, 010, Fins_Open, "192.168.250.1", "9600", "UDP"
; 2. CPU ステータス読出（0601）を送受信し CPU_STATUS へ格納
,SEQUENCE, 020, Fins_Send, "0601"
,SEQUENCE, 030, Fins_Receive, "CPU_STATUS", "HEX"
; 3. 取得ステータスが空文字でないか判定（空の場合は #NG_PROC# へ分岐）
,SEQUENCE, 040, Judge, "NEQ", "$CPU_STATUS$", "", "#NG_PROC#"
,SEQUENCE, 050, Fins_Close
,SEQUENCE, 060, Exit, "$CPU_STATUS$", 0
; 条件不成立時のクローズ＆異常終了経路
,SEQUENCE, #NG_PROC#, Fins_Close
,SEQUENCE, 080, Exit, "NG", -1
```

### 実行時の挙動・変数変化

* **実行前の状態**: `Fins_Send` により要求電文が送出され、変数プール（`PARAM`）に `"CPU_STATUS"` が定義されている状態。
* **実行後の状態**: `Fins_Receive` により応答フレームの終了コード（正常 `0x0000`）が検証され、抽出されたステータスデータが大文字 16 進文字列として `"CPU_STATUS"` に格納されます。後続の `Judge` で非空が確認されると回線を閉じて正常終了し、空文字時は `#NG_PROC#` へ分岐してクローズ後にエラー終了します（※無応答や異常終了コード時はステップ `030` で即停止します）。

---

## 🔄 動作フロー（シーケンス図）

```mermaid
sequenceDiagram
    autonumber
    participant Driver as "TermSequenceDriver (実行エンジン)"
    participant Cmd as "Fins_ReceiveCommand"
    participant Trans as "FinsTransportClient.Shared"
    participant Engine as "FinsFrameEngine"
    participant Buf as "変数プール (PARAM / SBuf)"

    Driver->>Driver: ライセンス検証 (ValidateStep)
    Driver->>Cmd: Func(parent, param)
    activate Cmd
    Cmd->>Cmd: ValidateParamRange(1, 3) ＆ $PARAM$ 展開
    Cmd->>Trans: ReadCachedOrBufferResponse(param)
    alt 応答データが空 (無応答) の場合
        Cmd->>Buf: DestKey へ "ERROR_TIMEOUT" を格納
        Cmd-->>Driver: 受信失敗 (IsCommandGo = false)
    else 応答バイト列を取得した場合
        Cmd->>Engine: TryParseResponse(respBytes)
        alt 終了コード正常 (0x0000)
            Cmd->>Cmd: Format ("DEC" / "ASCII" / "HEX") に従いペイロード整形
            Cmd->>Buf: DestKey および RecData へ整形結果を格納
            Cmd-->>Driver: 正常完了 (IsCommandGo = true)
        else フレーム不正・FCS不一致・異常終了コードの場合
            Cmd->>Buf: DestKey へエラー識別文字列を格納
            Cmd-->>Driver: 異常応答停止 (IsCommandGo = false)
        end
    end
    deactivate Cmd
```

---

## 🚫 エラーハンドリング・安全設計（仕様制限）

入力データの異常や実行時エラーに対し、解析・実行エンジンが実行するセーフティ規約です。

1. **引数範囲検証および例外防御**:
   - 引数が 0 個または 4 個以上の場合は `ValidateParamRange(1, 3)` により `_log.LogError` と `SetErrorAndHalt` で停止します。また、受信データ変換中に予期せぬ例外が発生した場合は `catch` 節で `_log.LogCritical` に記録し、`SetErrorAndHalt(-1, ex.Message)` で安全停止します。
2. **無応答タイムアウトおよび FINS／HostLink 異常終了コードの検出**:
   - 受信データが空の場合は `DestKey` に `"ERROR_TIMEOUT"` を、FCS 不一致や FINS／HostLink 異常終了コード（非 `0x0000`）の場合はエラー識別文字列を格納し、`param.IsCommandGo = false` として後続ステップへの誤進行を防止します。

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
| **2** | 異常系 | 引数 0 個（`new TermSequence { Name = "Fins_Receive" }`）で `Func(null, invalidParam)` を実行 | `ValidateParamRange(1, 3)` により拒否され、`invalidParam.HasError == true` かつ `!invalidParam.IsCommandGo` となる | 必須引数 `DestKey` 欠落時のバリデーション拒否検証 |
