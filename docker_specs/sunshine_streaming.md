# Docker仕様書：ARグラス向け映像配信（Sunshine）

---

## 1. 概要

スマートフォン経由でARグラス（RayNeo Air 4 Pro 等）に映像をストリーミングするコンテナ。
サーバー上の仮想ディスプレイ（Xvfb）でブラウザ等を動作させ、Sunshineが映像をキャプチャ・エンコードして配信する。

---

## 2. ベースイメージ／ビルド

- `lizardbyte/sunshine` 公式Dockerイメージをベースに **カスタムDockerfile** で構築
- 追加インストール: `Xvfb`（仮想ディスプレイ）、`PulseAudio`（仮想音声）、Google Chrome
- ベースOS: Ubuntu 24.04

> 詳細なDockerfileは別途開発

---

## 3. 環境変数

| 変数名 | 値（例） | 説明 |
| :--- | :--- | :--- |
| `DISPLAY` | `:99` | Xvfbの仮想ディスプレイ番号 |
| `RESOLUTION` | `1920x1080x24` | 仮想ディスプレイ解像度 |
| `PULSE_SERVER` | `unix:/tmp/pulse/native` | PulseAudioソケット |

---

## 4. ポート・ネットワーク

- **ネットワークモード:** `network_mode: "host"`
    - ホストのTailscale VPN経路をそのまま利用し、Moonlightクライアント（スマホ）とUDP通信する
    - ⚠️ コンテナ分離が弱まるトレードオフを意図的に受容（低遅延優先）

---

## 5. ボリューム

| コンテナパス | 用途 |
| :--- | :--- |
| `/root/.config/sunshine` | Sunshine設定ファイルの永続化 |

---

## 6. リソース割当

| リソース | 設定値 | 備考 |
| :--- | :--- | :--- |
| CPU | `cpuset-cpus` (実機確認後に設定) | Pコア（リアルタイム配信） |
| GPU | `--gpus all` | NVENCによるH.265エンコード |
| メモリ | `mem_limit: 4g`（暫定） | 実測後に調整 |

---

## 7. 依存関係

- **ホスト:** `tailscaled` が起動していること（`network_mode: host` で共有）
- **ホスト:** NVIDIA Driver 550+ および NVIDIA Container Toolkit が導入済みであること

---

## 8. GPU競合ポリシー（VOICEVOX共存）

RTX 3070 Ti（VRAM 8GB）をSunshineとVOICEVOXで共有する。優先順位はSunshine優先。

| コンテナ | VRAM使用量（目安） |
| :--- | :--- |
| Sunshine（1080p/60 H.265 NVENC） | 約1〜2GB |
| VOICEVOX（モデルロード＋推論） | 約2〜6GB |
| 余剰（バースト吸収） | 2GB以上を確保 |

同時高負荷時はVOICEVOX側の同時推論数を制限する。`nvidia-smi` で定期的にVRAM使用量・GPU温度を確認。

---

## 9. 検証項目

- [ ] Xvfbが1920×1080で起動すること
- [ ] SunshineのWebUI（ポート47990）にアクセスできること
- [ ] MoonlightクライアントからTailscale経由でペアリングできること
- [ ] 配信映像が1080p/60fpsで受信できること
- [ ] NVENCが使用されていること（`nvidia-smi` でGPU使用率が上昇することを確認）
