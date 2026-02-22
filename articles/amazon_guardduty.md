---
title: "Amazon GuardDutyによるアップロードファイルチェック"
emoji: "🦠"
type: "tech"
topics:
  - "web"
  - "security"
  - "malware"
published: true
published_at: "2026-01-05 21:59"
---

## 機能概要

### 課題

ファイル共有サービスでは、ユーザーがアップロードしたファイルにマルウェアが含まれるリスクがあります。感染ファイルが他のユーザーにダウンロードされると、被害が拡大する恐れがあります。

### 解決

Amazon GuardDuty Malware Protection for S3 を活用し、アップロードされたファイルを自動スキャンします。スキャン結果に応じてファイルへのアクセスを制御することで、安全なファイル共有を実現します。

```mermaid
flowchart LR
    A[ファイル<br>アップロード] --> B{マルウェア<br>スキャン}
    B -->|クリーン| C[ダウンロード許可]
    B -->|感染| D[ファイル削除<br>アクセス拒否]
    B -->|スキャン中| E[ダウンロード保留]

    style C fill:#22c55e,color:#000000
    style D fill:#ef4444,color:#000000
    style E fill:#f59e0b,color:#000000
```

### 主な機能

| 機能             | 説明                                                 |
| ---------------- | ---------------------------------------------------- |
| 自動スキャン     | S3へのアップロード時に自動でマルウェアスキャンを実行 |
| 感染ファイル削除 | マルウェア検出時は即座にS3から削除し、アクセスを拒否 |
| スキャン中の保護 | スキャン完了までダウンロードを保留（202レスポンス）  |
| 状態管理         | DynamoDBでスキャン状態を管理し、ダウンロード時に参照 |

## アーキテクチャ

```mermaid
graph TB
    subgraph "ユーザー操作"
        U[ユーザー]
    end

    subgraph "ストレージ層"
        S3F[(S3ファイル)]
        DDB[(DynamoDB)]
    end

    subgraph "マルウェアスキャン層"
        GD[GuardDuty<br>Malware Protection]
        EB[EventBridge]
        L5[ScanResult<br>Lambda]
    end

    subgraph "API層"
        L2[Download<br>Lambda]
    end

    U -->|アップロード| S3F
    S3F -->|自動スキャン| GD
    GD -->|結果通知| EB
    EB --> L5
    L5 -->|スキャン結果保存| DDB
    L5 -->|感染時削除| S3F

    U -->|ダウンロード要求| L2
    L2 -->|スキャン状態確認| DDB
    L2 -->|署名付きURL発行| S3F

    style GD fill:#00a4ca,color:#000000
    style EB fill:#ff9900,color:#000000
    style L5 fill:#ff9900,color:#000000
    style L2 fill:#ff9900,color:#000000
    style S3F fill:#569a31,color:#000000
    style DDB fill:#4b53bc,color:#000000
```

## スキャン結果とアプリケーション動作

| scanResultStatus   | アプリケーション動作       | HTTPステータス        |
| ------------------ | -------------------------- | --------------------- |
| `NO_THREATS_FOUND` | ダウンロード許可           | 200                   |
| `THREATS_FOUND`    | ファイル削除・アクセス拒否 | 403 (`ACCESS_DENIED`) |
| `PENDING`          | ダウンロード保留           | 202 (`SCAN_PENDING`)  |
| `UNSUPPORTED`      | ダウンロード許可           | 200                   |

## データフロー

```mermaid
sequenceDiagram
    participant User
    participant S3
    participant GuardDuty
    participant EventBridge
    participant ScanResultLambda
    participant DynamoDB
    participant DownloadLambda

    User->>S3: ファイルアップロード
    S3->>GuardDuty: 新規オブジェクト通知
    GuardDuty->>GuardDuty: マルウェアスキャン
    GuardDuty->>EventBridge: スキャン結果イベント

    alt 脅威検出
        EventBridge->>ScanResultLambda: THREATS_FOUND
        ScanResultLambda->>S3: ファイル削除
        ScanResultLambda->>DynamoDB: scanStatus = THREATS_FOUND
    else クリーン
        EventBridge->>ScanResultLambda: NO_THREATS_FOUND
        ScanResultLambda->>DynamoDB: scanStatus = NO_THREATS_FOUND
    end

    User->>DownloadLambda: ダウンロード要求
    DownloadLambda->>DynamoDB: スキャン状態確認

    alt クリーン
        DownloadLambda->>User: 署名付きURL (200)
    else スキャン中
        DownloadLambda->>User: 202 SCAN_PENDING
    else 感染
        DownloadLambda->>User: 403 ACCESS_DENIED
    end
```

## EventBridgeルールの例

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Malware Protection Object Scan Result"],
  "detail": {
    "s3ObjectDetails": { "bucketName": ["<バケット名>"] }
  }
}
```

**参考：**

### Amazon GuardDuty導入済みファイル共有アプリ

https://github.com/hanacus87/file-sharing

- [GuardDuty Malware Protection for S3](https://docs.aws.amazon.com/guardduty/latest/ug/gdu-malware-protection-s3.html)
- [Monitoring S3 object scans with Amazon EventBridge](https://docs.aws.amazon.com/guardduty/latest/ug/monitor-with-eventbridge-s3-malware-protection.html)
- [Using Amazon GuardDuty Malware Protection to scan uploads to Amazon S3 | AWS Security Blog](https://aws.amazon.com/blogs/security/using-amazon-guardduty-malware-protection-to-scan-uploads-to-amazon-s3/)
