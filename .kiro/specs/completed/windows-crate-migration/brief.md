# Brief: windows-crate-migration

## Problem
本クレートは Win32 FFI に `windows-sys`（生バインディング）を採用している。`windows-sys` は型安全性の薄い素の FFI 層であり、`HWND`/`WPARAM`/`LPARAM`/`LRESULT` などが生のポインタ・整数のまま扱われ、API 失敗時のエラーハンドリングも手書きの NULL/0 判定に依存している。Microsoft 公式の高水準寄りバインディングである `windows` クレートへ移行することで、newtype による型安全・`Result`/`?` によるエラー伝播・RAII・`s!`/`w!` マクロによる文字列の簡潔化といった恩恵を受けられる。

## Current State
- 依存: `windows-sys = "0.61.2"`（features: `Win32_Foundation` / `Win32_Graphics_Gdi` / `Win32_System_Threading` / `Win32_UI_WindowsAndMessaging`）。
- `windows_sys::` を直接参照しているのは実質 3 ファイル: `src/lib.rs`、`src/util/window.rs`、`src/util/msg_filter_hook.rs`（各ファイルのテストモジュール含む）。
- 使い方は一貫して「生の Win32 API を直接叩く」スタイル（`CreateWindowExA` / `GetMessageA` / `SetWindowsHookExA` / `PostMessageA` / `SetWindowLongPtrA` 等）。高水準ラッパは使っていない。
- 設計は徹底して thread-local（executor ウィンドウ・フック・タスクはすべて同一スレッド）。`HWND` をスレッド間で持ち回る箇所は確認されていない。

## Desired Outcome
- 依存が `windows-sys` から `windows`（0.62 系）へ置き換わり、`cargo build` / `cargo test` / `cargo doc` がすべて通る。
- 既存の公開 API の**挙動**が変わらない（メッセージループ・wake・モーダル対応・パニック持ち回りの振る舞いが従来通り）。
- 「慣用的移行」方針: 可能な箇所は `Result` + `?` 演算子・RAII を活用し、unsafe 境界を縮小する。ただし `CreateWindowExA` 等の生 Win32 API を直接呼ぶ基本スタイルは維持する（高水準ラッパへの全面書き換えはしない）。

## Approach
ユーザー選択 = **慣用的移行（Result/RAII 活用）**。
- 移行先は `windows = "0.62"`（`windows-sys` の 0.61 と番号を合わせて 0.61 を選ばないこと。0.62 が同一リンク機構世代で正しい）。
- features は使用モジュール単位で再定義（`Win32_Foundation` / `Win32_System_Threading` / `Win32_UI_WindowsAndMessaging` を中心に、`Graphics_Gdi` は未使用の疑いがあるため精査して除外候補）。
- newtype 化に伴う `.0` アンラップ、NULL/atom 返し API の `Result` 畳み込み、`GetMessageA` の `BOOL` 判定（Result にならない）温存、文字列リテラルの `s!` マクロ化を機械的に適用する。
- A 系 API（`CreateWindowExA` 等、ANSI）は現状維持を基本とし、W 系への切り替えはスコープ外（必要なら別途検討）。

## Scope
- **In**:
  - `Cargo.toml` の依存差し替えと features 棚卸し。
  - `src/lib.rs` コア（executor ウィンドウ、`run_loop`、`spawn_unchecked_lifetime`、`MessageLoop`）の型・API 移行。
  - `src/util/window.rs`（`Window<S>` RAII、`wndproc_setup`/`wndproc_typed`、`WindowMessage`）の移行。
  - `src/util/msg_filter_hook.rs`（`MsgFilterHook` RAII、`hook_proc`）の移行。
  - 上記 3 ファイルのテストモジュールの移行。
  - 可能な箇所での `Result`/`?`/RAII 化による unsafe 境界の縮小。
- **Out**:
  - 機能追加・挙動変更・パフォーマンス最適化。
  - 高水準ラッパ（`windows` の安全 API）への全面的な書き換え。
  - A 系 → W 系 API への切り替え。
  - 公開 API の意図的な再設計（型シグネチャは windows 型に変わるが、構造・契約は維持）。

## Boundary Candidates
- **B1 依存・feature 層**: `Cargo.toml` の `windows` 化と使用モジュールの精査（独立して着手可能、他の起点）。
- **B2 横断方針の確立**: newtype `.0` 規約、`Option<HWND>` への NULL 引数移行、`Result` 畳み込み、`s!` マクロ文字列、`BOOL` 判定パターンの統一ルール（他境界の前提となる規約）。
- **B3 `util/window.rs`**: ウィンドウ生成 RAII と wndproc コールバックの移行。
- **B4 `util/msg_filter_hook.rs`**: フック登録 RAII の移行。
- **B5 `lib.rs` コア**: executor ウィンドウとメッセージループの移行（B3/B4 に依存）。
- **B6 テスト**: 各ファイルのテストモジュールと、`FindWindowA`/`MessageBoxA` 等を使う統合テストの移行。

## Out of Boundary
- ランタイムの非同期挙動・wake 機構・モーダル対応ロジックそのものの変更。
- `async-task` の利用方法の変更。
- README / steering の大幅改訂（移行に伴う最小限の記述更新は許容）。

## Upstream / Downstream
- **Upstream**: 基盤層であり依存する内部スペックは無し。外部依存は `windows` クレート（Microsoft 公式、MIT/Apache-2.0、active）。
- **Downstream**: `examples/basic.rs`・`examples/threads.rs`（高水準公開 API のみ利用のため影響は小さい見込み、要確認）。本クレートの利用者全般。公開型（`WindowMessage` のフィールド型等）が `windows` 型に変わるため、SemVer 上は破壊的変更となり得る（現行 0.0.x のため許容範囲だが文書化する）。

## Existing Spec Touchpoints
- **Extends**: なし（`.kiro/specs/` は空。本スペックが第 1 号）。
- **Adjacent**: なし。

## Constraints
- **挙動非変更**: 移行後も `cargo test` の全テスト（メッセージループ・モーダルダイアログ・パニック持ち回り・フィルタ再入検出）がパスすること。
- **型制約**: `windows` の `HWND` 等は `!Send`/`!Sync`。本プロジェクトは thread-local 設計のため問題にならない見込みだが、`spawn_unchecked` の schedule クロージャが `HWND` をキャプチャする箇所（`src/lib.rs`）で `Send` 境界エラーが出ないことを移行初期に検証する（出た場合は `isize`/`*mut c_void` へ落とす Send ラッパーを設計に追加）。
- **API スタイル**: 生 Win32 API 直接呼び出しを維持（unsafe は不可避、SAFETY コメント慣習を踏襲）。
- **バージョン**: `windows = "0.62"` を採用（0.61 を選ばない）。
- **プラットフォーム**: Windows 専用。テストは実 Win32 API を叩くため Windows 環境で検証。
