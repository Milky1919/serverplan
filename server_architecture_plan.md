# サーバー構築・運用計画書 (Ubuntu + RTX 3070 Ti + Docker 統合管理)

---

## 0. ハードウェアリソース割当方針

> **方針:** 具体的なスレッド番号（`cpuset-cpus` の値）は、各サービスの `docker-compose.yml` を作成する段階で負荷特性を見ながら決定する。ここでは割当の基本方針のみを定める。追加サービスが増えるたびに柔軟に変更できる運用とする。

| コアの種類 | 担当する処理の方針 |
| :--- | :--- |
| **Pコア** (Thread 0-11) | 高負荷・リアルタイム性が命の処理（映像配信・エンコード等） |
| **Eコア** (Thread 12-15) | バックグラウンド・軽負荷処理（Bot、同期処理、補助サービス等） |
| **未割当（デフォルト）** | 新規追加サービスはまず割当なしでデプロイ → 負荷状況を見て随時 `cpuset` を追記 |
| **GPU (RTX 3070 Ti)** | 映像エンコード（NVENC）・AI処理が必要なコンテナのみ `--gpus all` を付与 |
| **メモリ (64GB)** | DDR4 16GB×4（CMK32GX4M2E3200C16）。各コンテナに `mem_limit` で上限を設定し、OOMを防止 |
| **ストレージ (1TB)** | Hanye 1TB PCIe Gen4x4 NVMe M.2 SSD。Dockerボリューム領域として利用 |

---

## 1. システム要件・大前提

本サーバーは以下の用途を並立稼働させることを目的とする。

- ARグラス向け映像ストリーミング（低遅延・高画質）
- AI音声合成（VOICEVOX）
- Discord Bot
- Webサイト（`kogecha.org` 向けヘッドレスCMS）
- その他、追加コンテナを随時デプロイできる拡張可能な基盤

ハードウェア: Core i5-12600KF（内蔵GPUなし）＋ RTX 3070 Ti（1枚）

---

## 2. アーキテクチャ方針

Proxmox等のハイパーバイザ・LXCを**採用しない**理由：
- 内蔵GPUが存在しないため、VMへのGPUパススルー時に物理コンソール（画面）が失われるリスクがある。
- LXC + Docker のネスト構成は cgroup・AppArmor の権限設定が複雑でメンテコストが高い。

**採用方針：Ubuntu 24.04 LTS（ベアメタル）+ Docker Compose による完全コンテナ分離**

各アプリケーションの実装詳細は個別のDocker仕様書（`docker_specs/` 配下）に記載し、本書はサーバー基盤に専念する。

---

## 3. ホストOS層（ベアメタルインフラ）

- **OS:** Ubuntu 24.04 LTS（物理マシンへ直接インストール）
- **役割:** コンテナを稼働させるための土台のみ。アプリは原則インストールしない。
- **必須導入パッケージ（ホストに直接導入するもの）:**
    - `NVIDIA Driver 550+`（RTX 3070 Ti 駆動用）
    - `Docker Engine` + `Docker Compose Plugin`
    - `NVIDIA Container Toolkit`（コンテナへのGPUパスに必要）
    - `Tailscale`（ホスト側でVPNメッシュを張り、ストリーミングコンテナに `host` ネットワークで共有）

- **必須のsystemd設定:**
    ```bash
    systemctl enable tailscaled   # 再起動後もTailscaleを自動起動
    systemctl enable docker        # 再起動後もDockerを自動起動
    ```

---

## 4. コントロール・統合管理システム層

### 管理ツール：`Dockge`（採用確定）
- `docker-compose.yml` スタック単位での管理に特化。軽量かつ直感的なUI。
- ブラウザから全コンテナの作成・起動・停止・ログ閲覧・リソース変更をGUI操作で完結。

### セキュアアクセス：`cloudflared`（Cloudflare Tunnel）
- **役割:** Dockge・各種WebUI・`kogecha.org` 等への外部からのアクセス口。
- 動的IPでも固定FQDN（カスタムドメイン）でアクセス可能。ポート開放一切不要。
- Cloudflare Accessと組み合わせてGoogle認証（ゼロトラスト）を適用。

---

## 5. アプリケーション層（稼働コンテナ一覧）

各コンテナの実装詳細（Dockerfile・環境変数・ポート設定等）は `docker_specs/` 配下の個別仕様書を参照のこと。

| カテゴリ | コンテナ | 仕様書 |
| :--- | :--- | :--- |
| 映像配信 | Sunshine（ARグラス向けストリーミング） | `docker_specs/sunshine_streaming.md` |
| AI処理 | VOICEVOXエンジン（音声合成API） | `docker_specs/voicevox.md` |
| バックグラウンド | Discord Bot (Python) | `docker_specs/discord_bot.md` |
| Web | ヘッドレスCMS（kogecha.org） | `docker_specs/cms_kogecha.md` |
| 監視 | Uptime Kuma（死活監視） | ※ 軽量のため割当なし、標準設定で運用 |
| 管理 | Dockge（コンテナ管理UI） | ※ 標準設定で運用 |
| トンネル | cloudflared（Cloudflare Tunnel） | ※ 標準設定で運用 |
| ゲームサーバー等 | マイクラ等（将来追加予定） | 追加時に個別仕様書を作成 |

### 共通設定ルール（全コンテナ共通）

- **自動再起動:** `restart: unless-stopped` を全コンテナで設定する（手動停止以外は常に自動復帰）。
- **メモリ上限:** 高負荷コンテナには `mem_limit` を設定し、ホストのOOM（メモリ枯渇）を防止。
- **重要インフラの保護:** Dockge・Uptime Kuma等の管理系コンテナには `oom_score_adj` を低く設定し、OOM Killerの対象から除外する（負の値には `cap_add: [SYS_RESOURCE]` が必要）。
- **GPU使用:** NVENCまたはCUDA処理が必要なコンテナのみ `deploy.resources.reservations.devices` でGPUを割り当てる。

---

## 6. ネットワーク経路の使い分け

| 用途 | ツール | 理由 |
| :--- | :--- | :--- |
| ARグラス向け映像ストリーミング | **Tailscale** (P2P / UDP) | 低遅延が最重要。CF Tunnelはレイテンシが高くストリーミング不向き |
| Web管理画面・各種UI・kogecha.org | **Cloudflare Tunnel** (TCP) | セキュリティ最重要。Google認証でゼロトラスト保護 |
| マイクラ等ゲームサーバー公開（将来） | **Playit.gg** | ゲーム用プロトコル対応、自宅IP隠蔽、ポート開放不要。追加時に設定する |

---

## 7. バックアップ戦略

- **対象データ:** Dockerボリューム（各サービスのデータ）、Composeファイル群（`/opt/stacks/`）、ホスト設定一式
- **ツール:** `rclone`（Google Drive 2TB へ差分同期）
- **フロー:**
    1. 対象ボリュームを tar.gz 圧縮（稼働中バックアップが必要なサービスは `docker stop` → バックアップ → `docker start`）
    2. `rclone sync` でGoogle Driveへアップロード
    3. 設定ファイル（Composeスタック一式、`rclone.conf`、Dockgeスタック定義）も別途保存
- **復旧:** Google DriveからComposeファイル一式を取得し、`docker compose up -d` を叩くだけで全サービスが復元される設計とする。

---

## 8. 保守・運用・アップデート方針

- **イメージのアップデート戦略（手動・月1回）**
    - Watchtower等による「自動更新」は導入せず、**月1回の手動Pull & 再デプロイ**（Dockge経由）を基本とする。
    - 特定バージョンに強く依存するコンテナが多いため、不意の自動アップデートによる障害を防止する。

- **OOM（メモリ枯渇）対策とホストインフラの保護**
    - 高負荷コンテナには `mem_limit` を適切に設定。
    - 重要インフラ（Dockge, Uptime Kuma 等）は `oom_score_adj` の数値を下げて設定し、OOM Killerの対象から優先的に除外する。

---

## 9. 障害・復旧フロー

| 障害内容 | 対応手順 |
| :--- | :--- |
| コンテナ単体の異常 | DockgeのUI上でコンテナを停止 → 再デプロイ（数十秒で完結） |
| OSレベルの障害 | Ubuntu再インストール → §3の必須パッケージ導入 → Google DriveからComposeファイル取得 → `docker compose up -d` で全復旧 |
| GPU関連エラー | ホストのNVIDIAドライババージョン確認 → 必要に応じてドライバ再インストール（コンテナには影響なし） |
| Tailscale障害 | `systemctl restart tailscaled`（`Restart=always` 設定により通常は自動復帰） |
