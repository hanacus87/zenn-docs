---
title: "認証・認可の仕組みと脆弱性 - SAML / OAuth 2.0 / OIDC"
free: false
---

:::message alert
**重要な法的注意事項**

本記事で紹介する技術は、セキュリティ教育および自身が管理するシステムでの脆弱性診断を目的としたものです。**許可なく他者のシステムに対してこれらの技術を使用することは、不正アクセス禁止法等の法律に違反し、刑事罰の対象となります。**

- 必ず事前に書面による許可を得てください
- 自身が所有・管理するシステム、または許可されたテスト環境でのみ使用してください
- 本記事の内容を悪用した場合、筆者は一切の責任を負いません
  :::

## 認証と認可の違い

```
認証（Authentication）：「あなたは誰？」を確認するプロセス
認可（Authorization）：「あなたは何をしていい？」を制御するプロセス
```

```mermaid
flowchart LR
    A[社員証を提示] -->|本人確認| B[認証<br/>あなたはAさんですね]
    B -->|権限確認| C[認可<br/>Aさんは3Fの入室を許可]
    C --> D[ドアが開く]
```

この2つは役割が異なりますが実装上は密接に絡み合っており、片方のミスがもう片方の崩壊につながるケースが多くあります。

**主要用語**

| 用語                            | 説明                                                                                                                                                      |
| ------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **IdP<br/>(Identity Provider)** | ユーザーのアイデンティティを管理・証明する主体（例：Google、Azure AD）                                                                                    |
| **SP<br/>(Service Provider)**   | IdP を信頼してサービスを提供する主体（例：社内業務システム）                                                                                              |
| **SSO<br/>(Single Sign-On)**    | 1度のログインで複数サービスにアクセスできる仕組み                                                                                                         |
| **JWT<br/>(JSON Web Token)**    | JSONをBase64URLエンコードし署名したトークン形式                                                                                                           |
| **アサーション**                | IdPが発行する「このユーザーは認証済み」という証明書（SAMLで使用）                                                                                         |
| **スコープ**                    | OAuthで要求するアクセス権限の範囲（例：`read:email`）                                                                                                     |
| **クレーム**                    | JWTのペイロードに含まれる情報の単位（例：`iss`,`aud`,`exp`,`nonce`）<br/>「このトークンは誰が発行し、誰宛で、いつ期限が切れるか」といったメタデータを表す |

## SAML 2.0 の仕組みと脆弱性

### SAMLとは

SAML（Security Assertion Markup Language）2.0 は XMLベースのSSO標準仕様です。主にエンタープライズ環境で「社内ポータルにログインすれば、他の業務アプリにも自動でログインできる」ような仕組みを実現します。

### SP-Initiated フロー

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant SP as SP（サービス提供者）
    participant IdP as IdP（認証基盤）

    User->>SP: 保護リソースへアクセス
    SP->>User: 未認証 → SAMLリクエストを生成しIdPへリダイレクト
    User->>IdP: SAMLリクエストを転送
    IdP->>User: ログイン画面を表示
    User->>IdP: 認証情報（ID/PW）を送信
    IdP->>User: SAMLレスポンス（署名付きアサーション）を返す
    User->>SP: SAMLレスポンスをPOSTで転送（ブラウザ経由）
    SP->>SP: 署名検証・アサーション解析
    SP->>User: ログイン完了・リソースへのアクセスを許可
```

ポイントはユーザーのブラウザが仲介者になっていることです。IdPとSPは直接通信せず、SAMLアサーション（XML）はIdPの秘密鍵で署名され、SPが公開鍵で検証します。

### SAMLアサーションの構造

```xml
<samlp:Response>
  <saml:Assertion ID="_assertion01">
    <saml:Issuer>https://idp.example.com</saml:Issuer>
    <ds:Signature>
      <ds:SignedInfo>
        <!-- "_assertion01" に対する署名 -->
        <ds:Reference URI="#_assertion01">...</ds:Reference>
      </ds:SignedInfo>
      <ds:SignatureValue>...</ds:SignatureValue>
    </ds:Signature>
    <saml:Subject>
      <saml:NameID>user@example.com</saml:NameID>
    </saml:Subject>
    <saml:Conditions NotBefore="2024-01-01T00:00:00Z"
                     NotOnOrAfter="2024-01-01T00:10:00Z"/>
    <saml:AttributeStatement>
      <saml:Attribute Name="role">
        <saml:AttributeValue>admin</saml:AttributeValue>
      </saml:Attribute>
    </saml:AttributeStatement>
  </saml:Assertion>
</samlp:Response>
```

### 典型的な脆弱性

#### XML Signature Wrapping（XSW）攻撃

XML署名は「特定のノード（ID参照）」に付与されますが、SPが検証後に実際に読むノードを別の場所に差し替えることで、有効な署名のまま偽のアサーションを受け入れさせます。

```mermaid
sequenceDiagram
    actor Attacker as 攻撃者
    participant SP as SP

    Note over Attacker: 正規のSAMLレスポンスを入手
    Attacker->>Attacker: 署名済み正規アサーション（A）を保持
    Attacker->>Attacker: 悪意あるアサーション（B：admin権限）を作成
    Attacker->>Attacker: XMLを細工<br/>署名の Reference URI="#A" のまま<br/>SPが最初に読む位置にBを配置
    Attacker->>SP: 細工したSAMLレスポンスをPOST
    SP->>SP: 署名検証：A を検証 → 正当
    SP->>SP: アサーション取得：B（admin）を使用
    Note over SP: 攻撃者がadminとしてログイン
```

細工されたXMLの構造は以下のようになります。

```xml
<samlp:Response>
  <!-- SPが実際に読むノード（悪意あるB）を先頭に配置 -->
  <saml:Assertion ID="_evil">
    <saml:NameID>admin@example.com</saml:NameID>
    <!-- 署名なし -->
  </saml:Assertion>

  <!-- 署名はAに対して有効なまま -->
  <saml:Assertion ID="_assertion01">
    <saml:NameID>user@example.com</saml:NameID>
    <ds:Signature>
      <ds:Reference URI="#_assertion01">...</ds:Reference>
    </ds:Signature>
  </saml:Assertion>
</samlp:Response>
```

根本原因は、署名検証に使ったノードと、その後データ取得に使うノードが一致していないことです。「検証したノード以外は使わない」ことが対策になります。

#### 署名検証の省略・設定ミス

ライブラリの設定で署名検証をスキップできてしまうケースがあります。CVE-2024-45409（Ruby-SAML）はこの種の実装不備に起因し、攻撃者が任意のユーザーとしてログイン可能な状態を引き起こしました。

```python
# 脆弱な実装例
settings = {
    "strict": False,
    "security": {
        "wantAssertionsSigned": False,
    }
}
```

#### XXE（XML外部エンティティ）インジェクション

SAMLはXMLを使用するため、XMLパーサーの設定次第でXXE攻撃の攻撃経路になります。パーサーが外部エンティティ参照を解決してしまうと、サーバー内のファイルや内部ネットワークへの到達が可能になります。

```xml
<!-- 攻撃者が送り込む悪意あるSAMLリクエスト -->
<?xml version="1.0"?>
<!DOCTYPE foo [
  <!ENTITY xxe SYSTEM "file:///etc/passwd">
]>
<samlp:AuthnRequest>
  <saml:Issuer>&xxe;</saml:Issuer>
</samlp:AuthnRequest>
```

## OAuth 2.0 の仕組みと脆弱性

### OAuth 2.0とは

OAuth 2.0 は認可（Authorization）のためのフレームワークです。「あるアプリが、ユーザーに代わって別サービスのリソースへアクセスする権限を委譲する」ための標準仕様であり、ユーザーはパスワードをクライアントアプリに渡さずに済みます。

### Authorization Code フロー（OAuth 2.0）

```mermaid
sequenceDiagram
    actor User as ユーザー
    participant App as クライアントアプリ
    participant AS as 認可サーバー
    participant RS as リソースサーバー（API）

    User->>App: ログイン / 連携ボタンをクリック
    App->>User: 認可エンドポイントへリダイレクト
    Note right of App: ?client_id=xxx<br/>&redirect_uri=https://app.example.com/callback<br/>&scope=read:email<br/>&state=RANDOM_VALUE<br/>&code_challenge=PKCE_HASH
    User->>AS: 認可リクエスト
    AS->>User: ログイン画面 + 権限承認画面
    User->>AS: 承認
    AS->>User: redirect_uri へリダイレクト + 認可コード（code）
    User->>App: code を転送
    App->>AS: code + client_secret でトークンリクエスト<br/>（バックエンド）
    AS->>App: access_token（+ refresh_token）
    App->>RS: access_token でAPIコール
    RS->>App: リソース（データ）を返す
```

### 主要パラメータ

| パラメータ       | 役割                                   | 攻撃対象リスク     |
| ---------------- | -------------------------------------- | ------------------ |
| `client_id`      | クライアントアプリの識別子（公開情報） | 低                 |
| `redirect_uri`   | 認可コードの送信先URL                  | 高                 |
| `scope`          | 要求する権限の範囲                     | 中                 |
| `state`          | CSRF対策用のランダム値                 | 高（省略は危険）   |
| `code`           | 1回限りの認可コード                    | 高                 |
| `code_challenge` | PKCEのハッシュ値                       | 防御パラメータ     |
| `client_secret`  | クライアントの秘密鍵                   | 要保護（漏洩禁止） |

### state パラメータ

#### stateがない場合の脅威：CSRF

`state` パラメータは、OAuthフローが「自分自身が開始したリクエストに対するコールバックである」ことを確認するためのものです。
攻撃シナリオは「アカウント紐付けの乗っ取り」です。多くのサービスでは「Googleアカウントと連携する」ような機能がありますが、`state` がないとこの機能が乗っ取られます。
被害者は自分のブラウザで意図せずコールバックURLを踏んでしまい、自分のアカウントに攻撃者のSNSアカウントを紐付けさせられてしまいます。攻撃者はその後、自分のSNSアカウントで被害者のアカウントに侵入できます。

```mermaid
sequenceDiagram
    actor Attacker as 攻撃者
    actor Victim as 被害者（ログイン済み）
    participant App as ターゲットアプリ
    participant AS as 認可サーバー（Google等）

    Note over Attacker: 攻撃者自身のアカウントで<br/>OAuthフローを開始
    Attacker->>AS: 認可リクエスト（攻撃者アカウントで承認）
    AS->>Attacker: /callback?code=ATTACKER_CODE を返す

    Note over Attacker: コールバックURLを<br/>コピーしてフローを中断<br/>（トークン交換を行わない）

    Note over Attacker: 被害者にそのURLを踏ませる<br/>（メールのimg src、フィッシング等）
    Attacker->>Victim: 細工したURLを送付

    Victim->>App: GET /callback?code=ATTACKER_CODE<br/>（被害者のセッションで実行）
    App->>App: state 未検証のままコードを処理
    App->>AS: ATTACKER_CODE でトークン交換
    AS->>App: 攻撃者のアクセストークンを発行

    Note over App: 被害者のアカウントに<br/>攻撃者のGoogleアカウントが紐付く

    Note over Attacker: 攻撃者は自分のGoogleアカウントで<br/>被害者のアカウントにログイン可能になる
```

#### stateによる防御の仕組み

`state` は「このフローは自分が開始した」ことを証明するためのワンタイムトークンです。`state` はセッションに紐付いているため、別のブラウザや別のセッションから同じ値を使い回すことができません。攻撃者はセッション内に保存された `state` を知る方法がないため、CSRF攻撃が成立しなくなります。

```mermaid
sequenceDiagram
    actor Attacker as 攻撃者
    actor Victim as 被害者（ログイン済み）
    participant App as ターゲットアプリ
    participant AS as 認可サーバー

    Note over Victim, App: 被害者が正規のOAuthフローを開始
    Victim->>App: 「Googleと連携する」をクリック
    App->>App: state（ランダム値）を生成セッションに保存
    App->>AS: 認可リクエスト（?state=***を付与）

    Note over Attacker: 攻撃者は state の値を知らない
    Attacker->>Victim: 細工したコールバックURLを踏ませる
    Victim->>App: GET /callback?code=ATTACKER_CODE&state=WRONG

    App->>App: セッションの stateとリクエストの stateを照合
    App->>App: 不一致 → リクエストを拒否

    Note over App: 攻撃失敗
```

### code_challengeパラメータ（PKCE）

#### PKCEがない場合の脅威：認可コードの漏洩とトークン交換

PKCE（Proof Key for Code Exchange、RFC 7636）を理解するには、まず「SPAが `client_secret` を持てない」という前提を押さえる必要があります。
サーバーサイドアプリと異なり、SPAはすべてのコードがブラウザ上で動作します。仮に `client_secret` をJSバンドルに含めても、DevToolsを開けば誰でも確認できるため、秘密として機能しません。そのため、SPAは `client_secret` なしで認可コードをトークンに交換しなければならず、「認可コードさえ手に入れれば誰でもトークンを取得できる」状態になってしまいます。
認可コードはどのように漏洩するのか。代表的な経路の一つが、悪意あるブラウザ拡張機能によるコールバックURLの盗み見です。
認可コードの漏洩経路はブラウザ拡張機能に限らず、`Referer` ヘッダーへの混入やオープンリダイレクターを経由した流出など複数存在します。いずれの場合も、`client_secret` なしでトークン交換できてしまうSPAは無防備です。PKCEはこの問題を解決します。

```mermaid
sequenceDiagram
    actor Victim as 被害者
    participant Browser as ブラウザ
    participant Extension as 悪意ある拡張機能
    participant App as SPAアプリ
    participant AS as 認可サーバー

    Victim->>Browser: ログインボタンをクリック
    Browser->>AS: 認可リクエスト
    AS->>Browser: ログイン画面を表示
    Victim->>AS: 認証・承認

    AS->>Browser: https://app.example.com/callback?code=AUTH_CODE へリダイレクト
    Note over Extension: ブラウザ拡張機能はURLの変化を監視できる
    Extension->>Extension: コールバックURLから code=AUTH_CODE を抽出

    Extension->>AS: code=AUTH_CODE でトークンリクエスト
    Note right of Extension: client_secret は不要（SPAは公開クライアントのため）
    AS->>Extension: access_token を発行

    Browser->>App: 同じ code=AUTH_CODE を転送
    App->>AS: code=AUTH_CODE でトークンリクエスト（正規）
    AS->>App: access_token を発行（コードが使い回せる場合）

    Note over Extension: 被害者のアクセストークンを入手
```

#### PKCEによる防御の仕組み

PKCEは「認可コードを要求したセッションだけがトークンに交換できる」という仕組みです。事前にランダムな秘密値のハッシュを認可サーバーに登録しておき、トークン交換時に秘密値の原文を提示させることで本人確認を行います。

```mermaid
sequenceDiagram
    participant App as SPAアプリ
    participant Extension as 悪意ある拡張機能
    participant AS as 認可サーバー

    Note over App: code_verifier を生成メモリ上にのみ保持
    Note over App: code_challenge = BASE64URL(SHA-256(code_verifier))

    App->>AS: 認可リクエスト（code_challenge + code_challenge_method=S256 を付与）
    AS->>AS: code_challenge をメモして認可コードを発行
    AS->>App: https://app.example.com/callback?code=AUTH_CODE

    Note over Extension: コールバックURLを監視して code を抽出
    Extension->>AS: code=AUTH_CODE でトークンリクエスト（code_verifier を知らないので送れない）
    AS->>AS: code_verifier なし → 拒否
    AS-->>Extension: 400 Bad Request

    Note over AS: 攻撃失敗：code_verifier を持つ正規セッションだけがトークンを取得できる

    App->>AS: code=AUTH_CODE + code_verifier でトークンリクエスト
    AS->>AS: BASE64URL(SHA-256(code_verifier)) と保存済み code_challenge を照合
    AS->>App: access_token を発行
```

**パラメータの対応関係**

| パラメータ              | タイミング       | 内容                                                                                          |
| ----------------------- | ---------------- | --------------------------------------------------------------------------------------------- |
| `code_verifier`         | アプリ内で保持   | ランダムな文字列<br/>認可リクエスト時には送信せず、トークン交換時にのみ認可サーバーへ送信する |
| `code_challenge`        | 認可リクエスト時 | `BASE64URL(SHA-256(code_verifier))` のハッシュ値                                              |
| `code_challenge_method` | 認可リクエスト時 | `S256`（SHA-256）を指定する。`plain` は非推奨                                                 |

`code_verifier` はSPAのメモリ上にのみ保持されます。認可サーバーにはそのハッシュ値（`code_challenge`）だけを事前に登録し、トークン交換時に`code_verifier`を提示することで「このコードを要求した本人である」と証明します。拡張機能などの第三者は `code_verifier` を知る手段がないため、認可コードを入手してもトークンに交換できません。
SPAに限らず、Authorization Code フロー全般で PKCE の使用が推奨されています。

### 典型的な脆弱性

#### redirect_uri の検証不足

認可コードは `redirect_uri` に送信されます。検証が甘いと、攻撃者が自分のサーバーにコードを盗み取ることができます。

```mermaid
sequenceDiagram
    actor Attacker as 攻撃者
    actor Victim as 被害者
    participant AS as 認可サーバー
    participant Evil as 攻撃者のサーバー

    Attacker->>Attacker: 細工したURLを作成
    Note right of Attacker: redirect_uri=https://app.example.com.evil.com/cb<br/>（前方一致チェックを通過）
    Attacker->>Victim: 細工したURLをフィッシング等で踏ませる
    Victim->>AS: 認可リクエスト（偽のredirect_uri）
    AS->>AS: 前方一致で検証 → 通過
    AS->>Evil: 認可コード（code）を evil.com へ送信
    Evil->>AS: code + client_secret でトークン取得
    Note over Evil: 被害者のアクセストークンを取得・アカウント侵害
```

```
# 前方一致の例
https://app.example.com           → 許可
https://app.example.com.evil.com  → 通過してしまう

# 正規表現（エスケープ漏れ）の例
^https://app\.example\.com.*      → https://app_example_com.evil.com も通過

# 正しい検証
登録済みURIとの完全一致のみ許可
```

#### 認可コードの使い回し

認可コードは1回限り使用可能で、かつ短命（数分以内）であるべきです。

```http
# 1回目のリクエスト：正常にトークン取得
POST /token
code=abc123&client_id=xxx&client_secret=yyy

HTTP/1.1 200 OK
{"access_token": "eyJ..."}

# 脆弱な実装では同じコードを再利用できてしまう
POST /token
code=abc123&client_id=xxx&client_secret=yyy

HTTP/1.1 200 OK
{"access_token": "eyJ..."}
```

コードはトークン発行時に即座に無効化し、再利用を試みた場合はそのコードに関連するトークンをすべて失効させるのがベストプラクティスです。

#### Implicit フローの危険性

`response_type=token` を使う Implicit フローは、アクセストークンがURLフラグメントに直接露出します。

```
# Implicitフローのリダイレクト例
https://app.example.com/callback#access_token=eyJhbGci...&token_type=bearer
                                 ^
                                 ブラウザ履歴・Refererヘッダ・JS から読み取り可能
```

Implicit フローは OAuth 2.1 草案で非推奨（SHOULD NOT）となっています。現在の推奨は Authorization Code + PKCE です。

## OIDC（OpenID Connect）の仕組みと脆弱性

### OIDCとは

OIDC（OpenID Connect）は、OAuth 2.0 を認可の基盤として、その上に認証レイヤーを積み重ねた仕様です。OAuth 2.0 は本来「認可」のためのフレームワークであり、「このユーザーが誰か」を標準的に確認する手段がありませんでした。OIDCはその問題を解決します。
OIDCを使うと、OAuth 2.0のフローに加えて `id_token`（JWT形式）が返されます。これがユーザー本人であることの証明書になります。

```mermaid
flowchart TB
    subgraph OIDC["OIDC（OpenID Connect）"]
        direction TB
        A["id_token（JWT）<br/>nonce<br/>/userinfo"]
        subgraph OAuth2["OAuth 2.0"]
        end
    end
```

### id_token の構造

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9        <- ヘッダー（Base64URL）
.
eyJzdWIiOiIxMjM0NTYiLCJpc3MiOiJodHRwczovL2...  <- ペイロード（Base64URL）
.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c  <- 署名
```

デコードしたペイロード：

```json
{
  "iss": "https://accounts.google.com",
  "sub": "123456789",
  "aud": "my-app-client-id",
  "exp": 1700003600,
  "iat": 1700000000,
  "nonce": "abc123xyz",
  "email": "user@example.com"
}
```

### Authorization Code フロー（OIDC）

```mermaid
sequenceDiagram
    participant App as クライアントアプリ
    participant OP as OpenID Provider

    Note over App, OP: 認可リクエスト時（OAuth 2.0との差分）
    App->>OP: scope に openid を追加<br/>nonce=RANDOM_VALUE を追加

    Note over App, OP: トークンレスポンス時（OAuth 2.0との差分）
    OP->>App: access_token に加えて id_token（JWT）が返される

    Note over App, OP: id_token の検証（OIDCで追加されるステップ）
    App->>App: 署名とiss / aud / exp / nonce を検証
```

### 典型的な脆弱性

#### alg:none 攻撃（JWT署名検証の欠落）

JWTのヘッダーには署名アルゴリズムが指定されています。一部の古いライブラリでは `"alg": "none"` を受け付けてしまい、署名なしのJWTを有効として扱う脆弱性があります。

```
# 正規のid_token（RS256署名あり）
eyJhbGciOiJSUzI1NiJ9.eyJzdWIiOiJ1c2VyMTIzIiwuLi59.VALID_SIGNATURE

# 攻撃者が作った改ざんid_token（alg:none、署名なし）
eyJhbGciOiJub25lIn0.eyJzdWIiOiJhZG1pbiIsImV4cCI6OTk5OTk5OTk5OX0.
                              ^^^^^^^^^^^^^
                              sub を "admin" に書き換え済み
                              末尾の署名部分は空文字列
```

```python
# 脆弱な実装例：アルゴリズムを指定していない
decoded = jwt.decode(id_token, public_key)

# 安全な実装例：許可アルゴリズムを明示（"none" は絶対に含めない）
decoded = jwt.decode(
    id_token,
    public_key,
    algorithms=["RS256"],
    audience="my-app-client-id",
)
```

#### nonce の未検証（リプレイ攻撃）

`nonce` は同じid_tokenを使い回すリプレイ攻撃を防ぐためのものです。検証を省略すると、一度傍受したid_tokenを再利用できます。

```mermaid
sequenceDiagram
    actor Attacker as 攻撃者
    actor Victim as 被害者
    participant App as アプリ

    Attacker->>Attacker: 通信傍受等で被害者のid_tokenを入手
    Attacker->>App: 盗んだid_tokenでログインリクエスト
    App->>App: 署名検証（本物なので通過）
    App->>App: nonce 未検証 → ログイン成功
    Note over App: 被害者アカウントに侵入
```

id_token 受信時に検証すべきクレームは以下の通りです。

| クレーム | 検証内容                               |
| -------- | -------------------------------------- |
| `iss`    | 期待するIdPのURL と一致するか          |
| `aud`    | 自アプリの `client_id` と一致するか    |
| `exp`    | 現在時刻より未来か（期限切れでないか） |
| `nonce`  | セッションに保存した値と一致するか     |

#### Dynamic Client Registration の悪用（SSRF）

OIDCプロバイダーが認証なしで動的クライアント登録を許可している場合、`logo_uri` や `jwks_uri` などのURIフィールドに内部アドレスを指定することでSSRFにつながります。
例では認可サーバーがクライアントのロゴ画像を取得しようとすると、クラウドのメタデータAPIにアクセスしてしまいIAMクレデンシャルが漏洩します。

```http
POST /register HTTP/1.1
Host: auth.example.com
Content-Type: application/json

{
  "redirect_uris": ["https://attacker.com/callback"],
  "logo_uri": "http://169.254.169.254/latest/meta-data/iam/security-credentials/admin"
}
```

```http
HTTP/1.1 201 Created
{"client_id": "new-client-xxx"}
```

## 各方式の比較

### 概要比較

| 観点                 | SAML 2.0                          | OAuth 2.0            | OIDC                   |
| -------------------- | --------------------------------- | -------------------- | ---------------------- |
| **主な目的**         | 認証・SSO                         | 認可                 | 認証・認可             |
| **データ形式**       | XML                               | JSON / Opaque        | JWT（id_token）        |
| **主な利用場面**     | 社内システム<br/>エンタープライズ | APIアクセス委譲      | ソーシャルログイン     |
| **ユーザー情報**     | SAMLアサーション内                | 規定なし（実装依存） | id_token<br/>/userinfo |
| **モバイル対応**     | △（XMLが重い）                    | ◎                    | ◎                      |
| **仕様の複雑さ**     | 高                                | 中                   | 中                     |
| **新規開発での採用** | 既存環境との互換性優先            | API認可用途          | 推奨                   |

## まとめ

**SAMLチェックリスト**

- [x] `strict: true` を必ず有効化し署名検証をスキップしない
- [x] 署名アルゴリズムは `RSA-SHA256` 以上を強制する
- [x] 署名検証したノードのみを後続処理で使用する（XSW対策）
- [x] XMLパーサーでDTD処理・外部エンティティを無効化する（XXE対策）
- [x] SAMLライブラリは定期的に最新版へ更新する（CVE追跡）
- [x] `NotBefore` / `NotOnOrAfter` の時刻検証を実施する
- [x] IdPから受け取る `Issuer` を検証する

**OAuth 2.0チェックリスト**

- [x] `redirect_uri` は登録済みリストとの**完全一致**で検証する
- [x] `state` パラメータを毎回生成し、コールバック時に検証する
- [x] 認可コードはトークン使用後に即座に無効化する
- [x] 認可コードの有効期限は10分以内にする
- [x] `client_secret` はバックエンドのみで保持し、フロントエンドに露出させない
- [x] Implicit フローを使用しない（Authorization Code + PKCE へ移行する）
- [x] スコープは最小権限の原則で設計する

**OIDCチェックリスト**

- [x] id_token の署名アルゴリズムを明示的に指定し、`"none"` を拒否する
- [x] `iss`（Issuer）が期待するIdPと一致するか検証する
- [x] `aud`（Audience）が自アプリの `client_id` と一致するか検証する
- [x] `exp`（有効期限）を検証する
- [x] `nonce` を生成し、id_token内の値と照合する（リプレイ対策）
- [x] Dynamic Client Registration には認証を必須とするか、無効化する
- [x] `logo_uri` / `jwks_uri` 等のURIフィールドにSSRF対策を施す

**参考：**

- [RFC 6749 - The OAuth 2.0 Authorization Framework](https://www.rfc-editor.org/rfc/rfc6749)
- [RFC 7636 - PKCE](https://www.rfc-editor.org/rfc/rfc7636)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)

:::message
セキュリティは攻撃と防御の両面を理解することで向上します。本記事で学んだ知識を、より安全なシステム構築に活かしてください。
:::
