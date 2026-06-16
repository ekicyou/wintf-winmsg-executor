# Product Overview

`winmsg-executor` は、Windows 専用のスレッドごとの非同期 Rust executor を提供するライブラリ crate です。各タスクは [message-only window][1] に紐付けられ、executor スレッドはネイティブの [Windows メッセージループ][2] を回します。wake メッセージがタスクのウィンドウプロシージャにディスパッチされ、そこで future がポーリングされます。

GUI アプリケーションや、Windows メッセージループと共存させながら非同期処理を行いたい開発者を主な対象とします。

## Core Capabilities

- **スレッドローカルな task spawn**: `spawn_local()` で future を起動し、`block_on()` / `MessageLoop::run()` で実行する。
- **`Send` / `Sync` 不要**: task future が同一スレッド内で完結するため、スレッド内のデータ共有が容易。
- **マルチタスク同時実行**: 同一スレッド上で複数タスクを動かし、タスクから新たなタスクを spawn して結果を `await` できる。
- **モーダルウィンドウ非ブロッキング**: メニューなどのモーダルウィンドウが開いても、同一スレッド上の他タスクが止まらない（`WH_MSGFILTER` フックを利用）。
- **ウィンドウプロシージャ補助**: state を持てる closure でウィンドウプロシージャを実装するヘルパーを提供。

## Target Use Cases

- Windows ネイティブアプリ内で、メッセージループと統合された軽量な非同期ランタイムが必要な場面。
- `Send` 制約を避けたい、スレッド固有のデータを多用する非同期コード。
- モーダルダイアログ/メニュー表示中もバックグラウンドの非同期処理を継続させたい場面。

## Value Proposition

類似 crate（`windows-executor`、`windows-async-rs`）は 1 スレッド 1 future で `block_on()` のみを公開するのに対し、本 crate は **真の executor** として複数タスクの並行実行・spawn・join をサポートします。さらにモーダルループ中でもメッセージフィルタフックで executor を生かし続ける点が独自の強みです。

## Project Origin & Goal

- **フォーク元 (upstream)**: [timokroeger/winmsg-executor](https://github.com/timokroeger/winmsg-executor)
- **基本記録 (base reference)**: 上流の公開 API・設計知見は [docs.rs v0.3.2](https://docs.rs/winmsg-executor/0.3.2/winmsg_executor/) を正典とする。現行コード（`Cargo.toml` version `0.3.2`）はこの公開 API（`spawn_local` / `block_on` / `MessageLoop` / `JoinHandle` / `FilterResult` / `util`）と一致している。
- **本リポジトリの目的**: 上流に対し **小規模で実用的な改修** を加え、公開 crate（crates.io）として出すこと。

### 改修方針への含意

- 改修は **小さく・実用的に** 保つ。大規模な再設計は本リポジトリのスコープ外。
- 公開 crate を前提に、**公開 API の安定性**と後方互換性に配慮する（破壊的変更は version 方針と整合させる）。
- dual-license（MIT / Apache-2.0）と上流のコーディング慣習（`tech.md` / `structure.md` の SAFETY コメント等）を踏襲する。

---
_Focus on patterns and purpose, not exhaustive feature lists_
_updated_at: 2026-06-16 — フォーク由来・参照ドキュメント・公開 crate 化の目的を追記_

[1]: https://learn.microsoft.com/en-us/windows/win32/winmsg/window-features#message-only-windows
[2]: https://learn.microsoft.com/en-us/windows/win32/winmsg/messages-and-message-queues
