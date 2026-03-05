# Ubuntu 24.04 サーバー構築フロー (Docker運用基盤構築まで)

このドキュメントは、ベアメタルのUbuntu 24.04 LTS環境において、各種Dockerコンテナ（Sunshine, VOICEVOX, Discord Bot等）を手軽にデプロイできる「運用基盤」を完成させるまでの手順をまとめたものです。

---

## 1. OSインストールと初期設定

### 1.1 Ubuntu 24.04 LTS のインストール
1. Ubuntu Server 24.04 LTS のインストールメディアを作成し、インストールを開始します。
2. インストール時、「**Ubuntu Server (minimized)**」を選択し、余計なパッケージを入れないようにします。
3. インストール時の選択項目で「**Install OpenSSH server**」に必ずチェックを入れ、外部からSSH接続できるようにします。

### 1.2 システムの最新化と基本ツールの導入
インストール完了後、SSHでサーバーにログインし、パッケージを最新にします。

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano net-tools software-properties-common ubuntu-drivers-common
```

### 1.3 CPUトポロジの確認 (cpuset設定の前提)
各Dockerコンテナに `cpuset-cpus` を設定する前に、必ず実機でCPUスレッドIDを確認してください。

```bash
lscpu -e
```

> **⚠️ 重要:** 上記の出力で、どのスレッドIDがPコア・Eコアに対応しているかを記録しておくこと。`docker-compose.yml` の `cpuset-cpus` に指定する値はこの実機確認に基づきます。

---

## 2. NVIDIAドライバのインストール (RTX 3070 Ti用)

GPUを使用するため、プロプライエタリ（非オープンソース）のNVIDIAドライバをインストールします。

### 2.1 推奨ドライバの確認とインストール
```bash
# 利用可能なドライバを確認（ubuntu-drivers-common が必要。§1.2 でインストール済み）
ubuntu-drivers devices

# 【推奨】上記の出力で「recommended」と表示されたバージョンを指定する（例: nvidia-driver-570）
# バージョンは実行時の環境によって異なるため、必ず確認すること
sudo apt install -y nvidia-driver-<確認したバージョン番号>

# または、自動で推奨ドライバをインストールする場合:
# sudo ubuntu-drivers autoinstall

# 再起動してドライバを適用
sudo reboot
```

### 2.2 ドライバの動作確認
再起動後、ログインしてGPUが認識されているか確認します。
```bash
nvidia-smi
```
※ RTX 3070 Ti の情報が表示されれば成功です。

---

## 3. Docker & Docker Compose のインストール

公式のリポジトリを使用して、最新のDocker環境を構築します。

### 3.1 Docker Engine のインストール
```bash
# 古いバージョンがあれば削除
sudo apt remove docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc

# Dockerの公式GPGキーを追加
sudo apt update
sudo apt install -y ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# リポジトリを追加
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
$(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Dockerをインストール
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 3.2 ユーザーをdockerグループに追加
`sudo` なしで `docker` コマンドを実行できるようにします。
```bash
sudo usermod -aG docker $USER
# 設定を反映させるため、一度SSHをログアウトして入り直す（推奨）
# またはセッションを再起動せず即時反映させたい場合は以下を実行
newgrp docker
```
> **ℹ️ 補足:** `newgrp docker` は現在のシェルセッションのみに有効です。新しいSSHセッションでは自動的にグループが反映されます。

---

## 4. NVIDIA Container Toolkit の導入

Dockerコンテナ内からホストのGPU（RTX 3070 Ti）を利用できるようにするためのツールキットです。

### 4.1 インストール手順
```bash
# リポジトリの設定
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg \
  && curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list | \
    sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' | \
    sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

# インストール
sudo apt update
sudo apt install -y nvidia-container-toolkit

# Dockerデーモンの設定を生成
sudo nvidia-ctk runtime configure --runtime=docker

# Dockerを再起動して設定を適用
sudo systemctl restart docker
```

---

## 5. ネットワークと管理基盤の構築

### 5.1 Tailscale のインストール (P2P VPN・Sunshine用)
```bash
# Tailscaleのインストール
curl -fsSL https://tailscale.com/install.sh | sh

# 再起動後も自動起動するよう有効化
sudo systemctl enable tailscaled

# Tailscaleの起動と認証
sudo tailscale up
```
※ 表示されたURLにブラウザからアクセスし、お持ちのGoogleアカウント等でログインしてルーターに追加します。

接続後、Tailscale の IP アドレスを確認するには以下を実行します（Dockge アクセス時などに使用）。
```bash
tailscale ip -4
```

### 5.2 ファイアウォール (ufw) の確認
Ubuntu Server では `ufw` がデフォルトで無効の場合があります。LAN内からのアクセスを想定して有効化する場合は以下を実行してください。

```bash
# ufw の状態確認
sudo ufw status

# ufw を有効化する場合（SSH接続が切断されないよう、22番を先に許可してから enable すること）
sudo ufw allow 22/tcp
sudo ufw allow 5001/tcp   # Dockge
sudo ufw enable
```

> **ℹ️ 注意:** Tailscale / Cloudflare Tunnel 経由のアクセスはポート開放不要です。上記は LAN 内アクセス用途での参考情報です。

### 5.3 オプショナル: cloudflared のインストール (WebUI用リバースプロキシ)
Dockgeや各種Web UIをセキュアに外部公開する場合にインストールします。

**手順:**
1. [Cloudflare Zero Trust ダッシュボード](https://one.dash.cloudflare.com/) でトンネルを作成します。
2. ダッシュボードに表示される以下の形式のコマンドをそのまま実行してください。

```bash
# Cloudflare ダッシュボードが生成するコマンド（例）
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
sudo cloudflared service install <ダッシュボードから取得したトークン>
```

---

## 6. Dockge のインストール (最終ステップ)

ブラウザベースで `docker-compose.yml` を直感的に管理できる「Dockge」を立ち上げます。
ここからはDocker Composeの出番となります。

### 6.1 デプロイ準備
```bash
# Dockge用のディレクトリと、スタック（Composeファイル）保存用ディレクトリを作成
sudo mkdir -p /opt/dockge
sudo mkdir -p /opt/stacks

# カレントユーザーにオーナーシップを付与してから作業
sudo chown -R $USER:$USER /opt/dockge /opt/stacks
cd /opt/dockge

# Dockgeの Compose ファイルをダウンロード
curl -O https://raw.githubusercontent.com/louislam/dockge/master/compose.yaml
```

### 6.2 Dockgeの起動
```bash
# コンテナをバックグラウンドで起動
docker compose up -d
```

### 6.3 Dockge へのアクセス
1. ブラウザから `http://<サーバーのローカルIP>:5001` または TailscaleのIPでアクセスします。
2. 初回アクセス時に管理者アカウント（ユーザー名・パスワード）を作成します。

---

## 🎉 完成！次のステップへ

以上で、サーバーのインフラ基盤構築は完了です！

以後はサーバーに直接SSH接続して作業する必要はほとんどありません。
ブラウザから **Dockge** にアクセスし、右上の「+ Compose」ボタンから、以下のドキュメントに記載された `docker-compose.yml` の内容を貼り付けてデプロイするだけで、各サービスを簡単に立ち上げることができます。

- 🎮 映像配信基盤: `docker_specs/sunshine_streaming.md`
- 🗣️ AI音声合成: `docker_specs/voicevox.md`
- 🤖 Discord Bot: `docker_specs/discord_bot.md`
