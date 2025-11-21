# Claude Code による技術記事自動レビュー実装

**日付**: 2025-11-19

## 概要

GitHub Actions と Claude Code Action を使用して、Zenn技術記事の自動レビューワークフローを実装した。記事のPR作成時に、複数の観点からインラインコメントを自動投稿する。

## レビュー原則

### 技術文書の3原則

1. 読み手にとって明確か（専門用語の説明、前提知識の確認）
2. 構成が論理的か（要点が冒頭にあるか、全体像が示されているか）
3. 明確に伝わる書き方か（具体的、短文、能動態、肯定文）

### 6つのルール

1. 要点を冒頭に書く
2. 根拠・条件をペアで書く
3. 箇条書きや表で分けて書く
4. 写真や図で視覚的に書く
5. 適切に組み合わせて書く
6. 明確に伝わる文を書く

### 重点チェック項目

- 誇張表現や過度な断定
- 二重否定
- 冗長表現
- 文体の混在
- 一文が長すぎる（80字超目安）
- 根拠のない主張

### 読者に対する心がけ（結城浩「文章を書く心がけ」より）

- 誰がその文章を読むのか：読者の技術レベルや前提知識を想定しているか
- 読者は何を知っているか：既知から未知へ、段階的に説明しているか
- 適切な例を示す：抽象的な説明だけでなく、具体例やコード例があるか
- 読者と対話する気持ち：一方的な説明ではなく、読者の疑問を先回りしているか

### 文章の品質チェック（結城浩「校正の実例」より）

- おかしな重なり：同じ言葉や表現が不自然に重複していないか
- 呼応の不一致：主語と述語、指示語と被指示語が正しく対応しているか
- あいまいな表現：「これ」「それ」などの指示語が何を指すか明確か
- 並列の乱れ：箇条書きや対比の構造が統一されているか
- 冗長な文章：より簡潔に表現できる箇所はないか
- わかりにくい説明：複雑すぎる文や説明を分割・整理できないか
- 情報の欠落：読者が理解するために必要な情報が抜けていないか

### コメント形式

各コメントの冒頭に該当する観点を【】で明記:
- 例: 【一文が長すぎる】【根拠のない主張】【並列の乱れ】

## ワークフロー構成

### トリガー条件

```yaml
on:
  pull_request:
    types: [opened, synchronize]
    paths:
      - "articles/**/*.md"
      - "books/**/*.md"
```

記事ファイルの変更時にのみ実行。

### Claude Code Action の設定

```yaml
- uses: anthropics/claude-code-action@v1
  with:
    claude_code_oauth_token: ${{ secrets.CLAUDE_CODE_OAUTH_TOKEN }}
    show_full_output: true
    claude_args: '--allowedTools "Bash(gh pr view:*)" "Bash(gh pr diff:*)" "Bash(gh api:*)"'
```

#### 許可ツール

- `Bash(gh pr view:*)`: PR情報取得
- `Bash(gh pr diff:*)`: 差分取得
- `Bash(gh api:*)`: GitHub API 呼び出し

#### 権限

```yaml
permissions:
  contents: read
  pull-requests: read
  issues: read
  id-token: write
```

### プロンプト構成

1. レビュー原則の説明（上記すべて）
2. GitHub API の使用方法
3. `line` と `side` パラメータの詳細
4. 観点の明記方法

## GitHub API によるインラインコメント投稿

### line + side パラメータ（採用）

ファイルの実際の行番号を使用する方式:

```bash
gh api repos/$REPO/pulls/$PR/reviews \
  --method POST \
  --field commit_id="$COMMIT_ID" \
  --field event="COMMENT" \
  --field body="レビュー完了" \
  --field 'comments[][path]=articles/example.md' \
  --field 'comments[][line]=14' \
  --field 'comments[][side]=RIGHT' \
  --field 'comments[][body]=【一文が長すぎる】...'
```

**パラメータ**:
- `line`: ファイルの行番号（1から開始）
- `side`:
  - `RIGHT`: 新しいバージョン（追加された行）
  - `LEFT`: 古いバージョン（削除された行）
- `start_line`: 複数行コメントの開始行（オプション）

**利点**:
- ファイルの行番号をそのまま使用
- 新規ファイル・既存ファイルで同じ方法
- 計算不要で直感的

### position パラメータ（不採用）

diff内の相対位置を指定する方式:

```bash
-F 'comments[][position]=12'
```

**仕様**:
- @@ hunkヘッダーの次の行を position 1 とする
- context行、削除行、追加行すべてカウント

**問題点**:
- GitHub公式で非推奨（"The position parameter is closing down"）
- 計算が複雑
- LLMが正確に計算できない

**position の計算例**:
```
@@ -6,31 +6,31 @@ topics:    ← カウント開始（含まない）
   - "postgresql"              ← position 1
   - "sql"                     ← position 2
 -published: true              ← position 4
 +published: false             ← position 6
 +データベース設計において...    ← position 12
```

## テスト結果

### テスト1: 新規ファイル（PR #10）

**ファイル**: `articles/test-verify-inline.md`

**結果**:
- 20件のインラインコメント投稿成功
- 各コメントが正しい行番号に配置
- 観点が【】で明記

### テスト2: 既存ファイル修正（position 使用、PR #11）

**ファイル**: `articles/184bf3b88c9f2a.md`

**問題**:
- Claude Code が position を誤計算
- 例: 正しくは position 12 → 実際は position 5 を使用
- コメントが意図しない行に投稿

### テスト3: 既存ファイル修正（line+side 使用、PR #11）

**ファイル**: `articles/184bf3b88c9f2a.md`

**結果**: 全てのコメントが正しい行番号に投稿

| ファイル行 | side | コメント内容 | 結果 |
|-----------|------|--------------|------|
| 14 | RIGHT | 【一文が長すぎる】135文字 | ✓ |
| 16 | RIGHT | 【根拠のない主張】 | ✓ |
| 18 | RIGHT | 【冗長な表現】 | ✓ |
| 24 | RIGHT | 【誇張表現】 | ✓ |

**検証**:
```bash
gh api repos/cozy-corner/zenn/pulls/11/comments \
  --jq '.[] | {line: .line, side: .side, body: .body[0:50]}'
```

## 実装のポイント

### GitHub API パラメータ選択

- **`position`**: 非推奨、計算複雑、LLMに不向き
- **`line` + `side`**: 推奨、直感的、正確

### gh CLI の活用

```bash
gh api <endpoint> --method POST --field <key>=<value>
```

配列パラメータ:
```bash
--field 'comments[][line]=14'
--field 'comments[][side]=RIGHT'
```

### Claude Code の制約

- ツール使用には明示的な許可が必要
- `--allowedTools` パラメータで指定
- パターンはコロン区切り: `Bash(gh pr view:*)`

## ファイル構成

```
.github/workflows/
  ├── claude-code-review.yml    # 自動レビューワークフロー
  └── claude.yml                 # @claude メンション対応

articles/
  ├── test-verify-inline.md     # テスト用新規記事
  └── 184bf3b88c9f2a.md         # テスト用既存記事

docs/learning/
  └── claude-code-review-implementation.md  # 本ドキュメント
```

## 参考資料

- [GitHub REST API - Pull Request Reviews](https://docs.github.com/en/rest/pulls/reviews)
- [GitHub REST API - Pull Request Review Comments](https://docs.github.com/en/rest/pulls/comments)
- [Claude Code Action](https://github.com/anthropics/claude-code-action)
- 結城浩「数学文章作法」シリーズ
