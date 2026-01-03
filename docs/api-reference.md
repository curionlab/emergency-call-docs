# API リファレンス

このドキュメントでは、Emergency Call Server が提供するREST APIの詳細仕様を説明します。

**Base URL（ローカル）**: `http://localhost:3000`

---

## 認証

### POST /login
発信者（Sender）のログイン

#### リクエスト
```http
POST /login
Content-Type: application/json

{
  "password": "your_password"
}
```

#### レスポンス（成功）
```json
{
  "success": true,
  "token": "session_token_string"
}
```

#### レスポンス（失敗）
```json
{
  "success": false,
  "error": "パスワードが正しくありません"
}
```

#### ステータスコード
- `200 OK` - ログイン成功
- `401 Unauthorized` - パスワード不一致

---

## 受信者管理

### POST /register
受信者（Receiver）の登録

#### リクエスト
```http
POST /register
Content-Type: application/json

{
  "receiverId": "user123",
  "authCode": "582937",
  "subscription": {
    "endpoint": "https://fcm.googleapis.com/fcm/send/...",
    "keys": {
      "p256dh": "...",
      "auth": "..."
    }
  }
}
```

#### レスポンス（成功）
```json
{
  "success": true,
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### レスポンス（失敗）
```json
{
  "success": false,
  "error": "認証コードが無効です"
}
```

#### ステータスコード
- `200 OK` - 登録成功
- `400 Bad Request` - 必須パラメータ不足
- `401 Unauthorized` - 認証コード無効

---

### POST /update-subscription
Push購読情報の更新

#### リクエスト
```http
POST /update-subscription
Content-Type: application/json
Authorization: Bearer <accessToken>

{
  "receiverId": "user123",
  "subscription": {
    "endpoint": "https://fcm.googleapis.com/fcm/send/...",
    "keys": {
      "p256dh": "...",
      "auth": "..."
    }
  }
}
```

#### レスポンス（成功）
```json
{
  "success": true
}
```

#### ステータスコード
- `200 OK` - 更新成功
- `401 Unauthorized` - トークン無効
- `404 Not Found` - 受信者未登録

---

## 認証コード管理

### POST /generate-auth-code
受信者登録用の認証コードを生成

#### リクエスト
```http
POST /generate-auth-code
Content-Type: application/json

{
  "receiverId": "user123"
}
```

#### レスポンス（成功）
```json
{
  "code": "582937"
}
```

#### 注意事項
- コードは6桁の数字
- 有効期限：30分
- 同一receiverIdで再発行すると上書き

#### ステータスコード
- `200 OK` - 生成成功
- `400 Bad Request` - receiverId不足

---

## 通知送信

### POST /send-notification
緊急通知を送信

#### リクエスト
```http
POST /send-notification
Content-Type: application/json
Authorization: Bearer <sessionToken>

{
  "receiverId": "user123",
  "sessionId": "emergency-1704297600000",
  "senderId": "sender-1704297600123"
}
```

#### レスポンス（成功）
```json
{
  "success": true
}
```

#### レスポンス（失敗）
```json
{
  "success": false,
  "error": "受信者が見つかりません"
}
```

#### Push Payload（受信者に送られる内容）
```json
{
  "title": "緊急通知",
  "body": "緊急コールを受信しました",
  "sessionId": "emergency-1704297600000",
  "senderId": "sender-1704297600123",
  "url": "http://localhost:5173"
}
```

#### ステータスコード
- `200 OK` - 送信成功
- `401 Unauthorized` - sessionToken無効
- `404 Not Found` - 受信者未登録
- `410 Gone` - Push購読が期限切れ

---

## システム情報

### GET /vapid-public-key
VAPID公開鍵を取得

#### リクエスト
```http
GET /vapid-public-key
```

#### レスポンス
```json
{
  "publicKey": "BNxS3M7zvz..."
}
```

#### ステータスコード
- `200 OK` - 取得成功

---

### GET /health
サーバーのヘルスチェック

#### リクエスト
```http
GET /health
```

#### レスポンス
```json
{
  "status": "ok"
}
```

#### ステータスコード
- `200 OK` - サーバー正常

---

## トークン更新

### POST /refresh-token
アクセストークンを更新

#### リクエスト
```http
POST /refresh-token
Content-Type: application/json

{
  "refreshToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### レスポンス（成功）
```json
{
  "accessToken": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
}
```

#### ステータスコード
- `200 OK` - 更新成功
- `401 Unauthorized` - refreshToken無効

---

## ICE設定取得（将来実装予定）

### GET /ice-config
WebRTC用のICEサーバー設定を取得

#### リクエスト
```http
GET /ice-config
X-API-Key: <api_key>
```

#### レスポンス（予定）
```json
{
  "iceServers": [
    {
      "urls": "stun:stun.l.google.com:19302"
    },
    {
      "urls": "turn:your-turn-server.com:3478",
      "username": "temp_user_123",
      "credential": "temp_pass_456"
    }
  ]
}
```

**注意**: 現在未実装。実装時にTURN短期資格を配布予定。

---

## エラーレスポンス共通フォーマット

### 一般的なエラー
```json
{
  "success": false,
  "error": "エラーメッセージ"
}
```

### バリデーションエラー
```json
{
  "success": false,
  "error": "必須パラメータが不足しています",
  "details": {
    "missing": ["receiverId", "authCode"]
  }
}
```

---

## レート制限

現在、レート制限は実装されていません。  
本番環境では、express-rate-limit 等の導入を推奨します。

---

## 次のステップ

- [アーキテクチャ](architecture.md) - APIがどのように動作するか
- [トラブルシューティング](troubleshooting.md) - API呼び出しの問題解決
