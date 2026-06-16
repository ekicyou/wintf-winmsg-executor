# Project Structure

## Organization Philosophy

小規模な単一ライブラリ crate。**公開 API（`src/lib.rs`）と Win32 ヘルパー（`src/util/`）を明確に分離**するレイヤー構成。`lib.rs` は executor のセマンティクスに集中し、生の Win32 ラップは `util` に隔離します。

## Directory Patterns

### Public API / Executor Core
**Location**: `src/lib.rs`
**Purpose**: crate の公開インターフェイスと executor 本体。`spawn_local()`、`block_on()`、`MessageLoop`、`JoinHandle`、`FilterResult` を定義。thread-local な `EXECUTOR_WINDOW` と `PANIC_PAYLOAD` もここに置く。
**Note**: `#![doc = include_str!("../README.md")]` で README を crate ドキュメントとして取り込む。

### Win32 Helper Layer
**Location**: `src/util/`（`mod.rs` で再公開）
**Purpose**: 再利用可能な Win32 ラッパ。`window.rs` = `Window<S>` / `WindowType` / `WindowMessage` などウィンドウ生成・ウィンドウプロシージャ補助、`msg_filter_hook.rs` = `MsgFilterHook`（モーダル対応フック）。
**可視性の慣習**: 外部にも有用なものは `pub use`（`window::*`）、内部専用は `pub(crate) use`（`msg_filter_hook::*`）。

### Examples
**Location**: `examples/`
**Purpose**: 使用例かつ動作確認。`basic.rs`（単一スレッドでの spawn/await）、`threads.rs`（複数スレッドでの `block_on`）。

## Naming Conventions

- **Modules / files**: snake_case（`msg_filter_hook.rs`）。
- **Types**: PascalCase（`MessageLoop`、`JoinHandle`、`WindowType`）。
- **Functions / methods**: snake_case（`spawn_local`、`run_loop`）。
- **Constants**: SCREAMING_SNAKE_CASE（`MSG_ID_WAKE`）。
- **内部の生 wndproc**: `wndproc_setup` / `wndproc_typed` のように役割を表す接尾辞。

## Import Organization

```rust
// std を先頭にグループ化
use std::{cell::Cell, future::Future, pin::pin};

// 外部 crate
use async_task::Runnable;
use windows_sys::Win32::UI::WindowsAndMessaging::*;

// crate 内（util など）
use crate::util::MsgFilterHook;
```

- Win32 シンボルはモジュール単位のグロブ（`...WindowsAndMessaging::*`）で取り込むのが慣習。

## Code Organization Principles

- **安全性の境界を関数契約で表現**: `unsafe fn spawn_unchecked_lifetime` のような内部 unsafe を、`spawn_local`（`'static`）/ `block_on`（return まで生存保証）が安全にラップする。
- **状態は thread-local に集約**: executor ウィンドウ・パニックペイロード・フックポインタは `thread_local!` で保持。
- **ウィンドウ寿命は Rust 側で管理**: `Window<S>` の `Drop` で `DestroyWindow`、`WM_NCDESTROY` で user data を解放。`WM_CLOSE` 時はデフォルト破棄を抑止。
- **再入対策**: `Window::new_checked` は `RefCell` で wndproc クロージャの再入を検出し、再入時はデフォルトプロシージャへフォールバック。
- **テストは対象モジュールに同居**: `lib.rs` / `window.rs` それぞれ末尾に `#[cfg(test)] mod test`。

---
_Document patterns, not file trees. New files following patterns shouldn't require updates_
