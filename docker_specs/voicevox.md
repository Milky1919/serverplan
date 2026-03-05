# Docker仕様書：AI音声合成エンジン（VOICEVOX）

---

## 1. 概要

VOICEVOX公式GPU対応イメージを使用し、RTX 3070 TiのCUDAで高速音声合成を行うAPIサーバー。Discord Bot等から内部ネットワーク経由でHTTP APIとして呼び出す。

---

## 2. ベースイメージ／ビルド

- `voicevox/voicevox_engine` 公式GPU対応イメージを使用（カスタムビルド不要）

---

## 3. 環境変数

| 変数名 | 値（例） | 説明 |
| :--- | :--- | :--- |
| `VOICEVOX_CORS_POLICY_MODE` | `localapps` | CORS設定 |

---

## 4. ポート・ネットワーク

| ポート | 用途 |
| :--- | :--- |
| `50021` | VOICEVOX HTTP API |

- 外部には非公開。Dockerの内部ネットワーク経由でBot等からのみアクセス可能とする。

---

## 5. ボリューム

- 特になし（モデルはイメージに同梱）

---

## 6. リソース割当

| リソース | 設定値 | 備考 |
| :--- | :--- | :--- |
| CPU | `cpuset-cpus: "12-15"` | Eコア（前処理・後処理用） |
| GPU | `--gpus all` | CUDA推論 |
| メモリ | `mem_limit: 8g`（暫定） | モデル数・同時推論数で変動。実測後に調整 |

---

## 7. 依存関係

- **ホスト:** NVIDIA Driver 550+ および NVIDIA Container Toolkit が導入済みであること
- **GPU競合:** Sunshine（NVENC）と同一GPUを使用する。詳細は `sunshine_streaming.md` のGPU競合ポリシーを参照

---

## 8. 検証項目

- [ ] `http://localhost:50021/docs` にアクセスしてSwagger UIが表示されること
- [ ] 音声合成APIにリクエストを送り、wavファイルが返ってくること
- [ ] `nvidia-smi` でCUDA使用が確認できること
