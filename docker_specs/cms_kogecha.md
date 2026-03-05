# Docker仕様書：ヘッドレスCMS（kogecha.org）

---

## 1. 概要

`kogecha.org` ドメインに向けて配信する自作ヘッドレスCMSのコンテナ。Cloudflare Tunnel経由で外部公開する。フロントエンドとバックエンドの構成は別途開発で決定する。

---

## 2. ベースイメージ／ビルド

> 自作のため、ベースイメージ・フレームワーク・言語等は別途開発時に決定する。

---

## 3. 環境変数

> 開発時に決定する。

---

## 4. ポート・ネットワーク

- Cloudflare Tunnel（`cloudflared`）経由で `kogecha.org` に接続
- 外部に直接ポートは開放しない
- Cloudflare Tunnelが `http://cms:（ポート番号）` に向けてリバースプロキシする構成

---

## 5. ボリューム

> コンテンツデータ・DBの永続化要件は開発時に決定する。

---

## 6. リソース割当

| リソース | 設定値 | 備考 |
| :--- | :--- | :--- |
| CPU | 割当なし（暫定） | 実測後に必要であれば設定 |
| GPU | なし | |
| メモリ | `mem_limit: 1g`（暫定） | 実測後に調整 |

---

## 7. 依存関係

- **cloudflared コンテナ（またはホストレベルのTunnel設定）:** `kogecha.org` への経路が確立していること
- **Cloudflare DNS:** `kogecha.org` がCloudflare管理下にあること

---

## 8. 検証項目

- [ ] `https://kogecha.org` にブラウザからアクセスできること
- [ ] CMSのAPIエンドポイントが正常に応答すること
- [ ] Cloudflare AccessのGoogle認証が管理画面に適用されていること（必要な場合）
