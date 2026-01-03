# アーキテクチャ

このドキュメントでは、緊急コールシステムの全体アーキテクチャ、技術スタック、データフローを説明します。

---

## システム構成図

```
┌─────────────────────────────────────────────────────────────┐
│                         Internet                             │
│                                                               │
│  ┌──────────────┐         ┌──────────────┐                  │
│  │   Browser    │         │   Browser    │                  │
│  │  (Sender)    │         │  (Receiver)  │                  │
│  │              │         │              │                  │
│  │ ┌──────────┐ │         │ ┌──────────┐ │                  │
│  │ │index.html│ │         │ │index.html│ │                  │
│  │ │  (PWA)   │ │         │ │  (PWA)   │ │                  │
│  │ └──────────┘ │         │ └──────────┘ │                  │
│  │      ↓       │         │      ↑       │                  │
│  │ ┌──────────┐ │         │ ┌──────────┐ │                  │
│  │ │  PeerJS  │ │←─WebRTC─→│  PeerJS  │ │                  │
│  │ └──────────┘ │         │ └──────────┘ │                  │
│  └──────┬───────┘         └──────┬───────┘                  │
│         │                        │                           │
│         │ HTTP API               │ Push Subscription         │
│         ↓                        ↓ + Push Receive           │
│  ┌─────────────────────────────────────────┐                │
│  │     Emergency Call Server (Node.js)     │                │
│  │  ┌─────────────────────────────────┐    │                │
│  │  │  Express.js REST API            │    │                │
│  │  │  - /login                       │    │                │
│  │  │  - /register                    │    │                │
│  │  │  - /send-notification           │    │                │
│  │  │  - /generate-auth-code          │    │                │
│  │  │  - /update-subscription         │    │                │
│  │  └─────────────────────────────────┘    │                │
│  │  ┌─────────────────────────────────┐    │                │
│  │  │  Web Push (web-push library)    │    │                │
│  │  └─────────────────────────────────┘    │                │
│  │  ┌─────────────────────────────────┐    │                │
│  │  │  Data Storage (data.json)       │    │                │
│  │  │  - Receivers                    │    │                │
│  │  │  - Auth Codes                   │    │                │
│  │  │  - Tokens                       │    │                │
│  │  └─────────────────────────────────┘    │                │
│  └─────────────────────────────────────────┘                │
│                        ↓                                     │
│  ┌─────────────────────────────────────────┐                │
│  │   Browser Push Service                  │                │
│  │   (Google FCM, Apple APNs, etc.)        │                │
│  └─────────────────────────────────────────┘                │
└─────────────────────────────────────────────────────────────┘
```

---

## 技術スタック

### Server（Backend）
- **Node.js** - JavaScript ランタイム
- **Express.js** - Webフレームワーク
- **web-push** - Web Push 通知ライブラリ（VAPID対応）
- **jsonwebtoken** - JWT トークン管理
- **cors** - CORS処理

### Client（Frontend）
- **HTML/CSS/JavaScript** - ネイティブWeb技術
- **Service Worker** - バックグラウンド通知処理
- **PeerJS** - WebRTC シグナリング
- **Web Push API** - Push通知受信
- **PWA（Progressive Web App）** - インストール可能なWebアプリ

### 通信プロトコル
- **HTTP/HTTPS** - REST API通信
- **WebRTC** - P2P音声/ビデオ通信（データチャネル含む）
- **Web Push Protocol (RFC 8030)** - Push通知配送
- **VAPID (RFC 8292)** - 送信者認証

---

## データフロー

### 1. Receiver登録フロー

```
┌─────────┐                    ┌─────────┐
│ Sender  │                    │ Server  │
└────┬────┘                    └────┬────┘
     │                              │
     │ POST /generate-auth-code     │
     │ (receiverId)                 │
     │─────────────────────────────→│
     │                              │ ◆ 認証コード生成（6桁）
     │ authCode (582937)            │ ◆ 30分有効期限設定
     │←─────────────────────────────│
     │                              │
     
┌─────────┐                    ┌─────────┐
│Receiver │                    │ Server  │
└────┬────┘                    └────┬────┘
     │                              │
     │ 1. 通知許可                   │
     │ 2. Push購読作成               │
     │                              │
     │ POST /register               │
     │ (receiverId, authCode,       │
     │  subscription)               │
     │─────────────────────────────→│
     │                              │ ◆ authCode検証
     │                              │ ◆ subscription保存
     │                              │ ◆ トークン発行
     │ accessToken, refreshToken    │
     │←─────────────────────────────│
     │                              │
     │ localStorage保存             │
     │                              │
```

### 2. 緊急通知送信フロー

```
┌─────────┐      ┌─────────┐      ┌──────────┐      ┌─────────┐
│ Sender  │      │ Server  │      │ Push     │      │Receiver │
│         │      │         │      │ Service  │      │         │
└────┬────┘      └────┬────┘      └────┬─────┘      └────┬────┘
     │                │                 │                 │
     │ POST /send-notification          │                 │
     │ (receiverId, sessionId, senderId)│                 │
     │───────────────→│                 │                 │
     │                │ ◆ subscription取得               │
     │                │ ◆ payload作成                    │
     │                │   - title/body                   │
     │                │   - sessionId                    │
     │                │   - senderId                     │
     │                │   - url (CLIENT_URL)             │
     │                │                                  │
     │                │ Web Push送信                     │
     │                │ (VAPID署名付き)                  │
     │                │────────────────→│                 │
     │                │                 │ Push配送        │
     │                │                 │────────────────→│
     │                │                 │                 │
     │                │                 │    ◆ sw.js: push event
     │                │                 │    ◆ 通知表示
     │ success        │                 │                 │
     │←───────────────│                 │                 │
     │                │                 │                 │
```

### 3. 通話開始フロー

```
┌─────────┐                              ┌─────────┐
│Receiver │                              │ Sender  │
└────┬────┘                              └────┬────┘
     │                                        │
     │ ◆ 通知タップ                            │
     │ ◆ sw.js: notificationclick            │
     │ ◆ URL open:                           │
     │   ?autoAnswer=true                    │
     │   &sessionId=xxx                      │
     │   &senderId=yyy                       │
     │                                        │
     │ ◆ index.html読み込み                   │
     │ ◆ autoAnswerCall() 実行                │
     │                                        │
     │ PeerJS接続                             │
     │ (receiverId → senderId)                │
     │───────────────────────────────────────→│
     │                                        │
     │ ◆ WebRTC Offer/Answer交換             │
     │←──────────────────────────────────────→│
     │                                        │
     │ ◆ ICE Candidate交換                    │
     │←──────────────────────────────────────→│
     │                                        │
     │ ◆ P2P通話確立                           │
     │════════════════════════════════════════│
     │     音声/ビデオストリーム               │
     │════════════════════════════════════════│
```

---

## セキュリティモデル

### 認証フロー

1. **発信者認証**：パスワードベース（`/login`）→ sessionToken発行
2. **受信者認証**：認証コード（30分有効）→ JWT（accessToken: 15分、refreshToken: 30日）

### データ保護

- **VAPID秘密鍵**：サーバー側のみ保持（環境変数）
- **トークン**：クライアント側は localStorage（XSS対策が必要）
- **認証コード**：30分で自動削除
- **HTTPS**：本番環境では必須

### CORS設定

Server側で `CLIENT_URL` をホワイトリスト登録し、クロスオリジンアクセスを制御。

---

## スケーラビリティ考察

### 現在の制約
- **ファイルベースストレージ**（`data.json`）：単一サーバー前提
- **インメモリセッション**：サーバー再起動で揮発

### 将来の拡張案
- **データベース導入**：PostgreSQL, MongoDB など
- **Redis**：セッション管理、トークンブラックリスト
- **マルチサーバー構成**：ロードバランサー + セッション共有
- **TURN サーバー**：NAT越え対応の強化

---

## WebRTC P2P通信

### シグナリング
PeerJS（0.peerjs.com の公開シグナリングサーバーを使用）

### ICE サーバー
- **現在**：STUN のみ（Google Public STUN: `stun:stun.l.google.com:19302`）
- **将来**：TURN サーバー追加（`/ice-config` から短期資格取得）

### NAT越えとIPv6の重要性

#### キャリアグレードNAT（CGNAT）の問題
モバイル回線のIPv4接続では、キャリアがCGNATを使用して複数ユーザーに同じグローバルIPを割り当てます。この環境では：

- **対称型NAT** が使われることが多い
- **STUNだけでは接続できない**（TURNサーバーが必要）
- 特にY!mobile、楽天モバイル等で顕著

#### IPv6による解決
IPv6接続では：

- 各デバイスが**グローバルIPv6アドレス**を持つ
- NATを経由せず直接P2P接続が可能
- STUNのみで接続成功率が大幅に向上

#### 確認方法
ユーザーに https://test-ipv6.com/ でのIPv6接続確認を推奨。

#### 将来の対応
IPv4+CGNAT環境でも接続できるよう、TURNサーバーを導入予定。

### メディアストリーム
- **優先**：ビデオ + 音声
- **フォールバック**：音声のみ
- **最悪**：接続のみ（メディアなし）


---

## 次のステップ

- [API リファレンス](api-reference.md) - 各エンドポイントの詳細仕様
- [ユーザーマニュアル](user-manual.md) - 実際の操作方法
