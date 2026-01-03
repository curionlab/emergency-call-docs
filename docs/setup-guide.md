# セットアップガイド

このガイドでは、緊急コールシステムをローカル環境および本番環境でセットアップする手順を説明します。

---

## 前提条件

### 必須
- Node.js 16.x 以上
- npm または yarn

### 推奨
- Git
- 対応ブラウザ（Chrome, Edge, Safari 16.4+）

---

## ローカル開発環境

### 1. Server のセットアップ

#### リポジトリのクローン
```bash
git clone https://github.com/curionlab/emergency-call-server.git
cd emergency-call-server
```

#### 依存関係のインストール
```bash
npm install
```

#### 環境変数の設定
`.env.example` を `.env` にコピーして編集：

```bash
cp .env.example .env
```

`.env` の内容例：
```env
PORT=3000
CLIENT_URL=http://localhost:5173

# VAPID鍵ペア（生成方法は後述）
VAPID_PUBLIC_KEY=your_public_key
VAPID_PRIVATE_KEY=your_private_key

# セッション用秘密鍵
SESSION_SECRET=your_random_secret_string

# デフォルトパスワード（発信者ログイン用）
DEFAULT_PASSWORD=your_secure_password
```

#### VAPID鍵の生成
```bash
npx web-push generate-vapid-keys
```

出力された公開鍵・秘密鍵を `.env` に設定してください。

#### サーバー起動
```bash
npm start
```

サーバーは `http://localhost:3000` で起動します。

---

### 2. Client のセットアップ

#### リポジトリのクローン
```bash
git clone https://github.com/curionlab/emergency-call-client.git
cd emergency-call-client
```

#### 静的ファイルの配信
```bash
# ポート5173で起動（serverのCLIENT_URLと一致させる）
npx serve -l 5173
```

または：
```bash
python3 -m http.server 5173
```

クライアントは `http://localhost:5173` で起動します。

---

### 3. 動作確認

#### Receiver（受信者）として登録
1. ブラウザで `http://localhost:5173` を開く
2. 「Receiver」を選択
3. 通知を許可
4. Server側で認証コードを発行（発信者画面の「認証コード発行」ツールを使用）
5. receiverId と authCode を入力して登録

#### Sender（発信者）として通知送信
1. 別のブラウザタブで `http://localhost:5173` を開く
2. 「Sender」を選択
3. パスワードでログイン（`.env` の `DEFAULT_PASSWORD`）
4. receiverId を入力
5. 「緊急コール」ボタンをクリック

#### 通知受信と通話
Receiver側で通知が届き、タップすると自動的に通話画面が開きます。

---

## 本番環境（インターネット公開）

### Server のデプロイ

#### 推奨プラットフォーム
- Render.com（無料プランあり）
- Railway
- Heroku
- VPS（Ubuntu + nginx）

#### 環境変数の設定（本番環境用）
```env
PORT=3000
CLIENT_URL=https://your-client-domain.com

VAPID_PUBLIC_KEY=...
VAPID_PRIVATE_KEY=...
SESSION_SECRET=...
DEFAULT_PASSWORD=...
```

#### HTTPSの確保
本番環境では必ずHTTPSを使用してください（Web Pushの要件）。

---

### Client のデプロイ

#### 推奨プラットフォーム
- Cloudflare Pages（無料、高速）
- Netlify
- GitHub Pages
- Vercel

#### デプロイ手順（Cloudflare Pages の例）
1. GitHubリポジトリを接続
2. ビルド設定：
   - ビルドコマンド：（なし、静的サイトのため）
   - 出力ディレクトリ：`/`（ルート）
3. デプロイ

#### HTTPS確認
デプロイ後、必ずHTTPSでアクセスできることを確認してください。

---

### iPhoneでのPWAインストール

1. Safari で client URL を開く
2. 「共有」ボタン → 「ホーム画面に追加」
3. ホーム画面から起動してReceiver登録

---

---

## モバイル回線でのIPv6設定（重要）

### なぜIPv6が必要か

モバイル回線のIPv4接続では、キャリアグレードNAT（CGNAT）により複数ユーザーが同じグローバルIPを共有します。この環境ではSTUNサーバーだけではWebRTC接続が確立できません。

IPv6接続では各デバイスがグローバルアドレスを持つため、P2P接続が成功しやすくなります。

### IPv6接続の確認

受信者（スマートフォン）で以下のサイトにアクセス：

- **test-ipv6.com**: https://test-ipv6.com/

「IPv6接続性: あり」と表示されればOKです。

### キャリア別IPv6設定

#### Y!mobile
APN設定を変更：
1. 設定 → モバイル通信 → モバイルデータ通信ネットワーク
2. APN: `plus.acs.jp.v6`
3. ユーザー名: `ym`
4. パスワード: `ym`
5. 再起動して再確認

#### その他主要キャリア
- **ドコモ/au/ソフトバンク**: 通常はIPv6自動適用（追加設定不要）
- **楽天モバイル**: APN `rakuten.jp` でIPv6対応

### IPv4のみの環境での代替案（将来実装）

現在はSTUNのみのため、IPv4+CGNATではつながらない場合があります。  
将来、TURNサーバー導入により改善予定です。


## トラブルシューティング

セットアップ中に問題が発生した場合は、[トラブルシューティング](troubleshooting.md)を参照してください。

---

## 次のステップ

- [アーキテクチャ](architecture.md) - システムの仕組みを理解する
- [ユーザーマニュアル](user-manual.md) - 操作方法を学ぶ
- [API リファレンス](api-reference.md) - カスタマイズのためのAPI仕様
