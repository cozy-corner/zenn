---
title: "現在開いているページを NextDNS の拒否ドメインリストへワンクリック追加する"
emoji: "📰"
type: "tech"
topics:
  - "chrome拡張"
  - "dns"
published: true
published_at: "2025-05-19 22:51"
---


## はじめに
私は数年前から [NextDNS](https://nextdns.io/) というサービスを利用しています。
このサービスを使って、PC・スマホを共通で広告ブロックしています。
また、ドメイン単位での閲覧制限もかけています。

ドメイン単位のブロックは、主にニュースサイトやまとめサイトを対象にしています。
目的は、不必要なネットサーフィンによる時間の浪費を減らすことです。
さらに、事件・災害・芸能人の不祥事などで、心を不必要に煩わせることもできるだけ回避したいと思っています。

しかし、「URLからドメインをコピーして、NextDNS のコンソールで Denylist に登録する」という作業は、なかなか手間です。
そこで、NextDNS のコンソール画面に行かずに、ドメインのブロックを行う方法を調べました。

結論としては、Chrome のプラグイン([Tampermonkey](https://chromewebstore.google.com/detail/tampermonkey/dhdgffkkebhmkfjojejmpbldmpobfkfo))を利用し、NextDNS が提供する API エンドポイントへ POST するだけです。
しかし、Tampermonkeyの基本的なところも含めて意外とハマりどころがあったので、記事としてまとめることにしました。

## 1. NextDNS とは
- **DNS フィルタリング & 可視化サービス**  
- 広告・マルウェア・トラッカーを DNS レベルで遮断できる  
- 各ユーザーは 1 つ以上の **Profile** を持ち、Profile には **Denylist（拒否リスト）** と **Allowlist** がある  
- 公式 **REST API**（*API v1*）を通じて Denylist の編集を自動化できる

## 2. Tampermonkey とは
- Chrome／Brave／Edge で動く（今回はChromeで動作確認） **Userscript マネージャー拡張**  
- JavaScript をページに注入し、ポップアップメニューやホットキーで実行できる  
- `GM_registerMenuCommand()` でアイコンメニューに独自コマンドを追加できる

## 3. 必要な前提

| 項目 | 取得・参照先 |
|------|-------------|
| **Profile ID** | NextDNS API `GET /profiles` の `id` フィールド<br>またはダッシュボード URL に含まれる 6 文字 ID |
| **API Key** | NextDNS アカウントページ最下部（公式ドキュメント *Authentication* 章） |
| **Tampermonkey v 5.3 以上** | [Chrome ウェブストア](https://chrome.google.com/webstore) / [Tampermonkey 公式 β](https://tampermonkey.net) 版<br>拡張詳細で **開発者モード ON** と **サイトへのアクセス → すべてのサイト** に設定 |

## 4. Userscript の追加手順

1. **Tampermonkey アイコン → Dashboard → “＋ Create a new script…”** を開く  
2. エディタを空にし、以下を **丸ごとコピーして保存**  
3. `PROFILE_ID` と `API_KEY` を **自分の値** に書き換える（それ以外は変更しない）
  
```javascript
// ==UserScript==
// @name         NextDNS Quick-Block
// @namespace    nextdns-tools
// @version      2.2
// @description  表示中ドメインを 1 クリックで NextDNS の拒否リストへ追加
// @match        *://*/*
// @grant        GM_registerMenuCommand
// @grant        GM_xmlhttpRequest
// @connect      api.nextdns.io
// @run-at       document-end
// ==/UserScript==

// ─── ここを自分の値に書き換える ───
const PROFILE_ID = 'YOUR_PROFILE_ID';
const API_KEY    = 'YOUR_API_KEY';
// ─────────────────────────────

function currentDomain() {
  return location.hostname.replace(/^www\./, '');
}

function pop(msg, ok = true) {
  alert(`${ok ? '✅' : '❌'} ${msg}`);
}

function block(domain) {
  GM_xmlhttpRequest({
    method: 'POST',
    url: `https://api.nextdns.io/profiles/${PROFILE_ID}/denylist`,
    headers: {
      'X-Api-Key': API_KEY,
      'Content-Type': 'application/json'
    },
    data: JSON.stringify({ id: domain, active: true }),
    timeout: 8000,

    onload: r => {
      const body = r.responseText ? JSON.parse(r.responseText) : {};
      if (r.status === 200 || r.status === 204) {
        pop(`追加完了: ${domain}`);
      } else {
        pop(`失敗 (${r.status}): ${JSON.stringify(body)}`, false);
      }
    },

    onerror: e => pop(`通信エラー: ${e.error || e}`, false),
    ontimeout: () => pop('タイムアウト', false)
  });
}

GM_registerMenuCommand('NextDNS: block this domain', () => block(currentDomain()));
```


4. スクリプト名左の丸が **緑（Enabled）** であることを確認
5. ブロックしたいサイトを開き直し、**Tampermonkey アイコン → “NextDNS: block this domain”** をクリック![](https://storage.googleapis.com/zenn-user-upload/7fbef3a7d4ed-20250519.png)

   * `✅ 追加完了: example.com` が表示されれば成功（Denylist に即反映）
6. Denylistの確認
![Denylist](https://storage.googleapis.com/zenn-user-upload/fc7864b29fc8-20250519.png)

## 5. トラブルシューティング

以下はコードを正しく貼り付けても発生し得るケースのみ記載しています。

| 症状                        | 主な原因                          | 対処                                      |
| ------------------------- | ----------------------------- | --------------------------------------- |
| メニューはあるが左の丸が灰色（実行できない）    | そのタブでスクリプトが手動 OFF             | ポップアップで灰色の丸を **1 回クリックして緑にする**          |
| `❌ 401 unauthorized`      | API Key が無効／Profile と紐付いていない  | 正しいキーを取得して貼り替える                         |
| `❌ 404 notFound`          | Profile ID が誤っている             | ダッシュボード URL や `GET /profiles` で ID を再確認 |
| `❌ 0 (network)` またはタイムアウト | ネットワーク遮断・Firewall・NextDNS 側障害 | 接続状態を確認し、時間をおいて再試行                      |

## 6. まとめ

Profile ID と API Key を設定したこの Userscript を Tampermonkey に登録すれば、**閲覧中ドメインをワンクリックで NextDNS の Denylist に登録**できます。

これにより、ニュースサイトやまとめサイトを見かけたその場で即座にブロックできるようになります。
コンソール画面を開く手間も省けるので、日常的なフィルタ運用が圧倒的に楽になります。

余談ですが、私のニュースを目にすることを極力避ける習慣は [News Diet](https://www.amazon.co.jp/dp/476313860X) という書籍の影響です。
興味のある方はぜひ読んでみてください。

続編の記事になります。
[NextDNSのカスタムDenylistを使って、リンクを視覚的にマークする](https://zenn.dev/cozy_corner/articles/fe0b8fe28c758f)