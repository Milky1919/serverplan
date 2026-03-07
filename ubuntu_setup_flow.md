# Ubuntu 24.04 サーバー構築フロー (Docker運用基盤構築まで)

このドキュメントは、ベアメタルのUbuntu 24.04 LTS環境において、各種Dockerコンテナ（Sunshine, VOICEVOX, Discord Bot等）を手軽にデプロイできる「運用基盤」を完成させるまでの手順をまとめたものです。

---

## 1. OSインストールと初期設定

### 1.1 Ubuntu 24.04 LTS のインストール

> 📖 **詳細な手順書:** [`ubuntu_install_guide.md`](./ubuntu_install_guide.md) を参照してください。

1. Ubuntu Server 24.04 LTS のインストールメディアを作成し、インストールを開始します。
2. インストール時、「**Ubuntu Server (minimized)**」を選択し、余計なパッケージを入れないようにします。
3. インストール時の選択項目で「**Install OpenSSH server**」に必ずチェックを入れ、外部からSSH接続できるようにします。

### 1.2 システムの最新化と基本ツールの導入
インストール完了後、SSHでサーバーにログインし、システムのタイムゾーンを日本時間に変更し、パッケージを最新にします。

```bash
sudo timedatectl set-timezone Asia/Tokyo
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git nano net-tools software-properties-common ubuntu-drivers-common
```

### 1.3 CPUトポロジの確認と割り当て方針
各Dockerコンテナに `cpuset-cpus` を割り当てるため、実機の `lscpu -e` で判明した Core i5-12600KF のトポロジは以下の通りです。

- **Pコア (Performance Core): `0-11`** (最大4.9GHz / 6コア12スレッド)
- **Eコア (Efficient Core): `12-15`** (最大3.6GHz / 4コア4スレッド)

> **ℹ️ コンテナ運用時の目安:**
> - **重負荷（AI音声合成など）:** Pコア（例: `0-7`）
> - **軽負荷（Botなど）:** Eコア（例: `12-15`）またはPコア残り（例: `8-11`）
> 今後作成する `docker-compose.yml` にここで判明した番号を指定します。

---

## 2. NVIDIAドライバのインストール (RTX 3070 Ti用)

GPUを使用するため、プロプライエタリ（非オープンソース）のNVIDIAドライバをインストールします。

### 2.1 推奨ドライバの確認とインストール
```bash
# 利用可能なドライバを確認（ubuntu-drivers-common が必要。§1.2 でインストール済み）
ubuntu-drivers devices

# 【重要】上記の出力で「recommended」と表示されたバージョンを指定します
# 例: `driver   : nvidia-driver-590-open - distro non-free recommended` と出力された場合
sudo apt install -y nvidia-driver-590-open

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

### 5.1 Tailscale のインストール (P2P VPN・Dockge用)
SSH用のZero Trust構築前に、設定用の一時的なアクセス手段とDockge管理用にTailscaleを入れておきます。

```bash
# Tailscaleのインストール
curl -fsSL https://tailscale.com/install.sh | sh

# 再起動後も自動起動するよう有効化
sudo systemctl enable tailscaled

# Tailscaleの起動と認証
sudo tailscale up
```
※ 表示されたURLにブラウザからアクセスし、お持ちのGoogleアカウント等でログインしてルーターに追加します。

接続後、Tailscale の IP アドレスを確認するには以下を実行します。
```bash
tailscale ip -4
```

### 5.2 ファイアウォール (ufw) の確認と設定
Ubuntu Server (minimized) では `ufw` がデフォルトでインストールされていません。LAN内からのアクセスを想定して有効化する場合は以下を実行してください。

```bash
# ufw をインストール
sudo apt install -y ufw

# ufw の状態確認（この時点では inactive のはずです）
sudo ufw status

# ufw を有効化する場合（SSH接続が切断されないよう、22番を先に許可してから enable すること）
sudo ufw allow 22/tcp
sudo ufw allow 5001/tcp   # Dockge
sudo ufw enable
```

> **ℹ️ 注意:** Tailscale / Cloudflare Tunnel 経由のアクセスはポート開放不要です。上記は LAN 内アクセス用途での参考情報です。

### 5.3 Webブラウザからの安全なSSH接続 (Cloudflare Zero Trust)
ターミナルソフトを使わず、スマホやPCのブラウザから `https://ssh.kogecha.org` にアクセスするだけで、Google認証を通って直接サーバーの黒い画面（SSH）を操作できる非常に便利でセキュアな環境を構築します。

#### 5.3.1 サーバー側: cloudflared のインストール
1. [Cloudflare Zero Trust ダッシュボード](https://one.dash.cloudflare.com/)の `Networks` > `Tunnels` でトンネル（例: `homeserver-tunnel`）を作成します。
2. ダッシュボードに表示されるコマンドをサーバーのターミナルに貼り付けて実行します。

```bash
# ダッシュボードから取得したコマンドの例（必ずご自身の環境のトークンを使用してください）
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
sudo cloudflared service install <ダッシュボードから取得したトークン>
```

#### 5.3.2 Cloudflare ダッシュボード側の設定
ダッシュボードで、ブラウザ上のSSHターミナル機能を有効化し、Google認証への紐づけを行います。

1. **Public Hostnameのルーティング設定:**
   - トンネルの `Public Hostname` タブを開きます。
   - `Add a public hostname` をクリックします。
   - **Subdomain:** `ssh` / **Domain:** `kogecha.org`
   - **Service Type:** `SSH` / **URL:** `localhost:22`
   - 保存します。（これで `ssh.kogecha.org` がサーバー自身の内部通信として22番に繋がります）

2. **Access Application (認証とブラウザターミナルの壁) の作成:**
   - ダッシュボードの `Access` > `Applications` に進み、`Add an application` をクリック。
   - `Self-hosted` を選択。
   - **Application Name:** (例: `Browser SSH`) / **Subdomain:** `ssh` / **Domain:** `kogecha.org`
   - **Session Duration:** 認証の有効期限（1ヶ月などに設定）
   - 左側メニューから **Settings** タブを開きます。
   - 少し下にスクロールし、**Browser rendering** という項目のプルダウンを `SSH` に変更します。（**※ここが最重要設定です**）
   - **Identity providers:** Googleを選択。
   - **Policies:** 次の画面で新しいポリシーを作成し、`Include` > `Emails` にご自身のGoogleアカウントメールアドレスを指定します。
   - 最後まで進めて保存します。

#### 5.3.3 ブラウザからの接続確認
1. ブラウザで `https://ssh.kogecha.org` にアクセスします。
2. Cloudflareの画面が表示されるので、ご許可したGoogleアカウントでログインします。
3. 認証が通ると、**ブラウザ内に黒いターミナル画面（Terminal over browser）が起動します。**
4. ログインユーザー名（例: `kenta`）を入力してEnter。
5. パスワードを入力してEnter。（入力が見えませんが反映されています）

> **⚠️ 注意事項:**
> この時点でブラウザから問題なく操作できることを必ず確認してください。ブラウザからのSSH接続が確立する前に以下の設定を行うと、サーバーから締め出されて物理コンソール（モニターとキーボードの直結）しか操作できなくなります。

#### 5.3.4 究極のセキュリティ: SSH直結の完全遮断
「ブラウザからの接続が正常に確認できた後」に、この設定を行います。物理コンソールとCloudflareのトンネル経由以外からは、たとえローカルネットワークにいても一切SSHアクセスできないようにします。

```bash
# 現在のターミナル（またはブラウザSSH）から設定ファイルを開く
sudo nano /etc/ssh/sshd_config
```

一番下の行に、以下の設定を追加します（※ `kenta` の部分は実際のログインユーザー名に変更してください）。

```text
AllowUsers kenta@127.0.0.1 kenta@::1
```

`Ctrl + O` -> `Enter` で保存し、`Ctrl + X` で終了します。
設定を反映させるため、SSHサービスを再起動します。

```bash
sudo systemctl restart ssh
```

> **🛡️ セキュリティ状態の確認:**
> この設定により、LAN内の別PCから `ssh kenta@192.168.x.x` で接続しようとしても、「Permission denied」と弾かれるようになります。これで**完全なゼロトラストSSHの構築完了**です！

### 5.4 今後の通信経路の使い分け（重要: マイクラや動画配信について）
今後Dockerで様々なサービス（Minecraftサーバー、Dockge、動画配信など）を立ち上げる際、**セキュリティと規約の観点から**以下の2つの経路を使い分ける必要があります。

#### ① Cloudflare Tunnel を使うべきもの（Web UIなど）
ブラウザで完結するHTTP/HTTPS通信（Webページ）に向いています。
- **適しているもの:** Dockgeの管理画面（ポート5001）、WordPressなどのWebサイト、各種設定用Web UI
- **設定方法:** 5.3.2と同様に、Cloudflareダッシュボードで `Public Hostname` を追加し、`Service Type` を `HTTP` にして `localhost:<ポート番号>` を指定するだけです。
- **⚠️ 注意点 (規約違反リスク):** Cloudflareの無料プランの規約上、**大量の動画ファイルやストリーミング映像（ネトフリ視聴、Sunshineなどの映像配信）の通信を通すことは禁止されています。** アカウントが凍結される恐れがあるため、動画やゲームのストリーミングは絶対にCloudflareを通さないよう注意してください。

#### ② Tailscale を使うべきもの（ゲーム、動画配信、大容量通信）
P2PのVPNであるTailscaleは、通信がCloudflareのサーバーを経由せず直接暗号化されて届くため、無料で広帯域・低遅延の通信が可能です。またUDPポートも通せます。
- **適しているもの:** Minecraft等ゲームサーバー（TCP/UDP）、Sunshine等映像配信、ネトフリ相当の自前動画サーバー（Jellyfin等）、NASのファイル転送
- **設定方法:** モバイル回線や外出先のPC側にもTailscaleアプリをインストールし、同じアカウントでログインするだけ。あとはサーバーのTailscale IP（`100.x.x.x` など）に向かってアクセスすれば繋がります。ポート開放の設定自体が不要です。

> **💡 結論:**
> - 「Webの管理画面」や「SSH」は **Cloudflare** で独自ドメイン（カッコよく、どこからでもアクセス可）。
> - 「重い通信」や「ゲーム」は **Tailscale** で直結（高速・低遅延・規約違反なし）。
> と使い分けるのが最も安全で快適な運用基盤構築のベストプラクティスです。

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
