# AI implement requirement

## 目的
`human-requirement.md` の内容を、実装作業に着手できる具体仕様へ分解する。

## 1. アーキテクチャ
- 採用スタックは **Tauri + React(TypeScript) + Rust** とする。
- 役割分担:
  - UI/キャンバス/操作系（ズーム、パン、スナップ、複数選択、Undo/Redo）: React(TypeScript)
  - ローカルファイルI/O、重い検証処理、エクスポート処理: Rust（Tauriコマンド）
  - デスクトップアプリ基盤（Windows中心、将来Linux/macOS対応）: Tauri
- OSごとの別実装は行わず、共通コードベースで管理する。

## 2. データモデル（JSON）
単一プロジェクト内に以下のJSONを保持する。

### 2.1 electrical.json（電気接続レイヤー）
- 目的: パーツ間の電気接続、信号、ピン到達性の表現。
- 主構造（例）:
  - `connectors`: コネクタ定義（id, partRef, pins）
  - `wires`: 電線定義（id, from, to, wireSpecRef, signalRef）
  - `signals`: 信号定義（id, name, class）
  - `nets`: 論理接続集合（id, memberPins）
- 必須要件反映:
  - 「どのコネクタの何番ピンがどこに繋がるか」を追跡可能な参照構造を持つ。

### 2.2 physical.json（物理レイヤー）
- 目的: 長さ、経路、束線・ツイスト・固定情報の管理。
- 主構造（例）:
  - `wireSegments`: 配線セグメント（wireId, path, lengthMm）
  - `bundles`: 束線情報（id, memberWires, type）
  - `twists`: ツイスト条件（bundleId or wireId, pitchMm）
  - `fixings`: タイバンド等（id, position, targetBundle）

### 2.3 parts.json（部品情報）
- 目的: 図面内で使用する部品の自己完結保持。
- 主構造（例）:
  - `instances`: 配置済み部品（instanceId, masterPartId, transform）
  - `specs`: 必須スペックのスナップショット（ピン数、許容電流、嵌合相手、型番 等）
  - `symbols`: 描画シンボル参照または埋め込み情報
- 必須要件反映:
  - 参照ライブラリ削除後も図面が崩れないよう、必要情報を保持する。

### 2.4 rules.json（適合ルール）
- 目的: チェックエンジンが実行する検証条件を宣言。
- 主構造（例）:
  - `wireGaugeTerminalMatch`
  - `wireLengthLimit`
  - `withstandVoltage`
  - `connectorMating`
  - `connectorCurrentLimit`
- 各ルールに `enabled`, `severityDefault`, `params` を持たせる。

### 2.5 bom.json（BOMデータ）
- 目的: CSV出力元データの正規形。
- 主構造（例）:
  - `items`: [{partNumber, description, quantity, unit, category, refs}]
  - `generatedAt`, `sourceRevision`

## 3. 単一ファイル形式
- 拡張子付き独自単一ファイル（実体はzipコンテナ）を採用する。
- コンテナ内の基本構成:
  - `manifest.json`（schemaVersion, appVersion, createdAt, updatedAt）
  - `electrical.json`
  - `physical.json`
  - `parts.json`
  - `rules.json`
  - `bom.json`（必要に応じ再生成可能）
  - `assets/`（図面テンプレート、埋め込みシンボル等）
- `schemaVersion` 方針:
  - 読込時に `schemaVersion` を確認。
  - 非互換時は読込拒否またはマイグレーション導線を表示。
  - 下位互換可能な更新はマイグレーションして保存時に最新化する。
- 読み書き方針:
  - 読込: zip展開（メモリまたは一時領域）→ JSON検証 → 内部モデル化。
  - 保存: 内部モデル → JSON生成 → zip再パック。
  - 破損対策として一時ファイル保存後に置換する。

## 4. チェックエンジン
- 判定レベルは **Error / Warning** の2段階。
- 実行タイミングは **リアルタイム + 手動実行** のハイブリッド。
  - リアルタイム: 編集イベント後に差分対象を優先チェック。
  - 手動: 全体再チェックを明示実行。
- 表示方式:
  - リスト表示（KiCad類似: 問題一覧）
  - 項目選択で該当オブジェクトをハイライト
- 初期対象ルール（human requirement準拠）:
  - 電線とコネクタターミナルの太さ適合
  - 電線長さ
  - 耐電圧
  - コネクタ嵌合
  - コネクタ許容電流

## 5. MVP範囲
- 優先実装:
  1. 2画面キャンバス（electrical/physical）
  2. 主要操作（ズーム、パン、グリッド吸着、スナップ、複数選択、Undo/Redo）
  3. 主要適合チェック（Error/Warning表示 + ハイライト）
  4. PDF出力
  5. BOM CSV出力
- 後回し:
  - 自動配置（オートプレース）
- 重点:
  - 配置自動化より、適合チェックと適合パーツの絞り込みを優先する。

## 6. ライブラリ連携方針（MVPで必要な範囲）
- ローカルフォルダライブラリを参照可能にする。
- 参照先更新検知時は、反映前にユーザー確認を行う。
- プロジェクト保存時は、参照ライブラリ由来の必要情報を `parts.json` / `assets/` に取り込み、単体ファイルで自己完結させる。

## 7. 非機能・運用要件
- オフライン前提で成立する設計とする。
- 将来拡張として、ネットワーク経由ライブラリ取得やKiCad連携を追加可能な構造を維持する。
