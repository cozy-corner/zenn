# Zenn CLI

* [📘 How to use](https://zenn.dev/zenn/articles/zenn-cli-guide)

## 文章校正

ローカルでチェック:
```bash
npm install
npm run lint
```

PRを作成すると自動でtextlintが実行され、reviewdogが指摘をコメントします。

### textlintルール

#### 基本設定
- **sentence-length**: 1文の長さを80文字以内に制限
- **max-kanji-continuous-len**: 連続する漢字を6文字以内に制限
- **no-mix-dearu-desumasu**: 本文は「ですます調」、リストは「である調」に統一

#### 表記とスペーシング
- **ja-space-between-half-and-full-width**: 全角文字と半角文字の間にスペースを入れる
- **ja-no-space-around-parentheses**: かっこの外側・内側にスペースを入れない

#### 表記ゆれと技術用語
- **prh**: 技術用語の表記ゆれをチェック（JavaScript、GitHub、TypeScript、VS Code、macOS、Node.js）
- **spellcheck-tech-word**: 技術用語のスペルチェック

#### 文章品質
- **ja-no-weak-phrase**: 弱い表現（「〜と思います」「〜かもしれません」）を検出
- **ja-no-redundant-expression**: 冗長な表現（「〜することができます」→「〜できます」）を検出
- **ja-unnatural-alphabet**: 不自然なアルファベット（全角英字）を検出
- **no-doubled-joshi**: 二重助詞（「には」「には」など）を検出
- **no-doubled-conjunction**: 連続する接続詞（「しかし」「しかし」など）を検出
- **no-double-negative-ja**: 二重否定（「〜ないことはない」）を検出
- **no-dropping-the-ra**: ら抜き言葉（「見れる」→「見られる」）を検出
- **no-doubled-conjunctive-particle-ga**: 連続する「が」を検出
- **ja-no-successive-word**: 連続する同じ単語（「このこの」）を検出
- **ja-no-abusage**: よくある誤用を検出（例：「例外を補足」→「例外を捕捉」）
- **arabic-kanji-numbers**: 算用数字と漢数字の使い分けをチェック
- **no-invalid-control-character**: 無効な制御文字を検出