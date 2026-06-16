# Requirements Document

## Project Description (Input)
本クレート（`wintf-winmsg-executor`）は Win32 FFI に生バインディングの `windows-sys`（0.61 系）を採用している。`HWND`/`WPARAM`/`LPARAM`/`LRESULT` などが生のポインタ・整数のまま扱われ、API 失敗時のエラーハンドリングも手書きの NULL/0 判定に依存している。これを Microsoft 公式の高水準寄りバインディングである `windows` クレート（0.62 系）へ移行し、newtype による型安全・`Result`/`?` 演算子によるエラー伝播・RAII・`s!` マクロによる文字列簡潔化の恩恵を受ける。移行は「慣用的移行（Result/RAII 活用）」方針とし、生 Win32 API を直接呼ぶ基本スタイル（`CreateWindowExA` 等、A 系 ANSI API）は維持しつつ、可能な箇所で unsafe 境界を縮小する。既存の公開 API の挙動は変えない。対象は `Cargo.toml` と 3 ファイル（`src/lib.rs`、`src/util/window.rs`、`src/util/msg_filter_hook.rs`）およびそれらのテストモジュール。

## Introduction
この機能は、`wintf-winmsg-executor` クレートの Win32 バインディングを `windows-sys`（0.61 系）から `windows`（0.62 系）へ置き換える、挙動非変更の内部移行である。本クレートの保守担当者を主な利用者とし、型安全性の向上とエラーハンドリングの慣用化により将来の保守性を高めることを目的とする。移行後も `cargo build` / `cargo test` / `cargo doc` がすべて通り、メッセージループ・wake・モーダル対応・パニック持ち回りといった観測可能な振る舞いが従来と同一であることを成功条件とする。生 Win32 API を直接呼ぶ既存スタイルは維持しつつ、文字列を扱う API は A 系（ANSI）から W 系（Unicode）へ切り替えることを優先する。高水準ラッパへの全面書き換えは行わない。

## Boundary Context
- **In scope**:
  - `Cargo.toml` の依存を `windows-sys` から `windows = "0.62"` へ差し替え、使用モジュール単位で features を再定義する（`WNDCLASSA`/`RegisterClassA` が要求する `Win32_Graphics_Gdi` を含めて維持）。
  - `src/lib.rs` コア（executor ウィンドウ、`run_loop`、`spawn_unchecked_lifetime`、`MessageLoop`）の型・API 呼び出しの移行。
  - `src/util/window.rs`（`Window<S>` RAII、`wndproc_setup`/`wndproc_typed`、`WindowMessage`）の移行。
  - `src/util/msg_filter_hook.rs`（`MsgFilterHook` RAII、`hook_proc`）の移行。
  - 上記 3 ファイルのテストモジュール、および `FindWindowA`/`MessageBoxA` 等を使う統合テストの移行。
  - 可能な箇所での `Result`/`?` 演算子・RAII 化による unsafe 境界の縮小。
  - Win32 API を A 系（ANSI）から W 系（Unicode）へ切り替える（`*A`→`*W` 関数・構造体、`s!`→`w!` 文字列マクロ）。A 系を残す必然性のある箇所を除き全面適用する。
  - 公開型シグネチャが `windows` 型へ変わることによる SemVer 上の影響の文書化。
- **Out of scope**:
  - 機能追加・挙動変更・パフォーマンス最適化。
  - `windows` クレートの安全（高水準）API への全面的な書き換え。
  - 公開 API の意図的な再設計（構造・契約は維持。型は `windows` 型へ変わる）。
  - ランタイムの非同期挙動・wake 機構・モーダル対応ロジックそのものの変更、`async-task` 利用方法の変更。
  - README / steering の大幅改訂（移行に伴う最小限の記述更新のみ許容）。
- **Adjacent expectations**:
  - 外部依存 `windows` クレート（Microsoft 公式、MIT/Apache-2.0、active）が利用可能であることを前提とする。
  - `examples/basic.rs`・`examples/threads.rs` は高水準公開 API のみを利用しており、ビルドが通り続けることを前提とする。
  - 本クレートは基盤層であり、依存する内部スペックは無い。

## Requirements

### Requirement 1: 依存クレートと feature の差し替え
**Objective:** 保守担当者として、Win32 バインディングを `windows-sys` から `windows`（0.62 系）へ置き換えたい。それにより型安全な newtype と慣用的なエラーハンドリングを利用できるようにするためである。

#### Acceptance Criteria
1. The 移行後のクレート shall `Cargo.toml` の依存として `windows-sys` を含まず、代わりに `windows = "0.62"` 系を含む。
2. The 移行後のクレート shall `windows` 依存の features を、ソースコードで実際に使用される Win32 モジュール単位で宣言する。
3. The 移行後のクレート shall `WNDCLASSA`/`RegisterClassA` が `Win32_Graphics_Gdi` feature でゲートされるため、当該 feature を依存宣言に維持する。
4. The 移行後のソースコード shall `windows_sys::` への参照を一切含まない。

### Requirement 2: 公開 API 挙動の非変更
**Objective:** 本クレートの利用者として、移行の前後で観測可能な振る舞いが同一であってほしい。既存の利用コードが従来通り動作することを保証するためである。

#### Acceptance Criteria
1. The 移行後のクレート shall メッセージループ・wake・モーダルダイアログ対応・ウィンドウプロシージャ/フック内パニックの持ち回りについて、移行前と同一の観測可能な振る舞いを示す。
2. When 移行前に存在した全テスト（メッセージループ・モーダルダイアログ・パニック持ち回り・フィルタ closure 再入検出を含む）を Windows 環境で実行したとき, the テストスイート shall すべて成功する。
3. The 移行作業 shall 既存の公開 API の構造・契約（`spawn_local` / `block_on` / `MessageLoop` / `JoinHandle` / `FilterResult` / `util` の責務と呼び出し契約）を変更しない。
4. The 移行作業 shall executor ウィンドウ宛ての wake メッセージ（`MSG_ID_WAKE`）がユーザーのフィルタ closure で drop されないというガード挙動を維持する。

### Requirement 3: ビルド・テスト・ドキュメントの健全性
**Objective:** 保守担当者として、移行後にクレートのビルド・テスト・ドキュメント生成がすべて成功してほしい。移行が完了した状態を客観的に確認できるようにするためである。

#### Acceptance Criteria
1. When `cargo build` を Windows 環境で実行したとき, the ビルド shall エラーなく完了する。
2. When `cargo test` を Windows 環境で実行したとき, the テスト実行 shall すべてのテストが成功した状態で完了する。
3. When `cargo doc` を実行したとき, the ドキュメント生成 shall エラーなく完了する。
4. When `examples/basic.rs` および `examples/threads.rs` をビルドしたとき, the ビルド shall エラーなく完了する。

### Requirement 4: 慣用的移行の規約適用
**Objective:** 保守担当者として、newtype・`Result`・RAII・文字列マクロを一貫した規約で適用したい。コード全体で型安全と可読性を統一的に高めるためである。

#### Acceptance Criteria
1. The 移行後のソースコード shall 生 Win32 API を直接呼び出す基本スタイルを維持し、文字列を扱う API は W 系（Unicode、`*W`）を優先的に使用する。
2. Where ある Win32 API が失敗を NULL・atom・エラーコード等で返し `windows` バインディングが `Result` を提供する場合, the 移行後のソースコード shall その `Result` を用いてエラーを判定・伝播する。
3. Where ウィンドウハンドル等の引数が NULL を取り得る場合, the 移行後のソースコード shall `windows` の対応するハンドル型（NULL 表現を含む）を用いて当該引数を表現する。
4. The 移行後のソースコード shall 固定の文字列リテラルを `windows` の W 系文字列マクロ（`w!`、PCWSTR）を用いて表現する。
5. While `GetMessageA` のように `windows` バインディングが `Result` ではなく `BOOL` 等の戻り値を返す API を扱う間, the 移行後のソースコード shall 従来同等の戻り値判定ロジックを維持する。
6. The 移行後のソースコード shall 残存する各 `unsafe` ブロックに対し SAFETY コメントで不変条件を明記する既存の慣習を維持する。
7. The 移行後の公開 API（`WindowMessage` のフィールド、`Window::hwnd()` の戻り値等）shall `windows` の newtype（`HWND`/`WPARAM`/`LPARAM`/`LRESULT`）を生整数へアンラップせず、そのまま公開する。

### Requirement 5: スレッド境界（Send/Sync）の検証
**Objective:** 保守担当者として、`windows` の `!Send`/`!Sync` なハンドル型に移行しても thread-local 設計がコンパイルを通ることを確認したい。移行初期に境界エラーの有無を確定させ、後続作業の前提を固めるためである。

#### Acceptance Criteria
1. The 移行後のクレート shall `spawn_unchecked` の schedule クロージャがウィンドウハンドルをキャプチャする箇所（`src/lib.rs`）を含め、`Send`/`Sync` 境界エラーなくコンパイルされる。
2. If `windows` 型のハンドルをキャプチャすることで `Send`/`Sync` 境界エラーが発生する場合, then the 移行作業 shall 当該ハンドルを `Send` 可能な表現へ退避するラッパー方針を後続フェーズの検討対象として記録する。

### Requirement 6: 下流影響と SemVer の文書化
**Objective:** 本クレートの利用者として、公開型の変更が後方互換性に与える影響を把握したい。アップグレード時の破壊的変更を予見できるようにするためである。

#### Acceptance Criteria
1. Where 公開型（`WindowMessage` のフィールド型等）が `windows-sys` 型から `windows` 型へ変わる場合, the 移行作業 shall その変更が SemVer 上の破壊的変更となり得ることを文書化する。
2. The 移行作業 shall README / steering への改訂を、移行に伴う最小限の記述更新の範囲にとどめる。
