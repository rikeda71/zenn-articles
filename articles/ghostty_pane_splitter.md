---
title: "Ghostty の pane 分割を CLI で自動化するツール ghostty-pane-splitter を作った"
emoji: "👻"
type: "tech"
topics: ["ghostty", "rust", "cli", "terminal"]
published: false
---

[Ghostty](https://ghostty.org/) は軽量かつ設定が簡単で、最近は [Claude Code](https://code.claude.com/) や [Codex CLI](https://github.com/openai/codex) などの AI Coding Agent を動かすのによく使われています。
Ghostty で pane を分割する場合、決まったフォーマットに分割したい場合でも、毎回手動で分割する必要があります。

この課題に対して、すでに[いくつかのアプローチ](#参考文献)が紹介されています。n番煎じではありますが、macOS / Linux の両方で動作する CLI ツール「ghostty-pane-splitter」を Rust で作ったので紹介します。

https://github.com/rikeda71/ghostty-pane-splitter

## できること

数値、グリッド、カスタムの3種類のレイアウト指定で pane 分割を自動化できます。

| 指定方法 | デモ |
| --- | --- |
| 数値指定 (`ghostty-pane-splitter 4`) | ![number](/images/ghostty_pane_splitter/demo-number.gif) |
| グリッド指定 (`ghostty-pane-splitter 2x3`) | ![grid](/images/ghostty_pane_splitter/demo-grid.gif) |
| カスタム指定 (`ghostty-pane-splitter 1,3`) | ![custom](/images/ghostty_pane_splitter/demo-custom.gif) |

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

## 使い方と設定

### Ghostty の設定

このツールは Ghostty の設定ファイルからキーバインドを読み取って動作します。以下の5つのアクションに対するキーバインドを Ghostty の設定ファイルに追加してください。

- `new_split:right`
- `new_split:down`
- `goto_split:next`
- `goto_split:previous`
- `equalize_splits`

キーの組み合わせは任意で構いません。設定の詳細は [Ghostty のキーバインドリファレンス](https://ghostty.org/docs/config/keybind/reference)を参照してください。以下は設定例です。

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

### レイアウト指定

```
ghostty-pane-splitter <LAYOUT>
```

`<LAYOUT>` にはpane数、グリッド指定（`列x行`）、またはカスタム列レイアウト（カンマ区切りで各列の行数を指定）を渡します。

#### 数値指定

pane数を数値で指定すると、自動的にグリッドレイアウトを計算して分割します。

```bash
# 4paneに分割 (2x2 グリッド)
ghostty-pane-splitter 4

# 9paneに分割 (3x3 グリッド)
ghostty-pane-splitter 9
```

#### グリッド指定（CxR 形式）

列数と行数を `列x行` の形式で明示的に指定できます。

```bash
# 2列 x 3行 で分割
ghostty-pane-splitter 2x3
```

#### カスタムレイアウト指定

カンマ区切りで各列の行数を指定することで、不均一なレイアウトも作成できます。

```bash
# 左 1 pane、右 3 pane
ghostty-pane-splitter 1,3

# 3列で各 2, 1, 3 行
ghostty-pane-splitter 2,1,3
```

## 実装について

ghostty-pane-splitter は [enigo](https://github.com/enigo-rs/enigo) というクレートを使ってキーボード入力をシミュレートし、Ghostty のpane分割を自動化しています。

https://github.com/enigo-rs/enigo

処理の流れは以下の通りです。

1. Ghostty の設定ファイルを読み取り、pane分割に必要なキーバインドを取得する
2. 指定されたレイアウトに応じて必要な分割操作の順序を計算する
3. enigo を通じてキーボード入力をシミュレートし、Ghostty のキーバインドを順番に発火させる

enigo は macOS では [Core Graphics Event API](https://developer.apple.com/documentation/coregraphics/cgevent)、Linux では [libxdo (xdotool)](https://github.com/jordansissel/xdotool) を利用してキーストロークを送信します。このクレートが OS ごとの差異を吸収してくれるため、同一のコードベースで macOS / Linux の両方に対応できています。

:::message
Windows は Ghostty 自体が未サポートのため、ghostty-pane-splitter も Windows には対応していません。
:::

## 終わりに

Ghostty のpane分割を CLI で自動化するツール ghostty-pane-splitter を紹介しました。よければご利用ください！

star や PR、Issue などもお待ちしています。

## 参考文献

https://zenn.dev/tackeyy/articles/deb12cbcb0f183

https://zenn.dev/meijin/articles/ghostty-pane-split-script
