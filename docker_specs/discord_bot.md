# Docker仕様書：Discord Bot (Python)

---

## 1. 概要

Python製のDiscord Bot。主にVOICEVOX APIを呼び出してボイスチャンネルで音声読み上げ等を行う。SQLiteで利用制限等を管理。

---

## 2. ベースイメージ／ビルド

- `python:3.12-slim` をベースにカスタムDockerfileで構築
- 依存ライブラリは `requirements.txt` で管理

> 詳細なDockerfileは別途開発

---

## 3. 環境変数

| 変数名 | 値（例） | 説明 |
| :--- | :--- | :--- |
| `DISCORD_TOKEN` | `（シークレット）` | Discord BotのAPIトークン |
| `VOICEVOX_URL` | `http://voicevox:50021` | VOICEVOXエンジンのURL |

---

## 4. ポート・ネットワーク

- 外部公開ポートなし（Discordサーバーへのアウトバウンドのみ）
- VOICEVOXコンテナと同一Dockerネットワークに配置

---

## 5. ボリューム

| コンテナパス | 用途 |
| :--- | :--- |
| `/app/data` | SQLiteデータベース（利用制限記録等）の永続化 |

---

## 6. リソース割当

| リソース | 設定値 | 備考 |
| :--- | :--- | :--- |
| CPU | `cpuset-cpus` (実機確認後に設定) | Eコア（軽負荷） |
| GPU | なし | |
| メモリ | `mem_limit: 512m`（暫定） | 実測後に調整 |

---

## 7. 依存関係

- **VOICEVOX コンテナ:** 音声合成APIが稼働していること

---

## 8. 検証項目

- [ ] Botがオンライン状態になること
- [ ] コマンドに応答すること
- [ ] VOICEVOX APIを呼び出して音声再生ができること
