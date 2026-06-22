---
title: "【セキュリティヘッダ】セキュリティヘッダを整理する"
emoji: "🕸️"
type: "idea"
topics: 
  - "security"
  - "http"
  - "web"
published: false
---

## はじめに

HTTPレスポンスヘッダを適切に設定・削除することで、XSS、クリックジャッキング、情報漏えいなどのリスクを低減できます。本記事では、OWASPをもとに、設定が推奨されるヘッダと削除が推奨されるヘッダを整理します。

出典は、OWASP Secure Headers Project（以下、OSHP）と、OWASP Cheat Sheet Series の HTTP Headers Cheat Sheet（以下、Cheat Sheet）です。推奨値はOSHPを基準とし、両者で異なる箇所は本文で補足します。

前提として、これらのヘッダはレスポンスを処理するクライアントがブラウザである場合に効果を持ち、ブラウザ以外のクライアントはこれらを強制しません。なお、推奨値や各ヘッダの扱いは変わりうるため、参照時には出典で最新の情報を確認してください。

## ヘッダのライフサイクル分類

OSHPはヘッダを次の3つに分類しています。これが設定対象と削除対象を判断する起点になります。

| 分類                      | ヘッダ                                                                                                                                                                                                                                                                                          |
| ------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Active（設定対象）        | Strict-Transport-Security, X-Frame-Options, X-Content-Type-Options, Content-Security-Policy, X-Permitted-Cross-Domain-Policies, Referrer-Policy, Clear-Site-Data, Cross-Origin-Embedder-Policy, Cross-Origin-Opener-Policy, Cross-Origin-Resource-Policy, Cache-Control, X-DNS-Prefetch-Control |
| Working draft（設定対象） | Permissions-Policy                                                                                                                                                                                                                                                                              |
| Deprecated（削除対象）    | Feature-Policy, Expect-CT, Public-Key-Pins, X-XSS-Protection, Pragma                                                                                                                                                                                                                            |

## 設定が推奨されるヘッダ

脅威カテゴリ別に解説します。推奨値はOSHPの構成案によります。

### 通信路の保護

対象ヘッダは以下です。

- Strict-Transport-Security（HSTS）

#### Strict-Transport-Security（HSTS）

HTTPでの接続試行時もブラウザにHTTPSでのみアクセスさせ、プロトコルダウングレード攻撃やCookieハイジャックを抑止します。HTTPS接続で受信した場合のみ有効で、HTTP上で受信したものは無視されます。

```
Strict-Transport-Security: max-age=63072000; includeSubDomains
```

| ディレクティブ      | 意味                                                                     |
| ------------------- | ------------------------------------------------------------------------ |
| `max-age=<秒>`      | ブラウザがHTTPSを強制する期間                                            |
| `includeSubDomains` | すべてのサブドメインにも適用する                                         |
| `preload`           | ブラウザのプリロードリストへの登録に同意する（別途リストへの申請が必要） |

Cheat Sheet は末尾に `preload` を付与します。この指定でプリロードリストに登録すると、HTTPへ戻す際にアクセス不能となる恐れがあるため、影響を理解したうえで判断します。

### XSS・インジェクション対策

対象ヘッダは以下です。

- Content-Security-Policy（CSP）
- X-Content-Type-Options

#### Content-Security-Policy（CSP）

読み込めるコンテンツのオリジンを指定し、XSS・インジェクション・クリックジャッキングを抑止します。

```
Content-Security-Policy: default-src 'self'; form-action 'self'; base-uri 'self'; object-src 'none'; frame-ancestors 'none'; upgrade-insecure-requests
```

OSHPの推奨値に含まれる主なディレクティブは次のとおりです。

| ディレクティブ              | 意味                                                                 |
| --------------------------- | -------------------------------------------------------------------- |
| `default-src`               | 明示されていないリソース種別の既定の取得元                           |
| `form-action`               | フォームの送信先として許可するURL                                    |
| `base-uri`                  | `<base>` 要素で指定できるURL                                         |
| `object-src`                | `<object>` や `<embed>` の取得元                                     |
| `frame-ancestors`           | このページをフレームに埋め込めるオリジン（クリックジャッキング対策） |
| `upgrade-insecure-requests` | HTTPでのサブリソース取得をHTTPSに自動的に変更する                    |

CSPには上記以外にも `script-src` や `style-src` など多数のディレクティブがあります。設定が複雑で定義後の検証が必要なこと、またXSS対策としては取得元の許可リストよりも nonce や hash を用いる方式（Strict CSP）が推奨されることから、詳細は CSP Cheat Sheet を参照してください。なお、XSS対策の唯一の手段とはせず、追加の防御層として導入します。

#### X-Content-Type-Options

宣言されたMIMEタイプをブラウザに尊重させ、MIMEスニッフィングを悪用した攻撃を抑止します。Content-Typeが正しく設定されていることが前提です。

```
X-Content-Type-Options: nosniff
```

値は `nosniff` のみで、MIMEタイプの推測を無効化します。

### クリックジャッキング対策

対象ヘッダは以下です。

- X-Frame-Options

#### X-Frame-Options

ページを `<frame>`・`<iframe>`・`<embed>`・`<object>` に埋め込めるかを制御し、クリックジャッキングを抑止します。

```
X-Frame-Options: deny
```

| 値           | 意味                                         |
| ------------ | -------------------------------------------- |
| `deny`       | 埋め込みを一切許可しない                     |
| `sameorigin` | 同一オリジンのページへの埋め込みのみ許可する |

かつて存在した `allow-from` は、現在のブラウザでは機能せず非推奨です。CSPの `frame-ancestors` が同じ役割を持ち、両方ある場合はそちらが優先されます。本ヘッダは `frame-ancestors` 非対応の古いブラウザ向けの後方互換として併用されます（OSHP分類上はActiveです）。

### クロスオリジン分離

対象ヘッダは以下です。

- Cross-Origin-Opener-Policy（COOP）
- Cross-Origin-Embedder-Policy（COEP）
- Cross-Origin-Resource-Policy（CORP）

いずれもクロスオリジンのリソースとの境界を制御するブラウザ向けのヘッダで、Spectreのようなサイドチャネル攻撃やXS-Leaksの抑止に関連します。COOPとCOEPを組み合わせると、SharedArrayBufferなどを利用するための「クロスオリジン分離」が有効になります。

#### Cross-Origin-Opener-Policy（COOP）

トップレベル文書がクロスオリジン文書とブラウジングコンテキストグループを共有しないようにし、ポップアップ経由で参照を悪用するXS-Leaksを抑止します。

```
Cross-Origin-Opener-Policy: same-origin
```

| 値                         | 意味                                                                   |
| -------------------------- | ---------------------------------------------------------------------- |
| `unsafe-none`              | 既定値。クロスオリジン文書との共有を制限しない                         |
| `same-origin`              | 同一オリジンかつ同じ値の文書とのみ共有する（クロスオリジン分離に必要） |
| `same-origin-allow-popups` | `same-origin` に加え、自身が開いたポップアップへの参照を保持する       |

#### Cross-Origin-Embedder-Policy（COEP）

no-corsモードで読み込むクロスオリジンリソースに、明示的な許可を要求します。リソースを読み込む側に適用されます。

```
Cross-Origin-Embedder-Policy: require-corp
```

| 値               | 意味                                                   |
| ---------------- | ------------------------------------------------------ |
| `unsafe-none`    | 既定値。許可なしにクロスオリジンリソースを読み込める   |
| `require-corp`   | CORPまたはCORSで許可されたリソースのみ読み込む         |
| `credentialless` | CORPなしでも読み込むが、資格情報（Cookie等）を送らない |

`require-corp` を設定すると、CORPまたはCORSで許可されていないクロスオリジンリソースはブロックされます。導入時はリソース側の設定とあわせた検証が必要です。

#### Cross-Origin-Resource-Policy（CORP）

リソースを読み込めるオリジンの範囲を制御し、サイドチャネル攻撃やクロスサイトスクリプトインクルージョン（XSSI）を抑止します。リソースを提供する側に適用されます。

```
Cross-Origin-Resource-Policy: same-origin
```

| 値             | 意味                                                     |
| -------------- | -------------------------------------------------------- |
| `same-origin`  | 同一オリジンからの読み込みのみ許可する                   |
| `same-site`    | 同一サイト（サブドメインを含む）からの読み込みを許可する |
| `cross-origin` | 任意のオリジンからの読み込みを許可する                   |

Cheat Sheet は `same-site` を提示します。配信構成に応じて選択します。

### 情報漏えいの抑制

対象ヘッダは以下です。

- Referrer-Policy
- Cache-Control
- Clear-Site-Data

#### Referrer-Policy

Refererヘッダで送るリファラ情報の量を制御し、リファラ経由の情報漏えいを抑止します。

```
Referrer-Policy: no-referrer
```

主な値は次のとおりです（厳格な順）。

| 値                                | 意味                                                                              |
| --------------------------------- | --------------------------------------------------------------------------------- |
| `no-referrer`                     | Refererを一切送らない                                                             |
| `same-origin`                     | 同一オリジンには完全なURLを送り、クロスオリジンには送らない                       |
| `strict-origin`                   | オリジンのみを送る。ただしHTTPS→HTTPでは送らない                                  |
| `strict-origin-when-cross-origin` | 同一オリジンには完全なURL、クロスオリジンにはオリジンのみ、HTTPS→HTTPでは送らない |

Cheat Sheet は `strict-origin-when-cross-origin` を推奨します。これは近年のブラウザの既定に近い挙動です。このほか `origin`・`origin-when-cross-origin`・`no-referrer-when-downgrade`・`unsafe-url` がありますが、`unsafe-url` は常に完全なURLを送るため推奨されません。

#### Cache-Control

レスポンスのキャッシュ方法を制御し、キャッシュ経由での機微情報の露出を抑止します。

```
Cache-Control: no-store, max-age=0
```

| ディレクティブ | 意味                                                                  |
| -------------- | --------------------------------------------------------------------- |
| `no-store`     | いかなるキャッシュにも保存させない                                    |
| `no-cache`     | 保存は許可するが、再利用前に必ず再検証させる                          |
| `private`      | 共有キャッシュ（CDN・プロキシ等）には保存させず、ブラウザのみ許可する |
| `max-age=<秒>` | レスポンスが新鮮とみなされる期間                                      |

`no-cache` はキャッシュを防ぐ指定ではなく、再検証を求める指定です。保存自体を防ぐには `no-store` を使います。CDN等の共有キャッシュにも残したくない機微なレスポンスでは `private` を併用します。

#### Clear-Site-Data

リクエスト元のサイトに関連付けられた、クライアント側の保存データを削除します。主にログアウト処理で使用します。

```
Clear-Site-Data: "cache","cookies","storage"
```

| 値          | 意味                                                 |
| ----------- | ---------------------------------------------------- |
| `"cache"`   | ブラウザのキャッシュを削除する                       |
| `"cookies"` | CookieとHTTP認証情報を削除する                       |
| `"storage"` | DOMストレージ（localStorage、IndexedDB等）を削除する |
| `"*"`       | すべての種別を削除する                               |

処理コストがあるため、OSHPはログアウト機能に限定して設定することを推奨しています。

### 機能・権限の制御

対象ヘッダは以下です。

- Permissions-Policy

#### Permissions-Policy

どのオリジンがどのブラウザ機能を使えるかを制御し、XSS等を通じたカメラ・マイク・位置情報などの不正な有効化を抑止します。`機能=(許可リスト)` の形式で、不要な機能を無効化します。

```
Permissions-Policy: geolocation=(), camera=(), microphone=()
```

許可リストに指定できる主な値は次のとおりです。

| 値                      | 意味                                       |
| ----------------------- | ------------------------------------------ |
| `()`                    | 空の許可リスト。その機能をすべて無効化する |
| `self`                  | 同一オリジンでのみ許可する                 |
| `*`                     | すべてのオリジンで許可する                 |
| `"https://example.com"` | 指定したオリジンで許可する                 |

OSHPは多数の機能を網羅的に無効化する値を提示しています。本ヘッダはWorking draftで、主にChromium系がサポートします。OSHPの推奨値にはFLoC拒否用の `interest-cohort=()` が含まれますが、FLoCはGoogleが2022年に開発を終了し、後継のTopics APIも2025年に非推奨化されています。

### その他のActive分類のヘッダ

対象ヘッダは以下です。

- X-Permitted-Cross-Domain-Policies
- X-DNS-Prefetch-Control

いずれも適用範囲が限定的なため、簡潔に示します。

#### X-Permitted-Cross-Domain-Policies

FlashやAcrobatなどが参照するクロスドメインポリシーファイルの扱いを制御します。

```
X-Permitted-Cross-Domain-Policies: none
```

主な値は、`none`（ポリシーファイルを一切許可しない）、`master-only`（マスターポリシーファイルのみ許可）、`all`（すべて許可）です。対象のレガシー技術を利用していない環境では `none` を設定します。

#### X-DNS-Prefetch-Control

ブラウザのDNSプリフェッチ（参照先ドメインの名前解決を先行して行う機能）を制御します。

```
X-DNS-Prefetch-Control: off
```

値は `on`（有効）と `off`（無効）です。非標準で主にChromium系が対象です。OSHPは、機微なコンテンツを返すレスポンスやHTMLインジェクションの余地があるレスポンスへの付与を提案しています。

### 設定が推奨されるヘッダのまとめ

| ヘッダ                            | 推奨値                                | 抑止対象                                    |
| --------------------------------- | ------------------------------------- | ------------------------------------------- |
| Strict-Transport-Security         | `max-age=63072000; includeSubDomains` | ダウングレード、Cookieハイジャック          |
| Content-Security-Policy           | `default-src 'self'; ...`             | XSS、インジェクション、クリックジャッキング |
| X-Content-Type-Options            | `nosniff`                             | MIMEスニッフィング                          |
| X-Frame-Options                   | `deny`                                | クリックジャッキング                        |
| Cross-Origin-Opener-Policy        | `same-origin`                         | XS-Leaks                                    |
| Cross-Origin-Embedder-Policy      | `require-corp`                        | サイドチャネル攻撃                          |
| Cross-Origin-Resource-Policy      | `same-origin`                         | サイドチャネル攻撃、XSSI                    |
| Referrer-Policy                   | `no-referrer`                         | リファラ経由の情報漏えい                    |
| Cache-Control                     | `no-store, max-age=0`                 | キャッシュ経由の情報露出                    |
| Clear-Site-Data                   | `"cache","cookies","storage"`         | ログアウト時のデータ残存                    |
| Permissions-Policy                | 機能を無効化する値                    | 機能の不正な有効化                          |
| X-Permitted-Cross-Domain-Policies | `none`                                | クロスドメインポリシーの悪用                |
| X-DNS-Prefetch-Control            | `off`                                 | DNS経由の情報持ち出し                       |

## 削除が推奨されるヘッダ

削除対象は、機能的に非推奨となったヘッダ（Deprecated分類）と、環境の技術情報を開示するヘッダの2系統です。後者はリバースプロキシやWAF、アプリケーション側で送出しないよう設定します。

| ヘッダ                                 | 分類       | 削除理由                                                       |
| -------------------------------------- | ---------- | -------------------------------------------------------------- |
| X-XSS-Protection                       | Deprecated | 安全なサイトにXSS脆弱性を生む場合がある                        |
| Expect-CT                              | Deprecated | 主要クライアントが証明書透明性（CT）準拠を必須化し役割が限定的 |
| Public-Key-Pins                        | Deprecated | 運用上の脆さ、主要ブラウザでサポート終了                       |
| Feature-Policy                         | Deprecated | Permissions-Policyへ置き換え                                   |
| Pragma                                 | Deprecated | Cache-Controlでの定義が推奨                                    |
| Server                                 | 情報開示   | サーバ情報を開示                                               |
| X-Powered-By                           | 情報開示   | 環境・フレームワーク情報を開示                                 |
| X-AspNet-Version / X-AspNetMvc-Version | 情報開示   | .NETのバージョンを開示                                         |

補足が2点あります。X-XSS-Protection は、ヘッダ自体の削除ではなく `X-XSS-Protection: 0` による無効化がOWASPの推奨です。Pragma は、Cache-Controlを持たないHTTP/1.0キャッシュとの互換が必要な場合のみ補助的に併用されます。

技術情報を開示するヘッダは、OSHPの「headers to remove」一覧に各種CMS・フレームワーク・プロキシ固有のものが多数挙げられています。網羅的な一覧はOSHPを参照してください。なお、削除は容易な開示経路を塞ぐ対策であり、攻撃者は他の手段でも技術を特定しうる点に留意します。

## まとめ

OSHPのライフサイクル分類を起点に、Active と Working draft のヘッダを設定し、Deprecated のヘッダと技術情報を開示するヘッダを削除する、というのが基本方針です。ただしX-XSS-Protectionは `0` で無効化します。対象がブラウザ向けかAPI向けか、また出典による推奨値の差を踏まえ、アプリケーションの文脈に応じて取捨選択してください。

**参考：**

- [OWASP Secure Headers Project](https://owasp.org/www-project-secure-headers/)
- [HTTP Headers Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/HTTP_Headers_Cheat_Sheet.html)
- [Content Security Policy Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Content_Security_Policy_Cheat_Sheet.html)
- [MDN HTTP ヘッダリファレンス](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers)
- [MDN HTTP Observatory](https://developer.mozilla.org/en-US/observatory/)
