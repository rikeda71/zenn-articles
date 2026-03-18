---
title: "Ghostty のペイン分割を CLI で自動化するツール ghostty-pane-splitter を作った"
emoji: "👻"
type: "tech"
topics: ["ghostty", "rust", "cli", "terminal"]
published: false
---

[Ghostty](https://ghostty.org/) はモダンなターミナルエミュレータで、ペイン分割機能を備えています。しかし、複数のペインを毎回手動でキーバインドを連打して分割するのは面倒です。特に、AI Coding Agent やエディタ、開発サーバーなど複数のペインを日常的にセットアップする場合、その煩わしさは顕著ではないでしょうか。

この課題に対して、すでに tackeyy さんの [ghostty-layout](https://github.com/tackeyy/ghostty-layout) や meijin さんの [AppleScript を使ったアプローチ](https://zenn.dev/meijin/articles/ghostty-pane-split-script)など、先行する取り組みがあります。n番煎じではありますが、macOS / Linux の両方で動作する CLI ツール「ghostty-pane-splitter」を Rust で作ったので紹介します。

https://github.com/rikeda71/ghostty-pane-splitter

![demo](/images/ghostty_pane_splitter/demo-number.gif)

## インストール

### Homebrew (macOS)

```bash
brew install rikeda71/tap/ghostty-pane-splitter
```

### Cargo

```bash
cargo install ghostty-pane-splitter
```

### curl (GitHub Releases)

```bash
# macOS (Apple Silicon)
curl -fsSL https://github.com/rikeda71/ghostty-pane-splitter/releases/latest/download/ghostty-pane-splitter-aarch64-apple-darwin.tar.gz | tar xz
sudo mv ghostty-pane-splitter /usr/local/bin/

# macOS (Intel)
curl -fsSL https://github.com/rikeda71/ghostty-pane-splitter/releases/latest/download/ghostty-pane-splitter-x86_64-apple-darwin.tar.gz | tar xz
sudo mv ghostty-pane-splitter /usr/local/bin/

# Linux (x86_64)
curl -fsSL https://github.com/rikeda71/ghostty-pane-splitter/releases/latest/download/ghostty-pane-splitter-x86_64-unknown-linux-gnu.tar.gz | tar xz
sudo mv ghostty-pane-splitter /usr/local/bin/
```

### ソースからビルド

```bash
git clone https://github.com/rikeda71/ghostty-pane-splitter.git
cd ghostty-pane-splitter
cargo install --path .
```

## 使い方

```
ghostty-pane-splitter <LAYOUT>
```

`<LAYOUT>` にはペイン数、グリッド指定（`列x行`）、またはカスタム列レイアウト（カンマ区切りで各列の行数を指定）を渡します。

### 数値指定

ペイン数を数値で指定すると、自動的にグリッドレイアウトを計算して分割します。

```bash
# 4ペインに分割 (2x2 グリッド)
ghostty-pane-splitter 4

# 9ペインに分割 (3x3 グリッド)
ghostty-pane-splitter 9
```

![number](/images/ghostty_pane_splitter/demo-number.gif)

### グリッド指定（CxR 形式）

列数と行数を `列x行` の形式で明示的に指定できます。

```bash
# 2列 x 3行 で分割
ghostty-pane-splitter 2x3
```

![grid](/images/ghostty_pane_splitter/demo-grid.gif)

### カスタムレイアウト指定

カンマ区切りで各列の行数を指定することで、不均一なレイアウトも作成できます。

```bash
# 左 1 ペイン、右 3 ペイン
ghostty-pane-splitter 1,3

# 3列で各 2, 1, 3 行
ghostty-pane-splitter 2,1,3
```

![custom](/images/ghostty_pane_splitter/demo-custom.gif)

## Ghostty の設定

このツールは Ghostty の設定ファイルからキーバインドを読み取って動作します。以下の5つのキーバインドを Ghostty の設定ファイルに追加してください。

```
keybind = super+d=new_split:right
keybind = super+shift+d=new_split:down
keybind = super+ctrl+right_bracket=goto_split:next
keybind = super+ctrl+left_bracket=goto_split:previous
keybind = super+ctrl+shift+equal=equalize_splits
```

Ghostty の設定ファイルは以下のパスにあります。

- **macOS**: `~/Library/Application Support/com.mitchellh.ghostty/config`
- **Linux**: `~/.config/ghostty/config`

キーバインドを追加・変更した後は、Ghostty を再起動してください。

:::message
設定ファイルが見つからない場合や必要なキーバインドが不足している場合はエラーが表示されます。上記の5つのキーバインドがすべて設定されていることを確認してください。
:::

## AI Coding Agent との連携

Ghostty は [Claude Code](https://code.claude.com/) や [Codex CLI](https://github.com/openai/codex) などの CLI ベースの AI Coding Agent に最適なターミナルとして注目されています。`ghostty-pane-splitter` を使えば、AI エージェント・エディタ・開発サーバーなどのマルチペインレイアウトをコマンド一発で構築できます。

```bash
# 3 ペインレイアウト: 左: AI エージェント、右上: エディタ、右下: ターミナル
ghostty-pane-splitter 1,2
```

また、[git worktree](https://git-scm.com/docs/git-worktree) と組み合わせて複数の AI エージェントを並列実行するワークフローが注目されています。`ghostty-pane-splitter` なら tmux 不要でマルチエージェントレイアウトを瞬時にセットアップできます。

```bash
# 4 ペインレイアウトで複数の AI エージェントを git worktree と並列実行
ghostty-pane-splitter 4
```

## 実装について

ghostty-pane-splitter は [enigo](https://github.com/enigo-rs/enigo) というクレートを使ってキーボード入力のシミュレーションを行い、Ghostty のペイン分割を自動化しています。

処理の流れは以下の通りです。

1. Ghostty の設定ファイルを読み取り、ペイン分割に必要なキーバインドを取得する
2. 指定されたレイアウトに応じて必要な分割操作の順序を計算する
3. enigo を通じてキーボード入力をシミュレートし、Ghostty のキーバインドを順番に発火させる

enigo は macOS では Core Graphics Event API、Linux では X11 (libxdo) を利用してキーストロークを送信します。このクレートが OS ごとの差異を吸収してくれるため、同一のコードベースで macOS / Linux の両方に対応できています。

:::message
Windows は Ghostty 自体が未サポートのため、ghostty-pane-splitter も Windows には対応していません。
:::

## 終わりに

Ghostty のペイン分割を CLI で自動化するツール ghostty-pane-splitter を紹介しました。

AI Coding Agent を複数ペインで並列実行するワークフローなどで活用いただけるかと思います。よければご利用ください！

star や PR、Issue などもお待ちしています。

https://github.com/rikeda71/ghostty-pane-splitter
