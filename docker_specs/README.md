# Docker仕様書インデックス

本ディレクトリは、サーバー上で稼働する各Dockerコンテナの個別仕様書を管理する。

## 仕様書一覧

| ファイル | サービス | 状態 |
| :--- | :--- | :--- |
| [sunshine_streaming.md](./sunshine_streaming.md) | ARグラス向け映像配信（Sunshine） | 📝 仕様検討中 |
| [voicevox.md](./voicevox.md) | AI音声合成エンジン（VOICEVOX） | 📝 仕様検討中 |
| [discord_bot.md](./discord_bot.md) | Discord Bot (Python) | 📝 仕様検討中 |
| [cms_kogecha.md](./cms_kogecha.md) | ヘッドレスCMS（kogecha.org） | 📝 仕様検討中 |

## 共通ルール

各仕様書は以下のセクションを含むこと：

1. **概要** — サービスの目的と役割
2. **ベースイメージ／ビルド** — 使用イメージ or カスタムDockerfileの方針
3. **環境変数** — 必要な設定値
4. **ポート・ネットワーク** — 公開ポートとネットワーク設定
5. **ボリューム** — 永続化が必要なデータ
6. **リソース割当** — CPU cpuset・メモリ上限・GPU有無
7. **依存関係** — 他コンテナ or ホストサービスとの依存
8. **検証項目** — デプロイ後に確認すべき動作
