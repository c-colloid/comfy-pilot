# Comfy Pilot — Claude Desktop / アプリ版 Claude 互換性計画

## 背景

Comfy Pilot は現在 **Claude Code (CLI)** 専用で設計されている。
Claude Desktop（アプリ版）でも使えるようにするための変更計画。

---

## 現状のアーキテクチャ

```
Claude Code CLI ←stdio→ mcp_server.py ←HTTP→ __init__.py (ComfyUI Plugin)
                                                    ↕ REST/WebSocket
                                              js/claude-code.js (ブラウザ)
                                                ├─ xterm.js ターミナル
                                                ├─ ワークフロー同期 (2秒間隔)
                                                └─ グラフコマンド実行 (200ms polling)
```

## CLI 依存箇所の分析

| ファイル | 箇所 (行) | 内容 | Desktop での扱い |
|---|---|---|---|
| `__init__.py` | 749-798 | `setup_mcp_config()` — `claude mcp add` で CLI に登録 | **要変更**: Desktop 用設定生成を追加 |
| `__init__.py` | 137-184 | `install_claude_code()` — CLI 自動インストール | **条件分岐**: Desktop 検出時はスキップ |
| `__init__.py` | 207-326 | `WebSocketTerminal` — PTY で CLI 起動 | **条件分岐**: Desktop モードでは不使用 |
| `__init__.py` | 42-65 | `has_claude_conversation()` — `~/.claude/projects/` 参照 | **条件分岐**: Desktop モードでは不使用 |
| `__init__.py` | 187-204 | `get_claude_command()` — CLI パス解決 + `-c` フラグ | **条件分岐**: Desktop モードでは不使用 |
| `__init__.py` | 813 | `IS_WINDOWS` で `setup_mcp_config()` スキップ | **要変更**: Desktop なら Windows でも設定生成 |
| `js/claude-code.js` | 42-645 | xterm.js ターミナル UI 全体 | **要変更**: Desktop モード UI を追加 |

## 変更が不要な箇所（両モード共通）

以下は CLI/Desktop で完全共通であり、**修正不要**:

- `mcp_server.py` — 全 15 ツールの実装（ComfyUI REST API 経由、トランスポート非依存）
- `js/claude-code.js` — ワークフロー同期 (`syncWorkflow`, L1000-1041)
- `js/claude-code.js` — グラフコマンド実行 (`pollGraphCommands` + `executeGraphCommand`, L1053-1423)
- `__init__.py` — REST エンドポイント群 (`/claude-code/workflow`, `/claude-code/graph-command` 等)
- `tests/test_edit_graph.py` — バックエンドロジックのテスト

---

## 変更計画

### Phase 1 (P0): 最小限の Desktop 動作

#### 1.1 Claude Desktop 向け MCP 設定の自動生成

**ファイル: `__init__.py`**

`setup_desktop_mcp_config()` 関数を追加。Claude Desktop は `claude_desktop_config.json` で MCP サーバーを管理する。

OS ごとの設定ファイルパス:
- macOS: `~/Library/Application Support/Claude/claude_desktop_config.json`
- Windows: `%APPDATA%\Claude\claude_desktop_config.json`
- Linux: `~/.config/Claude/claude_desktop_config.json`

処理:
1. 上記パスのディレクトリが存在するか検出（Desktop インストール判定）
2. 既存 JSON を読み込み、`mcpServers.comfyui` エントリを追加/更新（他のサーバー設定は保持）
3. Python インタープリタのパスと `mcp_server.py` の絶対パスを設定

生成例:
```json
{
  "mcpServers": {
    "comfyui": {
      "command": "/path/to/python3",
      "args": ["/path/to/comfy-pilot/mcp_server.py"]
    }
  }
}
```

#### 1.2 起動シーケンスの変更

**ファイル: `__init__.py` (L801-823)**

現在:
```python
if not IS_WINDOWS:
    setup_mcp_config()  # CLI 専用
```

変更後:
```python
# CLI 設定（claude CLI が見つかった場合のみ）
if not IS_WINDOWS and find_executable("claude"):
    setup_mcp_config()

# Desktop 設定（全 OS、Desktop ディレクトリ検出時）
setup_desktop_mcp_config()
```

これにより:
- CLI がインストールされていない環境でも Desktop 連携が動作
- Windows ユーザーも Desktop 経由で利用可能に

#### 1.3 MCP プロトコル互換性の確認・修正

**ファイル: `mcp_server.py`**

- `protocolVersion: "2024-11-05"` — Desktop が要求するバージョンとの整合性を確認
- `view_image` のレスポンス形式確認（L2817-2835）
  - 現在: `{"type": "image", "data": "base64...", "mimeType": "image/png"}`
  - Desktop が `"type": "resource"` 形式を要求する場合は調整
- ComfyUI 未起動時のエラーメッセージ改善（`make_request()` 失敗時に明示的なガイダンス）

---

### Phase 2 (P1): UX 改善

#### 2.1 フロントエンド UI のデュアルモード化

**ファイル: `js/claude-code.js`**

##### モード自動検出

```javascript
async function detectMode() {
    const res = await fetch("/claude-code/platform");
    const info = await res.json();
    // ターミナル非対応 or CLI 未インストール → Desktop モード
    if (!info.terminal_supported || !info.claude_cli_installed) {
        return "desktop";
    }
    return "cli";
}
```

##### Desktop モード UI

ターミナルの代わりに軽量ステータスパネルを表示:

```
┌──────────────────────────────────────────┐
│ Comfy Pilot                MCP ● 接続中  │
├──────────────────────────────────────────┤
│                                          │
│  ✓ MCP Server: 動作中                    │
│  ✓ ワークフロー同期: 有効                 │
│  ✓ ComfyUI: http://127.0.0.1:8188       │
│                                          │
│  Claude Desktop でお使いください          │
│                                          │
│  ⚠ 設定更新後はアプリの再起動が必要です   │
│                                          │
│  [Desktop 設定を更新]  [設定パスを開く]   │
│                                          │
└──────────────────────────────────────────┘
```

- ワークフロー同期・グラフコマンドポーリングは **両モードで常時稼働**（MCP ツール動作に必須）
- xterm.js の CDN 読み込みは Desktop モードではスキップ（帯域・メモリ節約）
- ヘッダーに現在のモード表示（CLI / Desktop）

##### CLI モード

既存の xterm.js ターミナル UI をそのまま維持（変更なし）。

#### 2.2 バックエンド API の拡張

**ファイル: `__init__.py`**

新しいエンドポイント:

```python
# Desktop セットアップ状態の確認
GET /claude-code/desktop-status
→ {
    "desktop_detected": true,
    "config_path": "~/Library/Application Support/Claude/claude_desktop_config.json",
    "mcp_configured": true,
    "needs_restart": false
  }

# Desktop 設定の手動セットアップ/更新
POST /claude-code/setup-desktop
→ claude_desktop_config.json を生成/更新して結果を返す
```

`/claude-code/platform` レスポンスの拡張 (L678-686):
```json
{
  "os": "darwin",
  "terminal_supported": true,
  "claude_cli_installed": true,
  "claude_desktop_detected": true,
  "desktop_mcp_configured": true
}
```

---

### Phase 3 (P2): 品質向上

#### 3.1 命名・ブランディング統一

ユーザー向け表示を「Comfy Pilot」に統一:

- ログメッセージ: `[Claude Code]` → `[Comfy Pilot]`
- UI タイトル: `"Claude Code"` → `"Comfy Pilot"`
- メニューボタン: `"Claude Code"` → `"Comfy Pilot"`
- 内部パス (`/claude-code/*`, CSS クラス) は後方互換のため維持

#### 3.2 Windows 対応強化

- Desktop 設定が Windows でも自動生成される（1.2 で対応済み）
- フロントエンドが Windows で自動的に Desktop モードに切り替わる
- ログメッセージ改善:
  - 現在: `"Terminal not supported on Windows. Use Claude Code CLI directly."`
  - 変更: `"Terminal not available on Windows. Using Desktop mode."`

#### 3.3 ドキュメント更新

README.md に追加:
- Claude Desktop セットアップ手順
- サポートクライアント一覧（Claude Code CLI + Claude Desktop）
- 「ComfyUI をブラウザで開いておく必要がある」ことの明記
- トラブルシューティング（Desktop 再起動の必要性等）

---

### Phase 4 (P3): 将来拡張

#### 4.1 SSE/Streamable HTTP トランスポート

リモート ComfyUI（クラウド GPU 等）向けに HTTP ベースの MCP トランスポート:
- `mcp_server.py --transport sse --port 3001`
- ローカル環境のみなら stdio で十分なので優先度低
- Claude Desktop が将来 SSE をネイティブサポートした場合に対応

---

## 重要な制約事項

### ブラウザ必須（両モード共通）

`edit_graph`, `center_on_node`, `run(queue)` 等のツールはフロントエンド JS 経由で実行される。
Desktop モードでも **ComfyUI をブラウザで開いておく必要がある**。これは設計上の制約。

### Desktop 再起動

`claude_desktop_config.json` を変更した後、**Claude Desktop アプリの再起動が必要**。
UI でこれを明確に案内する。

### MCP サーバーのライフサイクル

- CLI: MCP サーバーは Claude Code プロセス内で起動
- Desktop: MCP サーバーは **Desktop アプリから直接起動**される（ComfyUI プラグイン経由ではない）
- ComfyUI が未起動の場合 `make_request()` が失敗するため、エラーメッセージで「ComfyUI をブラウザで開いてください」と案内

---

## 実装順序まとめ

```
Phase 1 (P0) — 最小限の Desktop 動作
  ├─ 1.1 setup_desktop_mcp_config() 追加
  ├─ 1.2 起動シーケンス変更（全 OS 対応）
  └─ 1.3 MCP プロトコル互換性確認

Phase 2 (P1) — UX 改善
  ├─ 2.1 デュアルモード UI（CLI / Desktop）
  └─ 2.2 API 拡張（desktop-status, setup-desktop）

Phase 3 (P2) — 品質向上
  ├─ 3.1 命名統一（Comfy Pilot）
  ├─ 3.2 Windows 強化
  └─ 3.3 ドキュメント更新

Phase 4 (P3) — 将来拡張
  └─ 4.1 SSE/HTTP トランスポート
```

**想定される変更ファイル数**: 主に 3 ファイル (`__init__.py`, `js/claude-code.js`, `mcp_server.py`) + `README.md`
