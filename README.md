<div align="center">

[日本語](README.md) | [English](README_EN.md)

</div>

---

# 緊急コールシステム - ドキュメント

このリポジトリは、緊急コールシステム（Emergency Call System）の包括的なドキュメントを提供します。

## システム概要

Web Push（VAPID）とWebRTC（PeerJS）を組み合わせた、緊急時のビデオ/音声通話システムです。  
受信者は通知を受け取り、タップするだけで自動的に通話が開始されます。

### リポジトリ

- **Client**: https://github.com/curionlab/emergency-call-client
- **Server**: https://github.com/curionlab/emergency-call-server
- **Docs（このリポジトリ）**: https://github.com/curionlab/emergency-call-docs

---

## ドキュメント

### 📖 [セットアップガイド](docs/setup-guide.md)
システムの初期セットアップ手順（ローカル開発環境から本番環境まで）

### 🏗️ [アーキテクチャ](docs/architecture.md)
システムアーキテクチャの全体像、技術スタック、データフロー

### 📡 [API リファレンス](docs/api-reference.md)
サーバーAPIの詳細仕様（エンドポイント、リクエスト/レスポンス形式）

### 👤 [ユーザーマニュアル](docs/user-manual.md)
発信者・受信者それぞれの操作手順

### 🔧 [トラブルシューティング](docs/troubleshooting.md)
よくある問題と解決方法

---

## クイックスタート

最短でシステムを動かしたい方は、[セットアップガイド](docs/setup-guide.md)の「ローカル開発環境」セクションを参照してください。

---

## ライセンス

このドキュメントは MIT License のもとで公開されています。  
システム本体（client/server）も同様に MIT License です。
