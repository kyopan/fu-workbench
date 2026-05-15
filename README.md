# FU Workbench

Web Serial API ベースのブラウザ単体動作 ワークベンチ。
**Telemetry visualizer + Firmware flasher + 各種コマンド送信**を一画面に統合。

公開 URL: https://kyopan.github.io/fu-workbench/

---

## 動作要件

| 項目 | 要件 |
|---|---|
| ブラウザ | **Chromium 系** (Chrome / Edge / Arc / Brave / Opera) |
| 接続方法 | **HTTPS** または **`http://localhost`** (Web Serial API の必須条件) |
| OS | Windows / macOS / Linux / ChromeOS |
| 対応 FW (Visualizer) | Unit1 v2.3.29+ / Unit2 v2.2.97+ |
| 対応 FW (Install) | 既に PIO で初期化済の ESP32-S3 機体 (app partition 書き換えのみ) |

Safari / Firefox は Web Serial API 非対応のため動作しません。

---

## 機能

### Telemetry Visualizer
- BNO055 quaternion を 3D 表示 (Three.js)
- GridEye 重心
- Heading コンパス
- Telemetry 数値 (qw/qx/qy/qz, cx/cy, spY/spP)
- Serial log + 任意コマンド送信

### Mode / 設定 (Unit1)
- Tracking ON/OFF (POS,90 / POS,0)
- Heading mode / Winch mode 切替
- LED Control (RGB sliders + presets)
- USB Debug Mode (Winch 未接続時に CAL_READY 強制)

### Motor (Unit2 forwarded)
- Pitch ±60°
- Yaw ±20 rad/s
- Yaw STOP
- Horizon Cal / Pitch Cal クリア / Yaw Cal

### Firmware Install (Web Serial Flash)
- Winch / Unit1 / Unit2 各 firmware を S3 上の最新版から直接書き込み
- bootloader / partition は既存維持 (app 書き換えのみ、出荷済み機体の復旧用途)
- esp-web-tools (esptool.js) 経由、Chromium + USB ケーブルだけで完結

---

## 使い方

### Telemetry / 操作
1. ブラウザで URL を開く
2. デバイスを USB 接続 (起動 17 秒待つ)
3. **Connect** ボタン → ポート選択ダイアログから対象を選ぶ
4. 自動で Unit1 / Unit2 を判別 → 該当パネルが有効化

### Firmware Install (USB Serial Flash)
1. 既に Connect 中なら **Disconnect** してください (Web Serial port は 1 セッションしか持てない)
2. 対象 firmware の **⚡ Install** ボタンを押す
3. ポート選択ダイアログから対象 USB を選ぶ
4. esp-web-tools のダイアログで **Install** をクリック
5. 30-60 秒で書き込み完了 → デバイス自動再起動
6. その後 **Connect** で telemetry に戻れる

注意: bootloader / partition table を含む完全初期化が必要な場合 (新品 ESP32-S3) は本機能では足りません。PlatformIO 直書きが必要です。

---

## ローカル動作確認 (開発用)

`file://` では Web Serial API が動作しないため、HTTP サーバーが必要。

```bash
cd web_visualizer
python3 -m http.server 8080
# ブラウザで http://localhost:8080 を開く
```

S3 CORS が `http://localhost:8765` と `http://localhost:5050` を許可しているため、本番 firmware bin のダウンロードもローカルで動作確認可能。

---

## ファイル構成

| ファイル | 役割 |
|---|---|
| `index.html` | 単一ファイルアプリ (HTML + inline CSS/JS) |
| `.nojekyll` | GitHub Pages の Jekyll 処理スキップ用 (空ファイル) |
| `manifest_winch.json` | esp-web-tools manifest (Winch v2) |
| `manifest_unit1.json` | esp-web-tools manifest (Tube Unit1) |
| `manifest_unit2.json` | esp-web-tools manifest (Tube Unit2) |
| `README.md` | 本書 |

依存はすべて CDN ロード (Three.js / esp-web-tools)。リポジトリにバイナリは持たない。

---

## アーキテクチャ

```
[Browser]
  ├─ Web Serial API ──── USB ────→ [ESP32-S3]
  │                                 (Telemetry / Command)
  └─ esptool.js ──── USB (DL mode) → [ESP32-S3]
                                      (Flash write)

[Browser] ──── HTTPS ──→ [S3: comone-fragmented-unity-fw]
                              ├─ winch_v2/winch_v2_product.bin
                              ├─ tube_v2_unit1_product.bin
                              └─ tube_v2_unit2_product.bin
```

S3 bucket は ESP32 OTA でも使われており、Web Serial Install は **OTA と同じ binary を別経路 (USB) で書き込む**形になる。

S3 CORS 設定: `https://kyopan.github.io`, `http://localhost:8765`, `http://localhost:5050` からの GET/HEAD 許可済。

---

## トラブル

### Connect ボタン押下時に「No device selected」「ポートが出ない」
- USB ケーブルが「データ通信対応」か確認 (充電専用ケーブルでは Web Serial が反応しない)
- Windows の場合、デバイスマネージャー → ポート (COM と LPT) で `USB シリアル デバイス (COM*)` を確認
- macOS の場合、`ls /dev/cu.usbmodem*` で認識確認

### ページを開いても Connect ボタンが grey out
- Safari / Firefox を使っている → Chromium 系へ
- HTTPS or localhost でない (例: `file://`) → ローカルでは `python3 -m http.server` 経由で開く

### 接続したのに telemetry が来ない / `Detecting…` のまま
- 起動から 17 秒以内 (BOOT_CAL シーケンス中) → 待つ
- Unit2 を接続している場合、Unit2 は production FW では silent → 6 秒タイムアウト後 `UNIT 2` 表示で正常

### Install で「Failed to connect to ESP32-S3」
- 対象が既に Web Serial で Connect 中 → Disconnect してから再試行
- USB ケーブルが充電専用 → データ通信対応に交換
- Native USB-CDC (XIAO ESP32-S3) で boot mode に入らない → 一旦 USB 抜き差し、もしくは BOOT ボタン押下後リトライ

### LED が光らない
- `HEAD,0` を先に送信 (`HEAD,1` 中は RGB override 無効)
- FW 側で 70% にスケールされる

---

## デプロイ (本リポジトリの GitHub Pages 設定)

```bash
gh repo create kyopan/fu-workbench --public --source=. --push
gh repo edit kyopan/fu-workbench --enable-pages-source main:/
```

Settings → Pages → Source: `main` branch / root が選ばれていれば OK。
push 後 1-2 分で https://kyopan.github.io/fu-workbench/ で公開される。

---

## 既知の制限

| 項目 | 状態 |
|---|---|
| Firefox / Safari 対応 | 不可 (Web Serial API 仕様で除外) |
| 新品 ESP32-S3 の完全初期化 | 不可 (bootloader/partition が必要、PIO 直書きで対応) |
| 複数デバイス同時 telemetry | 不可 (Web Serial port は 1 タブ 1 ポート) |
| Mobile (iOS Safari / Android Chrome) | Web Serial 非対応端末あり |
| Three.js / esp-web-tools の CDN 切替 | 起こると一時的に動作しない、版固定で対応中 |
