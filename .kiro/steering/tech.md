# Technology Stack

## Architecture

スレッドごとの非同期 executor。中核は thread-local な executor 用 message-only window と、その上で回るネイティブ Windows メッセージループです。`async-task` がランタイムの足回り（`Runnable` / `Task`）を担い、future が wake されると `PostMessageA` で `MSG_ID_WAKE`（`WM_USER`）が executor ウィンドウに投函され、ウィンドウプロシージャが `Runnable::run()` を呼んで future をポーリングします。Waker は HWND をキャプチャしたクロージャとして実装されます。

## Core Technologies

- **Language**: Rust (edition 2021)
- **Platform**: Windows 専用（Win32 API への FFI）
- **Async runtime primitive**: `async-task`（`default-features = false`）
- **Win32 bindings**: `windows-sys`（`Win32_Foundation` / `Win32_Graphics_Gdi` / `Win32_System_Threading` / `Win32_UI_WindowsAndMessaging`）

## Key Libraries

- `async-task`: `spawn_unchecked` による `Runnable` / `Task` の生成。スケジューラはメッセージ投函クロージャ。
- `windows-sys`: 生の Win32 API（`CreateWindowExA`、`GetMessageA`、`SetWindowsHookExA` など）を直接呼ぶ。高水準ラッパは使わない。

## Development Standards

### Unsafe / FFI
- Win32 FFI を多用するため `unsafe` は不可避。ただし **`unsafe` ブロックには必ず SAFETY コメントで不変条件を明記する**（既存コードの慣習）。
- ライフタイムを跨ぐ `spawn_unchecked` 系の安全性は、呼び出し側の関数契約（`'static` 境界や「task 完了まで return しない」）で保証する。

### Type Safety
- 公開 API は型で安全性を担保（例: `spawn_local` は `T: 'static`、`block_on` はライフタイム付き `'a` を許容）。
- 型消去ポインタ（`*const ()`）はジェネリックを介して再具体化し、`wndproc_setup` をジェネリックフリーに保つパターンを踏襲。

### Testing
- 標準の `#[cfg(test)] mod test` を使用。外部テストフレームワークは導入しない。
- Windows メッセージ/モーダルダイアログを伴う統合的挙動を実際の `MessageBoxA` などで検証する。
- **cargo はテストを並列実行する**ため、ウィンドウ名は各テストで一意にする（既存テストの注意書きに従う）。
- 期待されるパニックは `#[should_panic]` で表現（フィルタ closure の再入検出など）。

## Development Environment

### Required Tools
- Rust toolchain（cargo）、Windows 環境（テストは実 Win32 API を叩く）。

### Common Commands
```bash
# Build:   cargo build
# Test:    cargo test
# Example: cargo run --example basic   /   cargo run --example threads
# Docs:    cargo doc --open
```

## Key Technical Decisions

- **message-only window をタスクの土台に**: 不可視で broadcast メッセージを受けない軽量ウィンドウを wake の受け皿にする。
- **wake メッセージはフィルタ不可**: executor ウィンドウ宛ての `MSG_ID_WAKE` はユーザーのフィルタ closure で drop させない（`run_loop` 内でガード）。
- **モーダル対応に `WH_MSGFILTER` フック**: モーダルループ中もメッセージを観測し executor を動かし続ける。フックは `Drop` で確実に解除。
- **パニックの持ち回り**: ウィンドウプロシージャ/フック内パニックは thread-local の `PANIC_PAYLOAD` に退避し、メッセージループ側で `resume_unwind` する。

## Upstream & Reference

- **フォーク元**: [timokroeger/winmsg-executor](https://github.com/timokroeger/winmsg-executor)（`Cargo.toml` の `repository` も上流を指す）。
- **正典ドキュメント**: [docs.rs v0.3.2](https://docs.rs/winmsg-executor/0.3.2/winmsg_executor/)。公開 API（`spawn_local` / `block_on` / `MessageLoop` / `JoinHandle` / `FilterResult` / `util`）の挙動・契約はこれを基本記録とする。
- **公開 crate 化の留意点**: 小規模改修を前提に、公開 API の安定性・後方互換性を重視する。crates.io 公開時は version 採番（SemVer）と `cargo publish` 前の `cargo test` / `cargo doc` 通過を確認する。
- **Win32 バインディング本家**: [microsoft/windows-rs](https://github.com/microsoft/windows-rs)。`windows`（"Safer bindings"）と `windows-sys`（"Raw bindings"）を提供。リリースは通し番号（最新 **release 73**、2026-02-17）で、各クレートは独立 SemVer（`windows` 本体の crates.io 最新は **0.62.2**、release 73 時点で据え置き）。ウィンドウ生成・ウィンドウプロシージャ・メッセージループの実装例は [create_window サンプル](https://github.com/microsoft/windows-rs/tree/master/crates/samples/windows/create_window)。

---
_Document standards and patterns, not every dependency_
_updated_at: 2026-06-16 — windows-rs 公式リポジトリ（release 73 / create_window サンプル）への参照を追記_
