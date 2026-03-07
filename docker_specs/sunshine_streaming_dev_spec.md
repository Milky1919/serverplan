# Docker仕様書：ARグラス向け映像配信（Sunshine）開発詳細

---

## 1. 概要と設計思想

本仕様書は、Ubuntuサーバー上で稼働する「ARグラス向け映像ストリーミング基盤」のDockerコンテナ開発に必要な要件を完全に網羅した単独ドキュメントである。インフラ構成やホストOSのセットアップ手順等の外部ドキュメントを参照することなく、本仕様書のみで `Dockerfile` および `docker-compose.yml` の作成が可能となるよう記述されている。

### 究極の目標とクライアント要件
- **目的**: サーバー側でレンダリングされたリッチな映像を、クライアントに極低遅延で配信する。
- **クライアント環境**: iPhone上の **Moonlight** アプリ（iOS内蔵の強力なハードウェアデコーダーを活用）。
- **出力先**: iPhoneに物理接続された **ARグラス**（RayNeo Air 4 Pro 等）。
- **絶対要件**: ARグラスの物理解像度である **`1920x1080`** を全プロセスで厳密に維持し、一切のスケーリングやアスペクト比変換を許容しない（ピクセルパーフェクトな表示の実現）。

---

## 2. ベースイメージとコンテナ実行権限

| 項目 | 指定・要件 | 理由・詳細 |
| :--- | :--- | :--- |
| **ベースイメージ** | `lizardbyte/sunshine:latest` (Ubuntu 24.04ベース) | 公式イメージを土台にカスタム `Dockerfile` で各種デバイスエミュレーターを統合する。<br>**本番推奨**: `lizardbyte/sunshine:v0.23.1` 等のタグ固定版を使用し、予期しない破壊的変更を防ぐ。開発時は `latest` で最新機能を検証し、安定版が確認できたらタグを固定する運用とする。 |
| **GPUパススルー** | `--gpus all` | ホストのRTX 3070 Tiにアクセスし、**NVENC** によるH.265ハードウェアエンコードを実行させるため。|
| **実行ユーザー** | 非特権ユーザー（例: `sunshine`）の新規作成 | Google Chromeの強力なサンドボックス機構を `--no-sandbox` などの危険なフラグ無しで正常に稼働させるには、root以外の一般ユーザーでのプロセス実行が必須。 |
| **ボリューム権限** | コンテナ起動時の自動修正 | 永続化パス（例: `/home/sunshine/.config/sunshine`）のマウント時、権限エラーを防ぐために起動スクリプト内で `mkdir -p` および `chown` ロジックを組み込むこと。 |

---

## 3. ネットワークとリソース割当

コンテナのパフォーマンスと安定性を担保するためのハードウェアリソース調整要件。

### 3.1 ネットワーク設計
- **ネットワークモード**: `host` (`network_mode: "host"`)
- **理由**: MoonlightのUDP通信において、Dockerのブリッジネットワークを跨ぐことによる遅延（ジッター）を完全に排除するため。ホスト上のTailscale (`tailscale0`) VPN経路をそのまま利用し、外出先のiPhoneからの直接ルーティングを実現する。
- **トレードオフ**: ポート分離レベルのセキュリティは低下するが、Tailscaleの暗号化によって外部からは保護されているため許容する。

### 3.2 リソース制限パラメーター
- **CPUコア指定 (`cpuset-cpus`)**:
  - ホストCPU（Core i5-12600KF）の **Pコア領域（スレッドID `0`〜`11`）** から専用スレッドを割り当てる。
  - *【確認事項】後述*
- **メモリ制限 (`mem_limit`)**:
  - `4g` （暫定の上限値）
  - Chromeのタブ肥大化等によるシステム全体のOOMパニックを防ぐフェイルセーフ。溢れた場合はコンテナごと再起動される。

---

## 4. 仮想プロセスのエミュレーション構造

ヘッドレス環境でARグラス向けUIを破綻なく描画・キャプチャするための三層構造。

### 4.1 仮想ディスプレイ (Xvfb)
- **役割**: メモリ上に `1920x1080x24` のフレームバッファキャンバスを展開する。
- **環境変数**: `DISPLAY=:99` （プロセスの紐付け先）

### 4.2 ウィンドウマネージャー (Fluxbox)
- **役割**: Xvfb単体の欠点（ウィンドウサイズが制御できない問題）を補う。
- **要件**: Chromeが起動した瞬間、枠やタイトルバーなどの装飾を消し去り、強制的に `1920x1080` の**フルスクリーン状態に固定（キオスクモード）**する設定を流し込む。
- **順序**: Xvfb起動完了直後にバックグラウンドプロセスとして起動させる。

### 4.3 仮想オーディオ (PulseAudio)
- **役割**: サーバー上に存在しないサウンドカードを偽装し、音声の迂回経路（ループバック）を作る。
- **実装手順**:
  1. `default.pa` 等の設定で、音を捨てる **`module-null-sink`**（仮想スピーカー）をロードする。
  2. Chromeの再生音を全てこの仮想スピーカーへルーティングする。
  3. 仮想スピーカーに付随する **Monitor端子（録音用マイクとして機能）** を生み出す。
  4. Sunshine側は、環境変数 `PULSE_SERVER` 等を通じ、このMonitor端子から音声をキャプチャする。

---

## 5. GPU競合対策（対 VOICEVOX）

RTX 3070 Ti（VRAM 8GB）は、同居するVOICEVOXコンテナと共有される最もクリティカルなリソースである。

- **処理の分離**: Sunshineのエンコードは **NVENC回路**、VOICEVOXの推論は **CUDAコア** を使用するため、計算処理自体の競合・カクつきは原理上発生しない。
- **最大の脅威**: VOICEVOXの推論スパイクによる **VRAM 8GBの枯渇**。
- **被害シナリオ**: VRAM限界突破によるOOM Killer発動 → Sunshineプロセス強制終了 → ストリーミングの突然のブラックアウト。
- **回避ポリシー**: システム全体の安全弁として、**常に余剰VRAMを2GB以上確保**する。
  - *開発上の対応*: Sunshine側でのVRAM制御は困難なため、稼働時は `nvidia-smi` でモニタリングを行いつつ、VOICEVOX側の起動オプション等でVRAM確保上限に上限（例: 4GB）を設ける運用を前提とする。

---

## 6. セキュリティ設定

### 6.1 コンテナセキュリティ
- **hostネットワークモードのリスク**:
  - ポート分離が効かず、コンテナ侵害時にホストの全ネットワークサービスが露出する。
  - **緩和策**: Tailscale ACLで接続元IPを厳格に制限（自分のiPhoneのみ許可）。
- **WebUI（ポート47990）の保護**:
  - Sunshine管理画面が `0.0.0.0:47990` でリッスンする。
  - **必須対応**: 初回起動時の強力なパスワード設定を徹底する。
  - **推奨**: Tailscale Funnel等を使わず、VPN内部からのアクセスのみに制限。
- **GPUパススルーの権限**:
  - `--gpus all` は `/dev/nvidia*` デバイスへの直接アクセスを許可する。
  - **リスク**: コンテナ内プロセスがNVIDIAドライバの脆弱性を突く攻撃パス。
  - **緩和策**: `--cap-drop=ALL --cap-add=SYS_ADMIN` で最小権限に絞る（ただしChromeサンドボックスとの兼ね合いで調整が必要）。

### 6.2 Chromeサンドボックスの扱い
- **原則**: `--no-sandbox` フラグは**絶対に使用しない**。
- **非特権ユーザー実行の重要性**:
  - rootで実行すると、Chromeの多層サンドボックス（seccomp-bpf, namespace isolation）が無効化される。
  - 非特権ユーザー `sunshine` での実行により、Webサイト由来の攻撃をコンテナ内に封じ込める。
- **必要なケイパビリティ**:
  - ChromeがGPUアクセルを使うには `CAP_SYS_ADMIN` が必要な場合がある。
  - 動作確認後、最小限のcapsに削減する。

---

## 7. ヘルスチェックとモニタリング

### 7.1 プロセス起動シーケンスの制御
コンテナ起動時、以下の順序で各プロセスを立ち上げる必要がある。各ステップで**前段プロセスの起動完了を確実に待機**すること。

1. **Xvfb（仮想ディスプレイ）**
   - 起動コマンド: `Xvfb :99 -screen 0 1920x1080x24 &`
   - **完了判定方法**: X11ソケットファイル `/tmp/.X11-unix/X99` の出現を `inotifywait` または `while` ループでポーリング
   - タイムアウト: 5秒

2. **Fluxbox（ウィンドウマネージャー）**
   - 起動コマンド: `DISPLAY=:99 fluxbox &`
   - **完了判定方法**: `xdpyinfo -display :99` が成功応答を返すまで待機
   - タイムアウト: 3秒

3. **PulseAudio（仮想オーディオ）**
   - 起動コマンド: `pulseaudio --start --exit-idle-time=-1`
   - **完了判定方法**: `pactl info` が正常終了するまで待機
   - タイムアウト: 3秒

4. **Sunshine（ストリーミングサーバー）**
   - 起動コマンド: `sunshine`
   - **完了判定方法**: ログ出力に `[info] Sunshine started` 等のメッセージが出現するまで `tail -f` でモニタ
   - タイムアウト: 10秒

5. **Chrome（アプリケーション）**
   - 起動コマンド: 前述のフラグ群を付与して起動
   - **完了判定方法**: `pgrep -f "chrome.*kiosk"` でプロセス存在を確認
   - **注意**: Sunshineが起動完了してから起動すること（順序逆転すると画面キャプチャに失敗する）

### 7.2 Docker Composeヘルスチェック定義
`docker-compose.yml` に以下のヘルスチェックを実装:

```yaml
healthcheck:
  test: ["CMD-SHELL", "pgrep -f 'sunshine' && pgrep -f 'chrome' && pgrep -f 'Xvfb' && pactl info"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 60s  # 初回起動時は全プロセス立ち上げに時間がかかるため猶予を持たせる
```

### 7.3 GPU使用状況のモニタリング
- **VRAM監視スクリプト** (`/usr/local/bin/vram-monitor.sh`):
  ```bash
  #!/bin/bash
  while true; do
    VRAM_USED=$(nvidia-smi --query-gpu=memory.used --format=csv,noheader,nounits)
    VRAM_TOTAL=$(nvidia-smi --query-gpu=memory.total --format=csv,noheader,nounits)
    VRAM_PERCENT=$((VRAM_USED * 100 / VRAM_TOTAL))

    if [ $VRAM_PERCENT -gt 75 ]; then
      echo "[WARNING] VRAM usage: ${VRAM_PERCENT}% (${VRAM_USED}MB / ${VRAM_TOTAL}MB)" | logger -t sunshine-vram
    fi
    sleep 60
  done
  ```
- コンテナ起動時にバックグラウンド実行し、75%超過で警告ログを出力。

### 7.4 ログ出力とデバッグ
- **標準出力への集約**: 各プロセス（Xvfb, Fluxbox, PulseAudio, Sunshine, Chrome）のログを全て標準出力にリダイレクトし、`docker logs` で一元確認できるようにする。
- **ログレベル**: Sunshineは `--verbose` フラグで詳細ログを有効化（開発環境のみ）。
- **Netflix解像度デバッグ**: Chrome起動時に `--enable-logging --v=1` を追加し、`/home/sunshine/.config/google-chrome/chrome_debug.log` に詳細ログを出力させる。

---

## 8. 開発者向け確認事項（未定義項目）

本仕様に基づき `Dockerfile` / `docker-compose.yml` を実装するにあたり、運用側・環境固有の値として確定が必要なパラメータ群。

> [!WARNING]
> 以下の項目については、実装前に決定方針を確認すること。

1. **CPU割当スレッド（`cpuset-cpus`）の確定**
   - Pコア（`0-11`）の中から、Sunshine専用に割り当てる具体的なスレッド番号（例: `0-3` など2〜4スレッド程度）。他プロセスとの物理的な重複を避ける必要がある。
   - **暫定デフォルト**: `0-3`（4スレッド）を仮設定し、実運用で調整可能にする。

2. **WebUI（ポート47990）の初期クレデンシャル設定フロー**
   - 初回起動時、管理者パスワードをブラウザから設定するステップが存在する。これを手動運用とするか、起動スクリプトでシークレット機能等を用いて自動注入するか。
   - **暫定方針**: 初回のみ手動設定を許容し、設定ファイルを永続化ボリュームに保存する。

3. **Netflix Premium契約の確認**
   - 1080p配信を受けるには、Netflixの**スタンダードプラン以上**の契約が必要（ベーシックプランは480pまで）。
   - 仕様実装前に契約プランを確認すること。

   #### 3-1. Netflix 1080p配信のための必須実装（2026年3月時点の最新手法）

   **背景**: Linux環境ではWidevine L3（ソフトウェアDRM）の制限により、Netflix配信は**デフォルトで720pに制限**される。ARグラスの物理解像度 `1920x1080` を活かすため、以下の技術的回避策を**必ず実装**すること。

   **方式A: User-Agent偽装（推奨・最も安定）**
   - ChromeOSのUAを偽装することで、Netflixサーバーに「Chromebookからのアクセス」と認識させ1080p配信を有効化する。
   - **起動オプション**:
     ```bash
     google-chrome \
       --user-agent="Mozilla/5.0 (X11; CrOS x86_64 15509.89.0) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/120.0.0.0 Safari/537.36" \
       --kiosk "https://www.netflix.com/browse"
     ```
   - **UA文字列の注意点**:
     - `CrOS` キーワードが必須（これがChromeOS判定の鍵）
     - Chromeバージョン番号は120以降を推奨（古すぎるとNetflixが弾く可能性）
     - 2026年3月時点で動作確認されているUA: `CrOS x86_64 15509.89.0`

   **方式B: ブラウザ拡張機能（UA偽装が効かない場合のフォールバック）**
   - Dockerビルド時に拡張機能をプリインストールする手法。
   - **推奨拡張機能**:
     - [Netflix 1080p (truedread版)](https://github.com/truedread/netflix-1080p): プロトコルレベルでビットレート要求を書き換える
     - [Netflux (Firefox)](https://addons.mozilla.org/ja/firefox/addon/netflux/): Firefox環境で5.1chオーディオ対応
   - **実装手順**（Chrome拡張の場合）:
     1. GitHub Release から crx ファイルをダウンロード
     2. Dockerfile内で `/opt/chrome-extensions/` へコピー
     3. Chrome起動オプションに `--load-extension=/opt/chrome-extensions/netflix-1080p` を追加
   - **注意**: 拡張機能方式はNetflixのポリシー変更で突然動作しなくなるリスクあり。方式Aを優先すること。

   **方式C: Widevine DRMの確認と強制有効化**
   - ベースイメージ `lizardbyte/sunshine:latest` に Widevine (`libwidevinecdm.so`) が含まれているか検証が必要。
   - 確認コマンド:
     ```bash
     find / -name "libwidevinecdm.so" 2>/dev/null
     ```
   - 不在の場合、Google Chromeの公式debパッケージから抽出して配置:
     ```dockerfile
     RUN wget https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && \
         ar x google-chrome-stable_current_amd64.deb && \
         tar -xf data.tar.xz && \
         mkdir -p /opt/google/chrome/WidevineCdm && \
         cp -r opt/google/chrome/WidevineCdm/* /opt/google/chrome/WidevineCdm/
     ```
   - Chrome設定で Protected Content を有効化: `chrome://settings/content/protectedContent`

   **解像度検証手順（実装後の動作確認）**
   - Netflix再生中に `Ctrl+Shift+Alt+D` で統計オーバーレイを表示
   - 以下を確認:
     - **Playing bitrate**: 5000 kbps 以上（1080pの目安）
     - **Resolution**: `1920x1080` と表示されること
     - **Video codec**: `H264 High 4.0` または `HEVC Main10`
   - Sunshineのログで実際にキャプチャされている解像度が `1920x1080` であることを二重確認。

   #### 3-2. ARグラス特化のUI制御オプション
   - **起動URLのデフォルト設定**:
     - 開発環境: `https://www.netflix.com/browse`
     - 本番調整: 環境変数 `CHROME_START_URL` で上書き可能にする

   - **必須Chromeフラグ**:
     ```bash
     --kiosk                                 # 全画面モード（Fluxboxと併用）
     --force-device-scale-factor=1.0         # ピクセル比固定（HiDPI誤認防止）
     --disable-features=OverlayScrollbar      # スクロールバー非表示
     --disable-infobars                       # 「Chromeは自動テストソフトウェアによって制御されています」警告を消す
     --no-first-run                           # 初回起動ウィザードをスキップ
     --disable-gpu-vsync                      # VSync無効化（遅延削減）
     --enable-features=VaapiVideoDecoder      # ハードウェアデコード有効化（NVIDIA環境では効果薄いが念のため）
     ```

   - **NVENC連携のための環境変数**:
     ```bash
     export LIBVA_DRIVER_NAME=nvidia
     export VDPAU_DRIVER=nvidia
     ```

   #### 3-3. アイドル制御とリソース管理
   - **暫定方針**: Moonlight切断後もChromeプロセスは**維持**する（VRAM常時確保）
   - **理由**:
     - 再接続時の起動遅延回避（Chrome + Netflix初期化で3〜5秒のロスが発生）
     - Widevine DRMセッションの再確立コストが高い
   - **将来的な最適化案**:
     - Sunshineの接続イベントフックで `systemctl start chrome.service` によるオンデマンド起動
     - ただしVRAM節約が必要になった段階での検討事項とする。

   #### 3-4. 広告ブロック・自動スキップ機能の実装（2026年3月時点）

   **技術的背景**:
   - Netflix広告ありプラン、YouTube等の動画サービスに表示される広告を、技術的手段により削減またはスキップする機能。
   - ARグラス視聴体験の最適化（中断のない連続視聴）を実現するための実装オプション。

   **重要な環境変化（2024-2026年）**:
   - Chrome Manifest V3への完全移行により、従来の強力な広告ブロッカー（uBlock Origin完全版等）が**Chrome Web Storeから削除**された（2024年後半）。
   - 全てのManifest V2拡張機能が**2025年7月に無効化**され、機能制限版のMV3拡張のみ利用可能。
   - Docker環境では、拡張機能を**手動インストール**する必要がある。

   **方式A: 広告自動スキップ拡張機能（推奨）**

   Netflix/YouTube等の「スキップボタン」を自動検出して即座にスキップする方式。広告ブロックではなく、広告の再生時間を最小化する。

   - **Netflix-Prime-Auto-Skip** (オープンソース):
     - GitHub: [Dreamlinerm/Netflix-Prime-Auto-Skip](https://github.com/Dreamlinerm/Netflix-Prime-Auto-Skip)
     - Netflix、Prime Video、Disney+、Crunchyroll等の主要サービスに対応
     - イントロ、エンディング、広告の自動スキップ
     - **実装手順**:
       ```dockerfile
       # Dockerfile内でGitHubからクローン
       RUN git clone https://github.com/Dreamlinerm/Netflix-Prime-Auto-Skip.git /opt/extensions/auto-skip && \
           cd /opt/extensions/auto-skip && \
           # ビルドが必要な場合はnpm install && npm run build
       ```
     - Chrome起動オプションに追加:
       ```bash
       --load-extension=/opt/extensions/auto-skip
       ```

   - **Multi Skipper** (Chrome Web Store):
     - 広告スキップボタンを検出した瞬間に自動クリック
     - Netflix、YouTube、Crunchyroll対応
     - 手動インストール: Chrome Web StoreからIDを取得し、crxファイルをダウンロード

   **方式B: 広告ブロッカー拡張機能（YouTube等向け）**

   Netflix以外の動画サービス（YouTube、Twitch等）での広告削減用。

   - **uBlock Origin Lite** (Manifest V3版):
     - 完全版のuBlock Originは使用不可（MV2のため）
     - Lite版は**フィルタルール数に制限**があり、効果が限定的
     - **インストール方法**:
       ```dockerfile
       # GitHubからLite版をダウンロード
       RUN wget https://github.com/uBlockOrigin/uBOL-home/releases/latest/download/uBOLite.chromium.zip && \
           unzip uBOLite.chromium.zip -d /opt/extensions/ublock-lite
       ```
     - Chrome起動オプション:
       ```bash
       --load-extension=/opt/extensions/ublock-lite
       ```
     - **注意**: YouTube側の広告回避対策（adblock検出）により、定期的に無効化される可能性あり。

   - **AdGuard for Chrome** (MV3対応版):
     - Manifest V3への移行が最も成功した商用広告ブロッカー
     - 直感的なUI、細かい制御が可能
     - **ライセンス**: 無料版は基本機能のみ、フル機能は有料
     - インストール: Chrome Web StoreのIDから取得可能

   - **Blockify**:
     - 動画・音声ストリーミング（YouTube、Twitch、Spotify）に特化
     - スマート広告スキップ技術を採用
     - 2026年時点でYouTube広告に最も効果的とされる

   **方式C: システムレベル広告ブロック（Docker環境では非推奨）**

   - **Pi-hole / AdGuard Home**:
     - DNSレベルでの広告ドメインブロック
     - コンテナ外（ホストOS側）での実装が必要
     - YouTube等の埋め込み広告には**効果が薄い**（同一ドメインから配信されるため）
     - **本仕様の範囲外**: 別途ネットワークインフラとして構築する場合のみ検討

   **実装上の注意事項**:

   1. **拡張機能の永続化**:
      - `/home/sunshine/.config/google-chrome/Default/Extensions/` を永続化ボリュームにマウント
      - コンテナ再起動時に拡張機能が消えないようにする

   2. **拡張機能の自動更新無効化**:
      - 手動インストール版は自動更新されないため、定期的な手動更新が必要
      - Chrome起動オプションに `--disable-extensions-except=` と `--load-extension=` を併用

   3. **複数拡張機能の同時読み込み**:
      ```bash
      google-chrome \
        --load-extension=/opt/extensions/auto-skip,/opt/extensions/ublock-lite,/opt/extensions/netflix-1080p \
        ...
      ```

   4. **パフォーマンス影響**:
      - 拡張機能が多すぎるとChromeのメモリ使用量が増加
      - 最大3〜4個の拡張機能に留める（Netflix 1080p、Auto-Skip、uBlock Lite程度）

   **検証手順**:
   - Netflix広告ありプランで動画再生 → 広告が自動スキップされるか確認
   - YouTube動画再生 → 広告がブロックまたはスキップされるか確認
   - Chrome拡張機能ページ (`chrome://extensions`) で拡張機能が有効化されているか確認

---

## 9. 実装後の検証手順（受入テスト）

本仕様に基づき実装した `Dockerfile` および `docker-compose.yml` が正しく動作するかを確認するための検証チェックリスト。

### 9.1 コンテナ起動と基本動作確認

- [ ] **1. コンテナが正常に起動する**
  ```bash
  docker-compose up -d
  docker-compose ps  # State が "Up (healthy)" になっているか確認
  ```

- [ ] **2. 全プロセスが稼働している**
  ```bash
  docker exec -it sunshine ps aux | grep -E "Xvfb|fluxbox|pulseaudio|sunshine|chrome"
  ```
  上記5プロセスが全て存在すること。

- [ ] **3. GPUが認識されている**
  ```bash
  docker exec -it sunshine nvidia-smi
  ```
  RTX 3070 Ti が表示され、Sunshineプロセスが GPU 0 を使用していることを確認。

### 9.2 Sunshine WebUIアクセス確認

- [ ] **4. WebUIにアクセスできる**
  - ホストから `https://<サーバーのTailscale IP>:47990` にアクセス
  - 初回起動時は管理者パスワード設定画面が表示される → 強力なパスワードを設定

- [ ] **5. Sunshineの設定を確認**
  - WebUI → Settings → Video で以下を確認:
    - **解像度**: `1920x1080` が選択可能か
    - **エンコーダー**: `NVENC` が選択されているか
    - **ビットレート**: 最低でも 10 Mbps 以上に設定

### 9.3 Moonlight接続テスト

- [ ] **6. iPhone MoonlightアプリからサーバーをPIN接続**
  - Moonlightアプリで「PCを追加」→ PINコードを入力して接続確立

- [ ] **7. ストリーミング開始**
  - 解像度 `1920x1080`、フレームレート `60fps` を選択して接続
  - ARグラスに映像が表示されることを確認

- [ ] **8. 遅延とフレームレートの確認**
  - Moonlightの統計情報（設定で有効化）を確認:
    - **Network latency**: 30ms 以下（理想は10ms以下）
    - **Frame rate**: 安定して 60fps を維持
    - **Packet loss**: 0% であること

### 9.4 Netflix 1080p配信の検証（最重要）

- [ ] **9. Chromeが1920x1080でフルスクリーン起動している**
  ```bash
  docker exec -it sunshine xdpyinfo -display :99 | grep dimensions
  # 出力: dimensions:    1920x1080 pixels
  ```

- [ ] **10. Netflixにログインして動画を再生**
  - Moonlight経由でキーボード/マウス操作を行い、Netflixにアクセス
  - 任意の動画を再生開始

- [ ] **11. 統計情報で1080p配信を確認**
  - Netflix再生中に `Ctrl+Shift+Alt+D` を押す
  - 統計オーバーレイで以下を確認:
    - **Playing bitrate**: 5000 kbps 以上
    - **Resolution**: `1920x1080`（720pと表示される場合は設定が失敗）
    - **Buffer**: 安定して緑色（バッファリングなし）

- [ ] **12. Sunshineのログで解像度を二重確認**
  ```bash
  docker logs sunshine 2>&1 | grep -i resolution
  # "Capturing 1920x1080" 等のログが出力されていることを確認
  ```

### 9.5 GPU競合とVRAM管理の確認

- [ ] **13. VRAM使用量の測定**
  ```bash
  nvidia-smi --query-gpu=memory.used,memory.total --format=csv
  ```
  - Sunshine単独稼働時のVRAM使用量を記録（例: 1200MB）
  - VOICEVOXと同時稼働時の合計が 6GB 以下（8GBの75%）に収まることを確認

- [ ] **14. 長時間稼働テスト**
  - 1時間以上Netflixを再生し続け、以下を確認:
    - VRAMリークが発生していないか（`nvidia-smi` で定期チェック）
    - フレームレートが低下していないか
    - コンテナが OOM Killer で強制終了されていないか

### 9.6 音声配信の確認

- [ ] **15. PulseAudioのダミーシンクが作成されている**
  ```bash
  docker exec -it sunshine pactl list sinks short
  # "null-sink" または "dummy" という名前のシンクが存在すること
  ```

- [ ] **16. Netflix音声がMoonlight経由で聞こえる**
  - ARグラス（またはiPhoneスピーカー）から音声が出力されることを確認
  - 映像と音声の同期ズレがないか確認

### 9.7 広告ブロック・自動スキップ機能の確認（実装した場合）

- [ ] **17. Chrome拡張機能が正しく読み込まれている**
  ```bash
  docker exec -it sunshine bash -c "DISPLAY=:99 google-chrome --headless --dump-dom chrome://extensions 2>&1 | grep -E 'Netflix.*Skip|uBlock|AdGuard'"
  ```
  インストールした拡張機能の名前が表示されることを確認

- [ ] **18. Netflix広告スキップの動作確認**（広告ありプランの場合）
  - Netflix広告ありプラン対応作品を再生
  - 広告が開始されたら、5秒以内に自動スキップされるか確認
  - スキップボタンが出現する前に飛ばされる場合は正常動作

- [ ] **19. YouTube広告ブロックの動作確認**
  - YouTube動画を再生し、プレロール広告がブロックされるか確認
  - ブロックされない場合: 拡張機能のフィルタリストが最新か確認
  - YouTube側の検出メッセージが出る場合: uBlock Liteのルール更新または広告ブロッカー無効化を検討

### 9.8 障害復旧テスト

- [ ] **20. コンテナ再起動後も設定が永続化されている**
  ```bash
  docker-compose restart
  ```
  - Sunshine WebUIの設定が消えていないか
  - Netflixのログイン状態が保持されているか（Chromeのプロファイルが永続化されているか）
  - インストールした拡張機能が消えていないか

- [ ] **21. プロセスクラッシュ時の自動再起動**
  - Chromeプロセスを手動でkillし、自動再起動されるか確認
  ```bash
  docker exec -it sunshine pkill -9 chrome
  sleep 10
  docker exec -it sunshine pgrep chrome  # プロセスが復活しているか
  ```

---

## 10. 既知の制限事項と将来的な改善点

### 10.1 現時点での制限
- **Netflix 4K（Ultra HD）は不可**: Linux環境ではWidevine L3の制限により、最大1080pまで。4K配信にはWidevine L1（ハードウェアDRM）が必要だが、Linuxでは未サポート。
- **HDRコンテンツの色域制限**: HDR10メタデータは配信されるが、Xvfbが8bit色深度のため、実際の表示はSDRに近い。
- **Chrome拡張機能の脆弱性**: Netflixのプロトコル変更で突然動作しなくなるリスクがある。定期的な動作確認が必要。

### 10.2 将来的な改善案
- **Wayland + wlroots への移行**: X11（Xvfb）の代わりにWaylandコンポジタを使用し、10bit色深度対応を実現。
- **AV1コーデック対応**: NetflixがAV1配信を拡大した場合、Chrome側でのデコード対応とNVENCのAV1エンコード（RTX 40シリーズ以降）の検討。
- **オンデマンド起動の実装**: Tailscaleのネットワークイベントと連動し、iPhone接続時のみコンテナを起動してVRAMを節約。

---

## 参考情報・出典

本仕様書の作成にあたり、以下の技術情報を参照・検証した。

### Netflix 1080p on Linux関連
- [The Quest for Netflix on Asahi Linux](https://www.da.vidbuchanan.co.uk/blog/netflix-on-asahi.html) - Widevine L3の制限とプロトコル解析
- [GitHub - truedread/netflix-1080p](https://github.com/truedread/netflix-1080p) - Chrome拡張機能の実装
- [Netflux - Firefox拡張機能](https://addons.mozilla.org/ja/firefox/addon/netflux/) - Firefox向けの1080p対応
- [Watch Netflix in 1080p on Linux and unsupported browsers](https://www.ghacks.net/2018/02/12/watch-netflix-in-1080p-on-linux-and-unsupported-browsers/) - User-Agent偽装手法

### Widevine DRM on Docker/Linux
- [GitHub - proprietary/chromium-widevine](https://github.com/proprietary/chromium-widevine) - ChromiumへのWidevineインストール手順
- [Install Docker-based Chromium Media Edition on Manjaro-ARM](https://www.linkedin.com/pulse/install-configure-docker-based-chromium-media-edition-joseph-davis) - DockerでのDRMコンテンツ再生

### Sunshine / Moonlight技術仕様
- [LizardByte/Sunshine公式ドキュメント](https://docs.lizardbyte.dev/projects/sunshine/) - Sunshineの設定リファレンス
- [Moonlight Game Streaming](https://moonlight-stream.org/) - Moonlightクライアントの対応コーデック

### 広告ブロック・自動スキップ関連
- [GitHub - Dreamlinerm/Netflix-Prime-Auto-Skip](https://github.com/Dreamlinerm/Netflix-Prime-Auto-Skip) - Netflix/Prime等の広告・イントロ自動スキップ
- [Multi Skipper - Chrome Web Store](https://chromewebstore.google.com/detail/multi-skipper-skip-ads-in/bimlffinhbdhgpomhngmnhidjgnfcnoc?hl=en) - Netflix/YouTube広告スキップ拡張機能
- [uBlock Origin Lite](https://chromewebstore.google.com/detail/ublock-origin-lite/ddkjiahejlhfcafbddmgiahcphecmpfh?hl=en) - Manifest V3対応版広告ブロッカー
- [How To Block Ads on Netflix in 2026](https://allaboutcookies.org/block-netflix-ads) - Netflix広告ブロック手法まとめ
- [Best YouTube Ad Blockers in 2026](https://allaboutcookies.org/best-ad-blockers-for-youtube) - YouTube広告ブロック比較
- [GitHub - gorhill/uBlock](https://github.com/gorhill/uBlock) - uBlock Origin本家（手動インストール用）

---

**本仕様書のバージョン**: v1.3 (2026-03-08)
**最終更新**: 広告ブロック・自動スキップ機能の実装要件を追加（Manifest V3対応版）、技術仕様に特化
