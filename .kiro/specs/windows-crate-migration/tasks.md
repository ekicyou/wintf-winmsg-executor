# Implementation Plan

> **全タスク共通方針**: `windows-sys` 0.61 → `windows` 0.62 への挙動非変更の in-place 移行。全タスクで karpathy-guidelines（7.1）を遵守し、型・Win32 API の置換に限定する。隣接コードのリファクタ・整形やコメントの増設を行わず（7.2, 7.3）、`?`/`.into()`/RAII により意味論を変えずに行数を抑える（7.4）。新規ファイル・新規コンポーネントは作成しない。A 系（ANSI）API は W 系（Unicode）へ切り替え、固定文字列は `w!` マクロで表す。
>
> **Deferred**: 6.2 — `README`/steering の改訂は移行に伴う最小限にとどめる制約。本 spec では steering/README 本体を改訂せず、移行完了後の最小限の記述更新を `kiro-complete` フェーズに委譲する。

- [x] 1. 依存クレートと feature の差し替え
  - `windows-sys` 依存を除去し、`windows` 0.62 系を追加する
  - 使用する Win32 モジュール単位で features を宣言する（Foundation / UI WindowsAndMessaging / System Threading / Graphics Gdi）
  - Graphics Gdi はウィンドウクラス登録（`WNDCLASSW`/`RegisterClassW`）がゲートするため維持する
  - 完了状態: `Cargo.toml` に `windows-sys` が存在せず `windows` 0.62 が宣言され、依存解決が成功する（`cargo metadata` でバージョン確認可）
  - _Requirements: 1.1, 1.2, 1.3_
  - _Boundary: Cargo.toml_

- [ ] 2. Win32 ヘルパー層の移行
- [x] 2.1 (P) ウィンドウ生成補助と公開型の移行
  - ウィンドウ生成・破棄の RAII、ウィンドウプロシージャ補助、および公開型を `windows` newtype・W 系 API・`Result` へ移行する
  - NULL を取り得るハンドル引数を `windows` のハンドル型表現（`Option`/`Default`）で表し、固定文字列を W 系文字列マクロで表す
  - 公開型が `windows` newtype（生整数へアンラップしない）へ変わった旨を、該当する公開要素の doc comment に記載する（SemVer 影響の文書化）
  - 完了状態: 当該モジュールに `windows_sys` 参照が無く、ウィンドウ生成・破棄のメッセージ順序テストと state 再入検出テストが移行後に成功する
  - _Requirements: 2.1, 2.3, 4.1, 4.2, 4.3, 4.4, 4.6, 4.7, 6.1, 7.1, 7.2, 7.3, 7.4_
  - _Boundary: util/window.rs_
  - _Depends: 1_

- [x] 2.2 (P) モーダルフックの移行
  - メッセージフィルタフックの登録・解除 RAII と フックプロシージャを `windows` newtype・W 系 API・`Result` へ移行する
  - フック解除は、thread-local ポインタの解放と確保メモリの回収を兼ねる現行の手動 Drop を維持する（ハンドル型の自動解放機構は採用しない）
  - 完了状態: 当該モジュールに `windows_sys` 参照が無く、フックの登録・解除が従来同等に動作する（モーダル系テストの前提を満たす）
  - _Requirements: 2.1, 4.1, 4.2, 4.3, 4.4, 4.6, 7.1, 7.2, 7.3, 7.4_
  - _Boundary: util/msg_filter_hook.rs_
  - _Depends: 1_

- [ ] 3. executor コアの移行
- [x] 3.1 メッセージループと spawn・wake の移行
  - executor ウィンドウ・メッセージループ・タスク spawn・wake 投函を `windows` newtype・W 系 API へ移行する
  - schedule クロージャがウィンドウハンドルをキャプチャする箇所から着手し、`Send`/`Sync` 境界エラーが出ないことを `cargo check` で確認する（万一出た場合はハンドルを `Send` 可能な表現へ退避するラッパーを検討する）
  - メッセージ取得の三値判定（終了メッセージ判定）と、executor 宛て wake メッセージをユーザーのフィルタで drop させないガードを維持する
  - 完了状態: lib.rs 本体に `windows_sys` 参照が無くコンパイルが通り、`Send` 境界エラーが発生しない
  - _Requirements: 2.1, 2.4, 4.1, 4.2, 4.4, 4.5, 4.6, 5.1, 5.2, 7.1, 7.2, 7.3, 7.4_
  - _Boundary: lib.rs_
  - _Depends: 2.1, 2.2_

- [x] 3.2 executor コアのテスト移行
  - メッセージループ・モーダルダイアログ・パニック持ち回り・フィルタ closure 再入・wake 非フィルタの各テストを `windows` newtype・W 系 API へ移行する
  - ウィンドウ名照合に W 系のウィンドウ検索を用い、`w!` による文字列が正しく構築・伝達されることをテスト成功で担保する
  - 完了状態: executor コアの全テストが移行後に成功する
  - _Requirements: 2.2, 4.1, 4.4, 7.1, 7.2, 7.3_
  - _Boundary: lib.rs_
  - _Depends: 3.1_

- [ ] 4. 統合検証
- [ ] 4.1 ビルド・テスト・ドキュメント・examples の全通過確認
  - クレート全体に `windows_sys` 参照が一切残っていないことを確認する
  - ビルド・全テスト実行・ドキュメント生成、および両 example のビルドがすべてエラーなく完了する
  - 完了状態: `cargo build` / `cargo test`（全パス）/ `cargo doc` / 両 example のビルドが成功し、`windows_sys` 参照ゼロが確認される
  - _Requirements: 1.4, 2.2, 3.1, 3.2, 3.3, 3.4, 7.5_
  - _Boundary: crate全体 / 統合検証_
  - _Depends: 2.1, 2.2, 3.2_

## Implementation Notes
- 3.1: `cargo build` がクリーン通過し、`spawn_unchecked` の `HWND`(!Send) キャプチャは Send 境界エラーなくコンパイルされた（5.1 確定、Send ラッパー不要）。lib.rs 本体移行後に残る unused import 警告（`ptr::self` / `HWND`）は test mod でのみ使われるため、3.2 で消費される。
