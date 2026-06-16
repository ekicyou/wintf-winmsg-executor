# Gap Analysis: windows-crate-migration

`windows-sys` 0.61 → `windows` 0.62 移行のための、要件と既存コードベースのギャップ分析。
実コード（`src/lib.rs` / `src/util/window.rs` / `src/util/msg_filter_hook.rs` および各テスト）と
`windows` 0.62.2 実ソース・コンパイル検証に基づく。

## 1. 現状調査サマリ

- **構成**: 小規模単一 crate。公開 API/executor コア（`src/lib.rs`）と Win32 ヘルパー（`src/util/`）の 2 層。`windows_sys::` 直接参照は 3 ファイルのみ（各テストモジュール含む）。
- **スタイル**: 生 Win32 API を A 系（ANSI）で直接呼ぶ。高水準ラッパ不使用。`unsafe` には SAFETY コメントを付す慣習。
- **RAII**: `Window<S>`（Drop で `DestroyWindow`）、`MsgFilterHook`（Drop で `UnhookWindowsHookEx`）は既に手動 RAII 実装済み。
- **examples**: `basic.rs` / `threads.rs` は公開 API（`spawn_local`/`block_on`）のみ利用。`windows`/`windows-sys` 直接依存なし → 影響は「ビルドが通り続けること」（R3.4）のみ。

## 2. Requirement-to-Asset Map

| 要件 | 対象資産 | ギャップ種別 | 内容 |
|---|---|---|---|
| R1 依存差し替え | `Cargo.toml` | **Constraint（仮定修正）** | `windows = "0.62"` へ。features は `Win32_Foundation` / `Win32_UI_WindowsAndMessaging` / `Win32_System_Threading` / **`Win32_Graphics_Gdi`（除去不可・要維持）**。`windows_sys::` 参照ゼロ化。 |
| R2 挙動非変更 | 全 3 ファイル + テスト | Constraint | 既存テスト群が安全網。観測挙動・公開契約・wake ガードを保持。 |
| R3 健全性 | crate 全体 + examples | Low gap | build/test/doc + examples ビルド。examples はソース改変不要。 |
| R4 慣用移行規約 | 全 3 ファイル | **Missing（機械変換）** | newtype `.0`、`Result`+`?`、`Option<HWND>`、`s!` マクロ、`BOOL` 判定温存、SAFETY 維持。 |
| R5 Send/Sync 検証 | `src/lib.rs` | **Unknown→解消見込み** | `spawn_unchecked` 使用ゆえ schedule クロージャに `Send` 不要 → `HWND`(!Send) キャプチャでもコンパイル可の公算大。実ビルドで確定させる。 |
| R6 下流影響/SemVer | `WindowMessage` 等公開型 | Constraint | フィールド型が `windows` 型へ → SemVer 破壊。文書化。 |

## 3. API 移行ファクト表（設計者向け即参照）

> **方針更新（要件ディスカッション）**: A 系（ANSI）→ W 系（Unicode）へ全面切り替え。下表の `*A` 関数は対応する `*W`（`CreateWindowExW`/`RegisterClassW`/`DefWindowProcW`/`SendMessageW`/`PostMessageW`/`GetMessageW`/`DispatchMessageW`/`SetWindowsHookExW`/`FindWindowW`/`MessageBoxW`、構造体 `WNDCLASSW`/`CREATESTRUCTW`）へ読み替え、文字列は `s!`→`w!`（PCWSTR）とする。Result/Option/newtype のシグネチャ形状は A 系と同一。ウィンドウ名は ASCII のみゆえ観測挙動は不変。

### Result 化される（`?` で畳める）
| 関数 | 0.62 戻り値 | NULL→Option 引数 | 現状コード箇所 |
|---|---|---|---|
| `CreateWindowExA` | `Result<HWND>` | parent/hMenu/hInstance/lpParam が `Option` | window.rs:142 `is_null()` 判定 → `?`/`map_err(WindowCreationError)` |
| `PostMessageA` | `Result<()>` | hwnd: `Option<HWND>` | lib.rs:75,236,245,273 / window.rs:347,356 戻り値無視 → `let _ =` |
| `DestroyWindow` | `Result<()>` | hwnd: 生 `HWND` | window.rs:61（Drop 内）→ `let _ =` |
| `SetWindowsHookExA` | `Result<HHOOK>` | hmod: `Option<HINSTANCE>` | msg_filter_hook.rs:28 → `?` か `.unwrap()`（register は unsafe fn）|
| `UnhookWindowsHookEx` | `Result<()>` | hhk: 生 `HHOOK` | msg_filter_hook.rs:44（Drop 内）→ `let _ =` |
| `FindWindowA` | `Result<HWND>` | class/title: `Param<PCSTR>` | lib.rs:384（テスト）→ `.ok()`/`.unwrap_or_default()` で `is_null()` 維持 |

### Result にならない（生戻り・判定温存）
| 関数 | 0.62 戻り値 | 注意 |
|---|---|---|
| `GetMessageA` | `BOOL` | **Result でない**。`== 0`(WM_QUIT) 判定は `.as_bool()`/`.0` で温存。lib.rs:180 |
| `RegisterClassA` | `u16`(ATOM) | 失敗 0。**`Win32_Graphics_Gdi` でゲート**。window.rs:132 |
| `SendMessageA` | `LRESULT` | hwnd 生 `HWND`（非 Option）。window.rs:246 / lib.rs:406,515 |
| `DefWindowProcA` | `LRESULT` | window.rs:251,295 |
| `DispatchMessageA` / `TranslateMessage` | `LRESULT` / `BOOL` | lib.rs:188,189。引数 `*const MSG` |
| `GetWindowLongPtrA`/`SetWindowLongPtrA` | `isize` | index 型 `WINDOW_LONG_PTR_INDEX`(GWLP_* は既にこの型)。window.rs:213,238,243,267 |
| `MessageBoxA` | `MESSAGEBOX_RESULT` | hwnd: `Option<HWND>`、text/caption: `Param<PCSTR>`、utype: `MESSAGEBOX_STYLE`。lib.rs:412,445,496 |
| `PostQuitMessage` / `GetCurrentThreadId` | `()` / `u32` | 変化なし |

### 型・定数
| 項目 | 0.62 定義 | 移行影響 |
|---|---|---|
| `HWND`/`HINSTANCE` | `struct(pub *mut c_void)`、Default=NULL 手動、`is_invalid()`=`is_null()`、`PartialEq/Eq` 有 | `== executor_hwnd` 比較は可（lib.rs:186）。`is_null()` は `.0.is_null()` か `== HWND::default()`。NULL 引数は `HWND::default()` または `Option::None` |
| `WPARAM`/`LPARAM`/`LRESULT` | `struct(pub usize/isize/isize)`、Default derive 有 | `lparam as *mut _` → `lparam.0 as *mut _`（lib.rs:29、msg_filter_hook.rs:57）。戻り `0`/`1` → `LRESULT(0)`/`LRESULT(1)`（window.rs:285,292、msg_filter_hook.rs:59）|
| `HHOOK` | WindowsAndMessaging 内、Default 無、`windows_core::Free` 実装（drop で自動 Unhook）| **設計判断**: 自動 Free を使うか現状の手動 Drop を維持するか（§7 参照）|
| `HWND_MESSAGE` | `HWND`(-3) 型 | window.rs:153 `HWND_MESSAGE` 直渡し可。`ptr::null_mut()`(TopLevel) → `HWND::default()` |
| `WINDOW_EX_STYLE` | `struct(pub u32)` | `ex_style: 0`（new の既定）→ `WINDOW_EX_STYLE(0)`。`new_ex` 引数型は既に `WINDOW_EX_STYLE`（window.rs:115）|
| `CW_USEDEFAULT` | `i32` | 変化なし |
| `s!` マクロ | `windows::core::s!("…")` → `PCSTR` | class 名 `c"…"`(window.rs:122) → `s!("wintf-winmsg-executor")`。テストの `c"…"`/`CStr` ヘルパも整理可 |
| `WNDCLASSA` | `lpfnWndProc: WNDPROC`(=`Option<fn(HWND,u32,WPARAM,LPARAM)->LRESULT>`), `lpszClassName: PCSTR`, GDI ゲート | window.rs:128 `mem::zeroed()` → `WNDCLASSA::default()` も可 |
| `CREATESTRUCTA` | `lpCreateParams: *mut c_void` | window.rs:234 `lparam.0 as *const CREATESTRUCTA` |

### 公式サンプルの実装イディオム（microsoft/windows-rs `create_window`）
A/W 共通で適用可能な実装作法（本クレート移行の手本。公式サンプル自体は A系だが本クレートは W系方針ゆえ `*W`/`w!` に読み替え）:
- `let hwnd = CreateWindowExW(/* … */)?;` — Result を `?` で伝播。
- `while GetMessageW(&mut message, None, 0, 0).into() { DispatchMessageW(&message); }` — `BOOL`→`bool` を `.into()` でループ条件に変換。第2引数は `None`（`Option<HWND>`）。本クレートの三値判定（`== 0` で WM_QUIT）は従来通り温存しつつ `.into()`/`.0` を活用。
- wndproc 戻り値は `LRESULT(0)` で構築。フォールスルーは `DefWindowProcW(window, message, wparam, lparam)` をそのまま返す。
- 文字列リテラルは `w!("…")`（PCWSTR）を直接渡す。
- 出典: https://github.com/microsoft/windows-rs/tree/master/crates/samples/windows/create_window

## 4. 主要な発見・リスク

1. **【最重要・仮定修正】`Win32_Graphics_Gdi` は除去不可**。`WNDCLASSA`/`RegisterClassA` がこの feature でゲートされ、外すとコンパイル不能（実証済み）。→ R1.3 の「未使用なら除外」は **不発火**。features は 4 つ全て維持が正解。設計・タスクで「除外しない」と明記すべき。
2. **wndproc 関数ポインタのキャスト**（window.rs:238 `subclassinfo.wndproc as usize as _`）。`SetWindowLongPtrA(hwnd, GWLP_WNDPROC, …)` は newtype index・isize 引数。`wndproc` の型 `unsafe extern "system" fn(HWND,u32,WPARAM,LPARAM)->LRESULT` は windows 0.62 でも同形（引数が newtype 化されるのみ）。キャスト経路は概ね温存可。
3. **`Send`/`Sync`（R5）**: `async_task::spawn_unchecked`（lib.rs:73）は schedule クロージャに `Send` を要求しない。`HWND`(!Send) キャプチャでも理論上コンパイル可 → 移行初期の実ビルドで即確定（ラッパー不要の公算大）。
4. **公開型 `WindowMessage`**（window.rs:37-42）のフィールド `hwnd/wparam/lparam` が `windows` 型へ。`WindowCreationError` は不変。`Window::hwnd()` 戻り `HWND` も型変更 → SemVer 破壊（R6 文書化対象、0.0.x ゆえ許容）。
5. **テスト移行量**: lib.rs テスト（`FindWindowA`/`MessageBoxA`/`SendMessageA`/`PostMessageA`/`c"…"`/`is_null()` 多数）が実質最大の変更面。挙動非変更ゆえ安全網としても機能。

## 5. 実装アプローチ

### Option A: 既存 3 ファイルを in-place 一括移行（推奨）
- **方法**: `Cargo.toml` 差し替え → 3 ファイル + テストを一斉に `windows` へ書き換え → `cargo build`/`test` で収束。
- ✅ 小規模（3 ファイル）ゆえ最短。新規ファイル不要。パターン統一が容易。
- ✅ 既存テストが挙動非変更の安全網。
- ❌ 依存差し替え直後は全ファイルがコンパイルエラー状態（中間で緑にならない）。
- **適合**: 本 spec に最適。スコープが明確で機械的変換が大半。

### Option B: 新規ヘルパー型の導入
- **方法**: `Send` ハンドルラッパ等を新規追加。
- ❌ R5 の検証結果次第では不要（公算大）。先回り新設は YAGNI。
- **適合**: 低。R5 が `Send` エラーを実際に出した場合のみ着手。

### Option C: `windows-sys`/`windows` 併存の段階移行（hybrid）
- **方法**: 両依存を一時併存させ、`util/window.rs` → `util/msg_filter_hook.rs` → `lib.rs` の順にファイル単位で置換、各段でビルド緑を保つ。
- ✅ 各ステップでコンパイル可能、差分レビューが細かい。`repr(transparent)` で相互運用可。
- ❌ 一時的に依存 2 本・`use` 衝突管理の手間。小規模ゆえ過剰。
- **適合**: 中。慎重を期すなら有効だが、規模的には Option A で十分。

## 6. Effort / Risk

- **Effort: S〜M（2〜4 日）** — 3 ファイル + テスト。変換は機械的だが箇所数が多く（newtype `.0` が全 API 呼び出しに波及）、テスト移行が嵩む。
- **Risk: Low** — 既知技術（Microsoft 公式・active）、明確スコープ、挙動非変更で既存テストが安全網。唯一の不確実性 R5 は移行初期に実ビルドで即解消。GDI feature 仮定は本分析で解消済み。

## 7. 設計フェーズへの推奨

**推奨アプローチ**: Option A（in-place 一括移行）。慎重運用が必要なら Option C を予備に。

**設計で決めるべき key decisions**:
1. **`MsgFilterHook` の Drop 戦略**: `HHOOK` の `windows_core::Free`（自動 Unhook）を活用するか、現状の明示的手動 Drop（`UnhookWindowsHookEx` + Box 解放 + thread-local クリア）を維持するか。手動 Drop は thread-local ポインタ解放も担うため、**手動維持が無難**（自動 Free は thread-local 後始末を行わない）。
2. **NULL 表現の規約統一**: `Option<HWND>`(None) を取る API（`PostMessageA`/`GetMessageA`/`CallNextHookEx`/`MessageBoxA`）と、生 `HWND`(`HWND::default()`) を取る API（`SendMessageA`/`DefWindowProcA`/`DestroyWindow`）の使い分けを規約化。
3. **`Result` の扱い**: Drop 内・投函系（`PostMessageA`/`DestroyWindow`/`UnhookWindowsHookEx`）は `let _ =` で従来同等（戻り値無視）。`CreateWindowExA` のみ `WindowCreationError` へ `map`。
4. **`s!` 経路**: `windows::core::s!` を `use` するか fully-qualified か（import 慣習＝モジュールグロブとの整合）。
5. **features 確定**: `Win32_Foundation` / `Win32_UI_WindowsAndMessaging` / `Win32_System_Threading` / `Win32_Graphics_Gdi` の 4 つを維持（**Gdi 除外しない**）。

**Research Needed（設計で確認）**:
- [R-1] `Window::hwnd()` 戻り `HWND` を `pub` で晒し続けるか（SemVer 上の公開面確定）。
- [R-2] `WindowMessage` の `wparam`/`lparam` を `windows` newtype のまま晒すか、`.0` を剥がして生整数で晒すか（公開 API の使い勝手 vs 一貫性）。
- [R-3] R5 実ビルド結果の確定（`Send` エラーの有無）— 設計の最初のタスクで検証し、Option B 要否を決定。
