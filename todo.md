# QuestBot XR TODO

設計書：[docs/questbot_xr_design.md](docs/questbot_xr_design.md)

最初のマイルストーンは **フェーズ0〜1**（模擬スキャン → 実機 `/scan` のVR表示）。速度指令は実装しない。
実機で確認していない項目は完了にしない。確認結果はREADMEへ記録する。

---

## 事前確認（環境調査）

- [ ] Ubuntu PCのOSバージョンを確認（`lsb_release -a`）
- [ ] Ubuntu PCのROS 2ディストリビューションを確認
  - [ ] 24.04の場合：Humble用のVM・コンテナ・対応OSを決定（22.04用aptを直接入れない）
  - [ ] 選んだ環境でDDSのLAN通信とGUI表示を検証
- [ ] Burger側の確認
  - [ ] `/scan` の継続受信とQoS（Best Effort）を再確認（`ros2 topic info -v /scan`）
  - [ ] LiDARの正式型番を確認（LDS-03 / coin_d4）
  - [ ] `/cmd_vel` の型を確認（`ros2 topic type /cmd_vel`、Twist / TwistStamped）※フェーズ5用に記録のみ
  - [ ] TFツリーを確認（`base_footprint` / `base_link` / `base_scan` のフレーム名）
- [ ] ネットワーク確認
  - [ ] Ubuntu PC（有線）・Burger・Quest 3（Wi-Fi）が同一LANで端末間通信できる（ゲストWi-Fi不可）
  - [ ] `ROS_DOMAIN_ID=30`、`ROS_LOCALHOST_ONLY=0` で Burger ↔ Ubuntu のトピック疎通
  - [ ] Ubuntu PCのLAN IPv4アドレスを記録
- [ ] MacのUnity環境を確認（Unity Hub、Editorバージョン）
- [ ] Quest 3の接続状態を確認（開発者モード、USBデバッグ、Mac許可）

## リポジトリ初期化

- [ ] `git init`、`.gitignore` 作成（Library / Temp / Logs / Builds / 秘密情報を除外）
- [ ] 推奨ディレクトリ構成を作成（設計書 §11）
  - [ ] `README.md`、`CLAUDE.md`
  - [ ] `docs/design.md`（設計書の配置整理）、`docs/setup-mac.md`、`docs/setup-ubuntu.md`、`docs/setup-quest.md`、`docs/acceptance.md`
  - [ ] `unity/QuestBotXR/`（Scripts: Networking / Visualization / XR / UI、Scenes、Tests）
  - [ ] `ros2_ws/src/questbot_xr_bridge/`（パッケージ本体・launch・config・test）
  - [ ] `scripts/`
- [ ] ROS-TCP-Endpointを固定リビジョンの依存として管理する方法を決定（submodule / vcs repos など）

## Unity・Quest開発環境（Mac）

- [ ] Quest/OpenXR互換を確認したUnity 6 LTS系Editorを導入
- [ ] Android Build Support、Android SDK & NDK Tools、OpenJDK を追加
- [ ] Unityプロジェクト `unity/QuestBotXR` を作成（URP）
- [ ] XR Plug-in Management、OpenXR、XR Interaction Toolkit を導入
- [ ] Android・ARM64・IL2CPP を設定し、Project Validation を解消
- [ ] ネットワーク権限を有効化（LAN関連権限は実機で確認）
- [ ] Unity / パッケージ / Endpoint のバージョン・コミットを固定して記録
- [ ] 手作業手順を `docs/setup-mac.md`・`docs/setup-quest.md` に記載

---

## フェーズ0：ROSとUnityの疎通・模擬スキャン

完了条件：Mac EditorとQuest APKの両方で同じ模擬形状を表示

### 通信（Ubuntu）
- [ ] ROS-TCP-Endpoint（`main-ros2`）をビルド・起動（ブランチ・CLI引数を公式READMEと照合）
- [ ] Firewallで信頼LANからのTCP 10000のみ許可（インターネット公開しない）
- [ ] 起動手順を `docs/setup-ubuntu.md` に記載

### 互換性検証
- [ ] ROS-TCP-Connector / Endpoint の現行Unity・Humble・Androidでの互換性を確認
- [ ] Endpointの購読QoSと `/scan`（Best Effort）の相性を確認
  - [ ] 合わない場合：`questbot_xr_bridge` に `rclpy` 中継ノードを実装
    - 入力 `/scan`：SensorData QoS、Best Effort、少量キュー
    - 出力 `/xr/scan`：Reliable、depth 1、最新優先、5〜10 Hz

### 模擬データ
- [ ] 模擬LaserScan publisherを作成（正面1m・左・右・後方に既知の点、NaN/Inf/0.0を含む）
- [ ] ユニットテスト：無効値（NaN、Inf、range_min未満、range_max超過、0.0）の除外

### Unity
- [ ] ROS-TCP-Connector導入、ROS 2メッセージ生成モードを選択
- [ ] 接続設定UI：IP/ポートの変更・保存・再接続（`localhost` を使わない）
- [ ] 座標変換を1か所に集約（ROSGeometry等、FLU → Unity：`(-y, z, x)`、Quaternionも対応関数）
- [ ] LaserScan → 点群変換（`angle_min + i * angle_increment`）
- [ ] 点群をMesh/Particleでまとめて描画（点ごとのGameObject生成禁止）
- [ ] 受信スレッドと描画更新を分離、Unity APIはメインスレッドのみ
- [ ] EditMode/PlayModeテスト：正面1mの点がUnity正面1m、左右が反転しない
- [ ] Mac Editor（マウス操作）で模擬形状を表示
- [ ] Quest APKをビルド・インストールし、同じ模擬形状を表示

## フェーズ1（MVP）：実機 `/scan` のVR表示

完了条件：壁までの距離・左右・前後が実機と一致し、更新が継続する

- [ ] Burgerの実 `/scan`（または `/xr/scan`）をEditorで表示
- [ ] 同じくQuest実機で表示（scan基準の実寸表示、移動履歴なし）
- [ ] UI：表示縮尺・点サイズ・高さ・再センタリング
- [ ] UI：「リアルタイムスキャン」であることを表示
- [ ] UI：接続状態表示、切断・更新停止から1秒以内に古いデータと分かる表示へ切替
- [ ] 2D点列を3D形状として誤表示していないことを確認

### 受け入れ検証（`docs/acceptance.md` に結果記録）
- [ ] `/scan` 継続受信、Endpointが対応QoSで購読
- [ ] Unity Editor・Quest APK の両方がUbuntuのLANアドレスへ接続
- [ ] 前方・左方・右方の既知の物体位置が正しく表示される
- [ ] 更新停止1秒以内に古いデータ表示へ切替
- [ ] 10分間表示でキュー・メモリが増え続けない
- [ ] Quest描画FPS（目標72 FPS以上）とセンサ更新Hzを分けて計測・記録
- [ ] README：採用バージョン、実行コマンド、テスト結果、未確認項目を記録

---

## フェーズ2：`/map` と地図内のロボット位置

完了条件：地図に対してロボットとスキャンが整合して動く

- [ ] TurtleBot3 SLAM（Cartographer等）をUbuntu PCで起動
- [ ] ゲートウェイ：`/map` をTransient Localで購読し、Unity再接続時に再送
- [ ] ゲートウェイ：`/tf_static` 保持 + `/tf` 最新と組合せ、`map → base_footprint` を解決し `/xr/robot_pose`（PoseStamped）配信
- [ ] Unity：OccupancyGrid → Texture2D（-1=灰、0付近=白、100付近=黒、中間値の表現を定義）
- [ ] Unity：サイズ（width×resolution、height×resolution）とoriginの位置・回転を反映
- [ ] Unity：スキャンを `frame_id → map` のTF（測定時刻）で地図上に表示
- [ ] `/odom` は診断用のみ（map内位置の代用にしない）
- [ ] UI：地図未受信・TF欠落・古いデータを明示
- [ ] 検証：map原点・回転・TF・再接続時の地図再送

## フェーズ3：USBカメラ映像の仮想スクリーン

完了条件：正しい向きの映像と遅延・接続状態を表示

- [ ] `/dev/video0` の形式・解像度・FPS・画像の上下を確認（`v4l2-ctl --list-formats-ext`）
- [ ] `v4l2_camera` / `usb_cam` のHumble対応を確認し一方に統一
- [ ] 640×480・10〜15 FPSで PiのCPU・Wi-Fi帯域を測定
- [ ] カメラトピック名を調査・記録
- [ ] ゲートウェイでJPEGフレーム配信（非圧縮ImageをLiDARと同じTCPで流さない）
- [ ] Unity：明示的なJPEGデコード → 仮想スクリーン表示（最新フレームのみ、単眼映像）
- [ ] 表示層で画像回転を補正
- [ ] UI：遅延・接続状態を表示
- [ ] 遅延が問題ならWebRTC方式を比較・実機検証

## フェーズ4：パススルーMRと座標合わせ

完了条件：基準点から実空間とROS地図を位置合わせできる

- [ ] Unity OpenXR Metaの対応機能を確認して導入（Meta XR SDKとの重複設定を避ける）
- [ ] パススルー背景の表示
- [ ] 手動校正UI：既知の基準位置・向きを指定（X/Z・yaw）、床高を別設定
- [ ] リセンター・追跡喪失・SLAMリセット時に校正を無効化し再校正UIを表示
- [ ] （将来）マーカー検出・空間アンカーの調査（Questカメラ権限・API確認）

## フェーズ5：低速テレオペ・Nav2ゴール（閲覧完成後）

完了条件：デッドマン・切断時停止・ゴール取消しを実機で検証

- [ ] 操作ゲートウェイ：Unity → `/xr/cmd_vel` → ゲートウェイ → `/cmd_vel`（直接送信しない）
- [ ] 速度上限：直進0.05 m/s、旋回0.3 rad/s（実測で調整）
- [ ] デッドマン押下中のみ操作
- [ ] ROS側0.3秒タイムアウト監視、切断・アプリ停止・フォーカス喪失でゼロ速度
- [ ] キーボード・Nav2・VR操作の同時指令を禁止または仲裁
- [ ] Nav2 Action Clientブリッジ：受付・進捗・取消・結果をQuestへ返す
- [ ] 停止UIを常時表示（物理非常停止の代わりにしない）
- [ ] 実機で停止・タイムアウト・指令仲裁を検証してから公開
