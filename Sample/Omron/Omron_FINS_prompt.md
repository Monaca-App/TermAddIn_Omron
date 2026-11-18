# オムロン PLC データ収集用 マクロスクリプト自動生成 AI指示書 (Omron_FINS.csv 生成用)

> **生成AI**: Antigravity (Gemini 4 Carbon (Medium))

あなたは、通信制御システム（Term / TermAddIn）向けマクロスクリプト（CSV形式）を設計・出力する専門のエンジニアリングAIです。  
以下の「参照ドキュメント」および「システム絶対制約」「FINS / HostLink 機能別パラメータ設計仕様」を厳格に遵守し、オムロン（OMRON）製 PLC（SYSMAC CS / CJ / CP / NJ / NX シリーズ）から **FINS プロトコル（FINS/UDP・FINS/TCP・シリアル HostLink）** で各種データを高速収集・制御するための Term マクロスクリプト（`Omron_FINS.csv`）を作成してください。

---

## 1. 参照ドキュメントおよび準拠規格

1. **システム文法・構文規約（最優先遵守）**
   * `Manual/TermAddIn/マクロスクリプト取扱説明書.Md`（Term マクロスクリプト取扱説明書 兼 AI自動生成指示書）
2. **通信・プロトコル個別仕様**
   * `Manual/TermAddIn/Omron/Fins_Mode.Md`（FINS 通信方式・ルーティング番号設定）
   * `Manual/TermAddIn/Omron/Fins_Open.Md` / `Manual/TermAddIn/Omron/Fins_Close.Md`（UDP/TCP/Serial 接続・切断）
   * `Manual/TermAddIn/Omron/Fins_ReadWord.Md`（ワード一括読出：コマンド `0101`）
   * `Manual/TermAddIn/Omron/Fins_WriteWord.Md`（ワード一括書込：コマンド `0102`）
   * `Manual/TermAddIn/Omron/Fins_ReadBit.Md` / `Manual/TermAddIn/Omron/Fins_WriteBit.Md`（ビット一括読出・書込）
   * `Manual/Core/Sequence/Judge.Md`（比較判定・条件分岐制御）
   * `Manual/Core/Sequence/Jump.Md` / `Manual/Core/Param/Exit.Md`（無条件ジャンプ・終了制御）
3. **対象通信仕様（実機エンジン `TermAnalysis.Omron` ランタイム引数規約）**
   * 制御対象: オムロン SYSMAC CS / CJ / CP / NJ / NX シリーズ（FINS/UDP・FINS/TCP 標準ポート `9600`、および RS-232C シリアル HostLink `9600,E,7,2`）
   * 接続条件設定:
     * `Fins_Mode, "[Protocol]", "[DestNet]", "[DestNode]", "[SrcNode]", "[UnitNo]"`（既定: `"UDP"`, DNA `"0"`, DA1 `"0"`, SA1 `"1"`, DA2/局番 `"0"`）
     * `Fins_Open, "[Endpoint]", "[PortOrOption]", "[Protocol]"`（Ethernet 例: `"192.168.250.1"`, `"9600"`, `"UDP"` ／ シリアル例: `"COM1"`, `"9600,E,7,2"`, `"Serial"`）
   * デバイスアクセス規約:
     * `Fins_ReadWord, "[Device]", "[Points]", "[DestKey]", "[Format]"`（`Format` は `"DEC"` または `"HEX"`。タイムアウト時は `"ERROR_TIMEOUT"`、異常終了コード時は `"ERROR_ENDCODE_xxxx"` を `DestKey` に格納）
     * `Fins_WriteWord, "[Device]", "[Values]"`（書込点数は `Values` の要素数から自動算出、終了コード `0x0000` を自動検証）
     * 対応デバイス記号: `DM`（または `D`）、`CIO`、`WR`（または `W`）、`HR`（または `H`）、`AR`（または `A`）、`EM`（およびバンク指定 `E0_`〜`EC_`）、`TIM`（`T`）、`CNT`（`C`）。

---

## 2. システム絶対制約（構文エラー・実行時クラッシュ防止の鉄則）

1. **表記言語・文字コード・AI生成元の明記**
   * **表記言語**: スクリプト内のコメント、パラメータ表示名、メッセージ等は **日本語をデフォルトとして記述・設定** してください。
   * **文字コード**: 生成する CSV ファイル（`Omron_FINS.csv`）は、日本語 Excel での文字化け防止のため **Shift-JIS (CP932)** ＋ **CRLF** で保存してください。
   * **AI生成元の明記（他者再現性の保証）**: CSV ファイルの最先頭ヘッダコメント行（`;`）に、対象システム情報とともに **実行環境（エージェント／IDE名）および自身のAIエンジン識別名（例: `; 生成AI: Antigravity (Gemini 4 Carbon (Medium))`）** を必ず記述してください。
   * **パス表記**: ファイル内に開発環境固有のローカル絶対パスを含めることは厳禁です。

2. **通信経路およびデバイス要素の完全機能別パラメータ分離と `List` 選択化**
   * 取り得る文字列が決まっている項目（`PROTOCOL`, `DEV_TYPE`）は必ず `List` 型で定義してください。
     * **通信プロトコル (`PROTOCOL`)**: `List`, 初期値 `"UDP"`, 追加選択肢 `"TCP"`, `"Serial"`
     * **接続先エンドポイント (`ENDPOINT`)**: `Text`, `"192.168.250.1"`（Serial 時は `"COM1"` 等を指定可能）
     * **ポートまたは通信条件 (`PORT_OPT`)**: `Text`, `"9600"`（Serial 時は `"9600,E,7,2"` 等を指定可能）
     * **宛先ネットワーク (`DEST_NET`)**: `Text`, `"0"`
     * **宛先ノード (`DEST_NODE`)**: `Text`, `"0"`
     * **自局ノード (`SRC_NODE`)**: `Text`, `"1"`
     * **ユニット・局番 (`UNIT_NO`)**: `Text`, `"0"`
     * **デバイス種別 (`DEV_TYPE`)**: `List`, 初期値 `"DM"`, 追加選択肢 `"CIO"`, `"WR"`, `"HR"`, `"AR"`, `"EM"`
     * **デバイスアドレス (`DEV_ADDR`, `ADDR_STATUS`, `ADDR_CTRL`)**: `Text`（例: `"100"`, `"101"`, `"200"`）
     * **アクセス点数 (`POINTS`)**: `Text`, `"1"`

3. **行頭判定・PARAM 必須フィールド・行末コメント禁止**
   * `TITLE`, `COMMAND` は行頭第1フィールドから記述し、`PARAM`, `SEQUENCE` は必ず行頭カンマ `,` から開始してください。
   * `PARAM` 行の第5フィールド（表示名）および第6フィールド（初期値）は省略厳禁です。
   * `COMMAND` 配下の `PARAM`・`SEQUENCE` 行に行末コメント（`;`）を含めてはなりません。

---

## 3. 対象システム仕様および TITLE ブロック定義

* **書式**:
  ```csv
  TITLE, "オムロンPLCデータ収集 (OMRON FINS / HostLink)"
  ```

---

## 4. 実装すべき COMMAND ブロック設計仕様

1. **`COMMAND, "PORT_OPEN"`**: `Fins_Mode` でルーティングを設定し、`Fins_Open` で UDP / TCP / Serial 回線を接続オープンする。
2. **`COMMAND, "READ_START", ON`**: 状態確認チャネル（`DM100`）のレディ値（`0`）を検証し、制御チャネル（`DM101`）へ開始要求値（`0x0001`）を書き込む。
3. **`COMMAND, "NUM_READ", "RESULT_DATA"`**: 計測データチャネル（`DM200`）から指定点数を 10進数（`DEC`）で読み出し、タイムアウト（`ERROR_TIMEOUT`）発生時は復旧コマンドを書いて再試行する。
4. **`COMMAND, "REG_READ", "RESULT_DATA"`**: 計測データチャネル（`DM200`）から指定点数を 16進数（`HEX`）で一括読み出しする。
5. **`COMMAND, "BIT_CHECK", "BIT_STATUS"`**: 運転ステータスビット（`CIO0.00`）から指定点数の ON/OFF 状態を `Fins_ReadBit` で読み出す。
6. **`COMMAND, "PORT_CLOSE"`**: 制御チャネル（`DM101`）を `0x0000` にリセットし、`Fins_Close` で回線を切断する。

---

## 5. 出力フォーマット

余計な前置きや解説を省き、すぐに CSV ファイル（`Omron_FINS.csv`）として保存できるよう、 **マークダウンのコードブロック（`csv` 形式）に作成した CSV コードのみ** を出力してください。

