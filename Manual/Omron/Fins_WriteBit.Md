# コマンドリファレンス：Fins_WriteBit

## 概要

**Fins_WriteBit** は、OMRON FINS メモリエリア書込コマンド（ビット単位：`MRC=01`, `SRC=02`）を用いて、PLC の **ビットデバイス（CIO, WR, HR, AR, DM, EM, TIM, CNT）へ ON/OFF 状態を一括書込** する通信制御系コマンドです。
書き込み完了後、PLC からの FINS／HostLink 応答終了コード（正常: `0x0000`）を検証します。

---

## 💡 書式・パラメータ仕様

### コマンド構文

```csv
,SEQUENCE, [ステップキー], Fins_WriteBit, [Device], [Values], "[Reserved]"
```

### パラメータ（引数）一覧

| 引数位置 | 引数名 | 必須/任意 | 説明・制約 |
| :--- | :--- | :---: | :--- |
| 第 1 引数 | `Device` | **必須** | 書き込みを開始する先頭ビットアドレス（対応エリア: `CIO`［接頭辞省略可］, `WR`/`W`, `HR`/`H`, `AR`/`A`, `DM`/`D`, `EM`/`E`/`E0_`〜`EC_` の `.00`〜`.15` ビット指定、および `TIM`/`T`, `CNT`/`C` 完了フラグ。例: `"CIO10.00"`, `"W0.05"`, `"H1.00"`, `"DM100.00"`）。 |
| 第 2 引数 | `Values` | **必須** | 書き込むビット状態列（`"1"`/`"ON"`/`"TRUE"` が ON、それ以外が OFF。カンマまたは空白区切りで指定し、要素数から書込点数が自動算出されます。例: `"1, 0, 1"` または `"ON, OFF, ON"`）。 |
| 第 3 引数 | `Reserved` | 任意 | 互換用の予備引数（`ValidateParamRange(2, 3)` により 3 引数まで許容されますが、指定された値は参照されません）。 |

---

## 🔄 特殊置換規約・分岐規約の適用

本コマンドの引数における特殊記法および解釈ルールです。
* **動的置換（`$`）**: 引数内で `$PARAMキー$` のように記述すると、実行時に変数プール（PARAM）の値へ動的置換されます。（適用可能な引数: 全引数対応）
* **ジャンプ先指定（`#`）**: 本コマンドはジャンプ制御を行わないため適用されません。（適用可能な引数: 非適用）
* **共通コマンド指定（`@`）**: 本コマンドは外部マクロを呼び出さないため、共通コマンド呼び出し記法は適用されません。（適用可能な引数: 非適用）
* **カンマのエスケープ**: 複数データのカンマ区切り値（例: `"1, 0, 1"`）を単一引数として渡す場合は、必ず `"..."` と二重引用符で囲んで記述します。
* **エイリアス（別名）**: `Fins_Write_Bit`, `Analysis_Fins_WriteBit`

---

## 💡 動作例（正しい構文での記述例）

### スクリプト記述例

※COMMAND ブロック配下の子要素行（PARAM, SEQUENCE）は必ず行頭カンマから開始し、スペースインデントや行末コメント（`;`）は記述しないでください。

```csv
COMMAND, "SAMPLE_FINS_WRITEBIT", ""
; 1. 接続開始
,SEQUENCE, 010, Fins_Open, "192.168.250.1", "9600", "UDP"
; 2. W10.00〜W10.02 に ON, OFF, ON を一括書込
,SEQUENCE, 020, Fins_WriteBit, "W10.00", "1, 0, 1"
; 3. 切断
,SEQUENCE, 030, Fins_Close
,SEQUENCE, 040, Exit
```

### 実行時の挙動・変数変化

* **実行前の状態**: `Fins_Open` により OMRON PLC（`192.168.250.1:9600`, UDP）との通信回線が開設されている状態。
* **実行後の状態**: ステップ `020` で FINS メモリエリア書込（`MRC=01`, `SRC=02`、ビット単位 3 点）フレームが送受信され、終了コード `0x0000` が確認されると `param.IsCommandGo = true` となってステップ `030` の `Fins_Close` へ進みます。

---

## 🔄 動作フロー（シーケンス図）

```mermaid
sequenceDiagram
    autonumber
    participant Driver as "TermSequenceDriver (実行エンジン)"
    participant Cmd as "Fins_WriteBitCommand"
    participant Engine as "FinsDeviceParser / FinsFrameEngine"
    participant Trans as "FinsTransportClient.Shared"

    Driver->>Driver: ライセンス検証 (ValidateStep)
    Driver->>Cmd: Func(parent, param)
    activate Cmd
    Cmd->>Cmd: ValidateParamRange(2, 3) ＆ $PARAM$ 展開
    Cmd->>Engine: Parse(devStr) ＆ ParseBitValues(valsStr) ＆ BuildBitWriteFrame
    Cmd->>Trans: TransceiveFinsFrame(reqFrame, param)
    alt 正常 FINS / HostLink 応答 (終了コード 0x0000)
        Cmd-->>Driver: 正常完了 (IsCommandGo = true)
    else 無応答タイムアウト または FINS 異常終了コード応答
        Cmd->>Cmd: SetErrorAndHalt(-1, errMsg)
        Cmd-->>Driver: 異常停止 (IsCommandGo = false)
    else 引数範囲外 または デバイス表記不正例外
        Cmd->>Cmd: _log.LogError / _log.LogCritical ＆ SetErrorAndHalt
        Cmd-->>Driver: 異常停止 (IsCommandGo = false)
    end
    deactivate Cmd
```

---

## 🚫 エラーハンドリング・安全設計（仕様制限）

入力データの異常や実行時エラーに対し、解析・実行エンジンが実行するセーフティ規約です。

1. **引数範囲およびデバイス構文の検証**:
   - 引数が 1 個以下または 4 個以上の場合は `ValidateParamRange(2, 3)` により `_log.LogError` と `SetErrorAndHalt` で遮断します。未対応エリアやビット位置不正により `FinsDeviceParser.Parse` が例外を送出した場合は、`catch` 節で `_log.LogCritical` に記録し `SetErrorAndHalt(-1, ex.Message)` で安全停止します。
2. **書込応答タイムアウトおよび FINS 異常終了コード時の停止**:
   - 受信データが空（タイムアウト）の場合、または PLC が非 `0x0000` の FINS 異常終了コードや不正フレームを返した場合は、`SetErrorAndHalt(-1, ...)` を呼び出してシーケンスを即座に安全停止します。

---

## 🧪 ユニットテスト仕様

本コマンドクラス（`Fins_WriteBitCommand.cs`）に実装されている `RunUnitTest()` メソッドが検証するユニットテストの仕様です。

### テスト実行前提条件

* **実行トリガー**: `RunUnitTest()` メソッドの呼び出し
* **有効化条件**: システム全体の最小ログレベル（`ErrorLogger.MinimumLevel`）が `LogLevel.Test` であること（それ以外の場合は無効化ガードにより早期リターン）
* **検証結果出力**: `_log.LogTest` によるログ出力。検証失敗時は `_log.LogError`、予期せぬ例外発生時は `_log.LogCritical` で記録し、`finally` ブロックで `FinsTransportClient.Shared.IsSimulationMode` を開始前の状態へ復元します。

### テストケース一覧

| No | テストカテゴリ | テスト条件（入力値） | 期待される結果（出力値） | 検証の目的 / 観点 |
| :---: | :--- | :--- | :--- | :--- |
| **1** | 正常系 | `IsSimulationMode = true` 環境下で引数 `"W10.05"`, `"1,0,1"` および擬似 FINS `0102` 正常応答（`C0 00 02 00 01 00 00 00 00 01 01 02 00 00`）を与えて実行 | `!param.HasError` かつ `param.IsCommandGo == true` で 3 点ビット一括書込が正常完了する | ビットアドレス指定での FINS 0102 ビット書込フレーム構築および正常応答検証 |
| **2** | 異常系 | 引数 0 個（`new TermSequence { Name = "Fins_WriteBit" }`）で `Func(null, invalidParam)` を実行 | `ValidateParamRange(2, 3)` により拒否され、`invalidParam.HasError == true` かつ `!invalidParam.IsCommandGo` となる | 必須引数欠落時のバリデーション拒否検証 |
