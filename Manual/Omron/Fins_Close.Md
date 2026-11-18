# コマンドリファレンス：Fins_Close

## 概要

**Fins_Close** は、`Fins_Open` によって開設された OMRON FINS / HostLink 通信回線（UDP ソケット、TCP ストリーム、またはシリアル COM ポート）を安全に切断し、OS リソースを解放するコマンドです。

---

## 💡 書式・パラメータ仕様

### コマンド構文

```csv
,SEQUENCE, [ステップキー], Fins_Close
```

### パラメータ（引数）一覧

| 引数位置 | 引数名 | 必須/任意 | 説明・制約 |
| :--- | :--- | :---: | :--- |
| （引数なし） | - | - | 接続中の OMRON FINS / HostLink 通信回線を安全に切断し解放します。 |

---

## 🔄 特殊置換規約・分岐規約の適用

本コマンドの引数における特殊記法および解釈ルールです。
* **動的置換（`$`）**: 本コマンドは引数を取らないため、引数に対する `$PARAMキー$` 動的置換は適用されません。（適用可能な引数: 非適用）
* **ジャンプ先指定（`#`）**: 本コマンドはジャンプ制御を行わないため適用されません。（適用可能な引数: 非適用）
* **共通コマンド指定（`@`）**: 本コマンドは外部マクロを呼び出さないため、共通コマンド呼び出し記法は適用されません。（適用可能な引数: 非適用）
* **カンマのエスケープ**: 通信条件文字列（例: `"9600,E,7,1"`）や複数データのカンマ区切り値を単一引数として渡す場合は、必ず `"..."` と二重引用符で囲んで記述します。
* **エイリアス（別名）**: `Analysis_Fins_Close`, `Sqn_Fins_Close`

---

## 💡 動作例（正しい構文での記述例）

### スクリプト記述例

※COMMAND ブロック配下の子要素行（PARAM, SEQUENCE）は必ず行頭カンマから開始し、スペースインデントや行末コメント（`;`）は記述しないでください。

```csv
COMMAND, "SAMPLE_FINS_CLOSE", ""
; 1. FINS/TCP で接続
,SEQUENCE, 010, Fins_Open, "192.168.250.1", "9600", "TCP"
; 2. DM200 を読出
,SEQUENCE, 020, Fins_ReadWord, "DM200", 1, "@DM200_VAL", "HEX"
; 3. 通信回線をクローズして終了
,SEQUENCE, 030, Fins_Close
,SEQUENCE, 040, Exit
```

### 実行時の挙動・変数変化

* **実行前の状態**: Term 通信エンジンが初期化され、必要な通信パラメータおよび変数プール（`PARAM`）が準備されている状態。
* **実行後の状態**: ステップ実行により `Fins_Close` の通信制御・プロトコル処理が正常に遂行され、結果または回線状態が更新されます。

---

## 🔄 動作フロー（シーケンス図）

```mermaid
sequenceDiagram
    autonumber
    participant Script as "シーケンス実行エンジン"
    participant Cmd as "Fins_CloseCommand"
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

本コマンドクラス（`Fins_CloseCommand.cs`）に実装されている `RunUnitTest()` メソッドが検証するユニットテストの仕様です。

### テスト実行前提条件

* **実行トリガー**: `RunUnitTest()` メソッドの呼び出し
* **有効化条件**: システム全体の最小ログレベル（`ErrorLogger.MinimumLevel`）が `LogLevel.Test` であること（それ以外の場合は無効化ガードにより早期リターン）
* **検証結果出力**: `_log.LogTest` によるログ出力。検証失敗時は `_log.LogError`、予期せぬ例外発生時は `_log.LogCritical` で記録し、`finally` ブロックで `FinsTransportClient.Shared.IsSimulationMode` を開始前の状態へ復元します。

### テストケース一覧

| No | テストカテゴリ | テスト条件（入力値） | 期待される結果（出力値） | 検証の目的 / 観点 |
| :---: | :--- | :--- | :--- | :--- |
| **1** | 正常系 | `FinsTransportClient.Shared.IsSimulationMode = true` 環境下で引数なしで `Fins_Close` を実行 | `FinsTransportClient.Shared.Close()` が実行され、`!param.HasError` かつ `param.IsCommandGo == true`、`!IsConnected` となる | FINS／HostLink 通信回線の正常切断および接続状態解除の検証 |
| **2** | 異常系 | 引数 2 個以上の指定（`ValidateParamRange(0, 1)` 違反）またはクローズ処理中の例外発生 | 引数範囲外エラー出力による早期リターン、または `_log.LogCritical` 記録と `SetErrorAndHalt(-1, ex.Message)` による安全停止 | 引数境界ガードおよび回線切断例外時のクラッシュ防御検証 |
