---
title: "Claude Codeが非インタラクティブシェルでエイリアスを使える仕組み"
emoji: "🐚"
type: "tech"
topics: ["shell", "zsh", "claudecode", "cli"]
published: false
---

## はじめに

Claude Codeを使っていて、不思議に思ったことはありませんか？

普段ターミナルで使っている `ll` や `gs` といったエイリアスが、Claude Code内でもそのまま使えるのです。しかし、通常の非インタラクティブシェル（スクリプト実行など）では、`.zshrc` が読み込まれないため、エイリアスは使えないはずです。

```bash
# 通常のスクリプトではエイリアスは使えない
$ cat > test.sh << 'EOF'
#!/bin/zsh
ll  # .zshrcで定義したエイリアス
EOF

$ chmod +x test.sh
$ ./test.sh
./test.sh:2: command not found: ll
```

それなのに、Claude Codeでは動く。一体どうやって実現しているのでしょうか？

この疑問から、Claude Codeの内部実装を調査してみました。

## 問題の本質

### シェルの種類と設定ファイル

シェルには「インタラクティブ」と「非インタラクティブ」という区別があります。

| | インタラクティブ | 非インタラクティブ |
|---|---|---|
| 例 | ターミナルで対話的に使う | スクリプト実行 |
| プロンプト | あり | なし |
| `$-` の内容 | `i` が含まれる | `i` がない |

そして、zshの設定ファイルの読み込みルールは以下の通りです：

| | 読み込まれるファイル |
|---|---|
| **インタラクティブ** | `.zshenv` → `.zprofile` → `.zshrc` |
| **非インタラクティブ** | `.zshenv` のみ |

**重要**: `.zshrc` はインタラクティブシェルの時のみ読み込まれます。つまり、スクリプト実行では `.zshrc` で定義したエイリアスや関数は使えないのです。

### Claude Codeのシェルは非インタラクティブ

実際にClaude Code内で確認してみましょう：

```bash
$ echo $-
569Xl  # 'i' がない → 非インタラクティブ

$ alias ll
ll='eza -la --icons'  # それなのにエイリアスがある！

$ ll
# → 動作する
```

非インタラクティブなのにエイリアスが使える。これは一体どういうことでしょうか？

## スナップショット機構の発見

調査を進めると、`~/.claude/shell-snapshots/` に興味深いファイルが見つかりました。

このファイルの中身を見てみると：

```bash
# Snapshot file
# Unset all aliases to avoid conflicts with functions
unalias -a 2>/dev/null || true

# Functions
bashcompinit () { ... }
gbr () { ... }  # .zshrcで定義されたユーザー関数
gc () { ... }

# Aliases
alias ll='eza -la --icons'
alias gs='git status'
...

# Environment variables
export PATH='/Users/...'
```

なんと、`.zshrc` で定義した関数やエイリアスが、すべてこのファイルに保存されていました！

### ファイルの構造

**保存場所**: `~/.claude/shell-snapshots/`
**ファイル名**: `snapshot-zsh-<タイムスタンプ>-<ランダム文字列>.sh`
**内容**:
- 関数定義
- エイリアス
- 環境変数

**ライフサイクル**:
- 作成: Claude Codeのセッション起動時
- 削除: 正常終了時
- 保持: 異常終了時（残ってしまう）

## 仕組みの全体像

Claude Codeは、以下の2段階で環境を再現しています：

```
┌─────────────────────────────────┐
│ 1. セッション起動時（1回のみ）  │
├─────────────────────────────────┤
│ zsh -l -c "スクリプト"          │
│   ↓                             │
│ .zshenv → .zprofile を読み込む  │
│   ↓                             │
│ source ~/.zshrc < /dev/null     │
│   ↓                             │
│ 関数・エイリアスを抽出          │
│   ↓                             │
│ スナップショットファイルに保存  │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│ 2. 各コマンド実行時（毎回）      │
├─────────────────────────────────┤
│ zsh -l -c "..."                 │
│   ↓                             │
│ .zshenv → .zprofile を読み込む  │
│   ↓                             │
│ source <スナップショット>       │
│   ↓                             │
│ ユーザーコマンドを実行          │
└─────────────────────────────────┘
```

一度だけ環境をキャプチャして、毎回それを復元する設計です。毎回 `.zshrc` を読み込む必要がないため、高速に動作します。

## 実装の工夫

### 1. スナップショット作成の詳細

Claude Codeの実装（`cli.js`）では、スナップショット作成スクリプトを生成する関数が定義されています：

```javascript
async function mSY(A,q,K){
  let Y=qyA(A),  // .zshrc または .bashrc のパス
  ...
  return`SNAPSHOT_FILE=${L7([q])}
    ${K?`source "${Y}" < /dev/null`:"# No user config file to source"}

    # First, create/clear the snapshot file
    echo "# Snapshot file" >| "$SNAPSHOT_FILE"

    # Unset all aliases to avoid conflicts with functions
    echo "unalias -a 2>/dev/null || true" >> "$SNAPSHOT_FILE"

    ${w}  // 関数定義を出力
    ${H}  // 環境変数を出力
    ...
  `
}
```

このスクリプトは `zsh -l -c` で実行され、以下の処理を行います：

```bash
# 1. 関数定義を取得
echo "# Functions" >> "$SNAPSHOT_FILE"
typeset -f > /dev/null 2>&1
typeset +f | grep -vE '^(_|__)' | while read func; do
  typeset -f "$func" >> "$SNAPSHOT_FILE"
done

# 2. エイリアス
echo "# Aliases" >> "$SNAPSHOT_FILE"
alias | sed 's/^alias //g' | sed 's/^/alias -- /' >> "$SNAPSHOT_FILE"

# 3. 環境変数
export PATH="..."
```

重要なポイント：
- **`-l` (ログインシェル)**: `.zprofile` も読み込まれるため、`PATH` などの環境変数が正しく設定される
- **`source "${Y}" < /dev/null`**: 明示的に `.zshrc` を読み込む（非インタラクティブでも）
- **`< /dev/null`**: 標準入力をクローズして、対話的なプロンプトを防ぐ

### 2. 冪等性の確保

スナップショットファイルは、以下のように始まります：

```bash
# Unset all aliases to avoid conflicts with functions
unalias -a 2>/dev/null || true
```

なぜ最初にすべてのエイリアスをクリアするのでしょうか？

zshでは、同じ名前のエイリアスと関数がある場合、**エイリアスが優先**されます：

```bash
# 既存のエイリアス
alias foo='echo "alias"'

# スナップショットで関数を定義
foo() { echo "function"; }

# 結果: エイリアスが優先される
$ foo
alias  # 期待: function
```

この問題を避けるため、スナップショットは以下の順序で実行されます：

```bash
1. unalias -a          # すべてのエイリアスをクリア
2. 関数定義            # 関数を定義
3. エイリアス定義      # エイリアスを定義
```

これにより、何度 `source` しても同じ結果が得られます（冪等性）。

### 3. 各コマンド実行時の復元

Claude Codeでコマンドを実行すると、内部で以下の関数が呼び出されます：

```javascript
async function nW6(...){
  let {binShell:_,snapshotFilePath:J}=await zyA();  // キャッシュ取得
  if(J){
    if(!QSY(J))  // ファイルが存在しない場合は再作成
      h(`Snapshot file missing, recreating: ${J}`),
      zyA.cache?.clear?.(),
      J=(await zyA()).snapshotFilePath;
    if(J){
      let B=tA()==="windows"?cx(J):J;
      G.push(`source ${L7([B])}`)  // スナップショットをsource
    }
  }
}
```

実際に実行されるコマンドは以下の形式です：

```bash
zsh -l -c "source ~/.claude/shell-snapshots/snapshot-zsh-*.sh; <ユーザーコマンド>"
```

実行の流れ：
1. zsh起動（非インタラクティブ、`-l`フラグ付き）
2. `.zshenv`、`.zprofile` を読み込む
3. `source ~/.claude/shell-snapshots/snapshot-zsh-*.sh` でスナップショットを復元
4. ユーザーのコマンドを実行

スナップショットファイルが消えていた場合は、自動的に再作成されます。

## sourceコマンドの重要性

この仕組みの鍵となるのが `source` コマンドです。

### 直接実行との違い

```bash
# 直接実行（./script.sh）
┌─────────────┐
│ 親シェル    │
└──────┬──────┘
       │ fork + exec
       ↓
┌─────────────┐
│ サブシェル  │  ← 新しいプロセス
│(スクリプト) │     変数はサブシェル内のみ
└─────────────┘

# source script.sh
┌─────────────┐
│ 現在のシェル│  ← 同じプロセス内
│             │     変数・関数が残る
└─────────────┘
```

### sourceの特徴

- **現在のシェル内で実行**: 変数、関数、エイリアスがすべて現在のシェルに残る
- **実行権限不要**: ファイルを読み込むだけなので、実行権限（`chmod +x`）は不要
- **シェバン不要**: 現在のシェルで実行されるため、`#!/bin/zsh` も不要

スナップショットファイルのパーミッションを見てみると：

```bash
$ ls -l ~/.claude/shell-snapshots/
-rw-r--r--  snapshot-zsh-*.sh  # 実行権限なし（644）
```

`source` で読み込むため、実行権限は不要なのです。

## 設計の利点

この仕組みには、いくつかの優れた点があります：

### 1. 高速

毎回 `.zshrc` を読み込むのではなく、スナップショットを `source` するだけなので高速です。

### 2. 非インタラクティブでもユーザー環境を再現

通常は不可能な「非インタラクティブシェルでエイリアスを使う」を実現しています。

### 3. シェルの種類に対応

bash、zshなど、複数のシェルに対応しています。

### 4. 環境の完全な再現

ログインシェル（`-l`）を使うことで、`.zprofile` も読み込まれ、`PATH` などの環境変数が正しく設定されます。

## まとめ

Claude Codeが非インタラクティブシェルでエイリアスを使える仕組みは：

1. **セッション起動時**: `zsh -l -c` で `.zprofile` と `.zshrc` を読み込み、環境をスナップショット
2. **コマンド実行時**: スナップショットを `source` で復元してから実行

この「一度キャプチャ、何度も復元」という設計により：
- 高速な実行
- ユーザー環境の完全な再現
- 非インタラクティブシェルでもエイリアスが使える

という3つのメリットを実現しています。

## 参考資料

- 調査環境: macOS, zsh, Claude Code 2.1.38
- 実装ファイル: `/opt/homebrew/lib/node_modules/@anthropic-ai/claude-code/cli.js`
- zsh マニュアル: `man zsh` → "STARTUP/SHUTDOWN FILES"
