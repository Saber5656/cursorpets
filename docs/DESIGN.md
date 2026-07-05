# cursorpets 設計書 (v1)

- Repository: `github.com/Saber5656/cursorpets`
- Tagline: "A modern take on classic oneko-style pets that chase your cursor."
- License: MIT（前提）/ クラウド送信なし / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存クローンとの位置づけ

cursorpets は、マウスカーソルを追いかける小さなキャラクターをデスクトップに常駐させるジョークソフトである。
1989 年の oneko（X11 用、パブリックドメイン）へのノスタルジーを核に、
「メニューバーから 1 クリックで飼える」「Apple Silicon ネイティブで CPU をほぼ食わない」「Gatekeeper に怒られない」
という現代 macOS の作法に合わせて再構築する。機能を盛らず、"入れて 10 秒で笑える" 体験だけに集中する。

**既存クローンとの差別化（車輪の再発明にならない位置づけ）**

| 既存 | 動作範囲 | cursorpets との違い |
|---|---|---|
| oneko.js（Web で人気の実装） | ブラウザの 1 ページ内のみ | cursorpets は **OS デスクトップ全体** で動く。ページを離れても猫は付いてくる |
| オリジナル oneko / oneko-sakura | X11。macOS では XQuartz が必要 | ネイティブ macOS アプリ。インストールは brew 1 行 |
| RunCat 等のメニューバー系 | メニューバー内のみ | 画面全域をキャラが走る。メニューバーは制御 UI に徹する |

つまり「oneko.js のデスクトップ版を、非開発者でも入れられる形で出す」ことが存在理由。
機能で勝負せず、**導入の軽さと動作の軽さ** で差別化する。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 |
|---|---|---|
| 入れる | カーソル追跡ペット 1 体（猫） | idle / alert / run(8 方向) / sleep のアニメ |
| 入れる | メニューバー常駐 UI | Start/Pause、Speed(2 段階)、Launch at Login、Quit |
| 入れる | クリックスルー・常時最前面・全 Space 追従 | 作業の邪魔を絶対にしない |
| 入れる | マルチディスプレイ追従（後述の小窓方式で自然に対応） | 特別な処理を足さない範囲で |
| 入れる | 自作 or CC0 の新規スプライト 1 種 | 原作スプライトは同梱しない |
| 入れる | Homebrew Cask + GitHub Releases 配布、署名 + notarization | 導入 1 行 |
| 入れない | 複数ペット同時表示 | 状態機械を配列化すれば後付け可能。v1 では非対応 |
| 入れない | スキン/キャラの外部追加・スキン市場 | スプライトシート仕様だけ固定し、v2 の口を残す |
| 入れない | 設定画面（ウィンドウ） | メニューバーのメニュー項目だけで完結 |
| 入れない | 自動更新（Sparkle） | brew upgrade / 手動 DL で十分。依存を増やさない |
| 入れない | Windows / Linux 対応 | §3 参照 |
| 入れない | クリック時のリアクション等のインタラクション | クリックスルー方針と矛盾するため |
| 入れない | 効果音 | ジョークソフトとして音は事故のもと |

---

## 3. 対応プラットフォームと優先順位

| 優先度 | OS | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | macOS 13+ (Apple Silicon / Intel Universal) | 対応 | 開発者の環境。検証・署名・配布まで一人で完結できる |
| 2 (v2 候補) | Windows | 保留 | 需要は最大だが、layered window / WinAPI の検証コストが v1 の「早く出す」に反する |
| 3 | Linux | 見送り | X11 には原作 oneko が現存、Wayland はグローバル座標取得と最前面制御が原理的に困難 |

macOS 特化を先に決めることで、技術選定（§4）でネイティブ一択にでき、実装量が最小になる。
クロスプラットフォームは「移植したい人の PR を歓迎する」と README に書くに留める。

---

## 4. 技術選定

### 比較

| 候補 | バイナリ/メモリ | 透過・クリックスルー | 開発コスト | 判定 |
|---|---|---|---|---|
| **Swift + AppKit（採用）** | 数 MB / 10–30MB 級 | `NSWindow.ignoresMouseEvents` で公式サポート。挙動が安定 | macOS のみなら最小。API 直叩き | ✅ |
| Tauri v2 (Rust + WebView) | ~10MB / 30–50MB | 透過窓の「透明部だけクリックスルー」は未解決の feature request（tauri#13070）。窓全体の ignore は可 | Rust + WebView の 2 層デバッグ | ❌ 過剰 |
| Electron | 80–150MB / 150–300MB | 可能だが Electron 7+ に macOS クリックスルーの既知トラブルがあり workaround が必要（loomhq/ElectronMacOSClickThrough） | 低いが、猫 1 匹に Chromium 同梱は本末転倒 | ❌ |

Tauri/Electron の利点（Web UI・クロスプラットフォーム）は本プロダクトでは使い道がない。
描画は 32px のスプライト 1 枚であり、HTML レンダラは不要。**Swift + AppKit ネイティブ一択**とする。

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | Swift 5.9+ | 標準。署名・notarization ツールチェーンと親和 |
| 常駐/メニュー | `NSStatusItem`（AppKit）+ `LSUIElement = YES` | Dock アイコンなしの純メニューバー常駐 |
| ペット表示 | ボーダーレス `NSWindow` + `CALayer.contents` によるスプライト表示 | SpriteKit すら過剰。レイヤーに CGImage を貼るだけ |
| ループ | `Timer`（可変インターバル、§5・§6 参照） | CVDisplayLink は 60fps 前提になり消費電力で不利 |
| マウス座標 | `NSEvent.mouseLocation` のポーリング | **権限不要**（下記） |
| 自動起動 | `SMAppService.mainApp`（macOS 13+） | Launch at Login を数行で実装 |
| 依存ライブラリ | なし | ゼロ依存を売りにする |

### グローバルマウス座標と権限（調査結果）

- `NSEvent.mouseLocation` はスクリーン座標のカーソル位置を返し、**Accessibility 権限不要**。タイマーで読むだけでよい。
- `NSEvent.addGlobalMonitorForEvents(matching: .mouseMoved)` も**マウス移動イベントに限れば権限不要**（キーボード監視は要 Accessibility）。
- `CGEventTap`（HID レベル）は Accessibility 信頼または root が必要 → **不採用**。
- v1 は実装が最も単純な「アニメ tick 内で `NSEvent.mouseLocation` を読む」方式とする。イベント駆動化は省電力の実測次第で検討。

**結論: cursorpets は TCC 権限ダイアログを一切出さずに動く。** これは非開発者向けの導入体験として決定的に重要。

### オーバーレイウィンドウ方式（重要な設計判断）

画面全体を覆う透明ウィンドウ**ではなく**、ペットサイズ（例 64×64pt）の小さなボーダーレス窓を
`setFrameOrigin` で毎 tick 動かす「**小窓追跡方式**」を採用する。

- 全画面透過窓はコンポジタ負荷・スクリーンショット写り込み・マルチディスプレイで窓を複数管理する問題を抱える
- 小窓方式なら **グローバル座標がディスプレイをまたいで連続しているため、マルチディスプレイ対応が実質タダ**で手に入る
- 窓設定: `styleMask = .borderless`, `isOpaque = false`, `backgroundColor = .clear`, `hasShadow = false`,
  `ignoresMouseEvents = true`, `level = .statusBar`（全画面アプリ上は追わない。割り切り）,
  `collectionBehavior = [.canJoinAllSpaces, .stationary, .ignoresCycle]`（全 Space 追従、Mission Control で無視）

---

## 5. アーキテクチャ

主要コンポーネントは 4 つ。1 プロセス・1 スレッド（main）で完結する。

```
┌─────────────────────────────────────────────────────┐
│ CursorPetsApp (NSApplicationDelegate, LSUIElement)  │
│                                                     │
│  ┌──────────────┐    設定変更     ┌───────────────┐ │
│  │ StatusBar     │──────────────▶│ PetEngine      │ │
│  │ Controller    │  start/pause  │ (Timer tick)   │ │
│  │ (NSStatusItem)│  speed        │                │ │
│  └──────────────┘               │ 1. mouseLocation│ │
│                                  │ 2. StateMachine │ │
│  ┌──────────────┐   frame番号    │ 3. 位置積分     │ │
│  │ SpriteSheet   │◀─────────────│ 4. tick間隔調整 │ │
│  │ (CGImage切出し │               └──────┬────────┘ │
│  │  キャッシュ)   │                      │ 位置+画像  │
│  └──────────────┘               ┌──────▼────────┐ │
│                                  │ PetWindow      │ │
│                                  │ (borderless    │ │
│                                  │  NSWindow +    │ │
│                                  │  CALayer)      │ │
│                                  └───────────────┘ │
└─────────────────────────────────────────────────────┘
```

### 処理フロー（1 tick）

1. `NSEvent.mouseLocation` を取得し、ペット現在位置との距離 d を計算
2. StateMachine を更新（下表）
3. run 中なら速度 v でカーソル方向へ移動（8 方向に量子化してスプライト選択）
4. `CALayer.contents` に該当フレームの CGImage をセットし、`setFrameOrigin` で窓を移動
5. 状態に応じて次回 tick 間隔を再設定（§CPU 対策）

### アニメ状態機械

| 状態 | 遷移条件 | アニメ | tick 間隔 |
|---|---|---|---|
| `idle` | d < 停止半径（例 24pt） | 座り 1–2 フレーム | 250ms |
| `alert` | idle/sleep 中に d > 追跡開始半径 | 驚き 1 フレーム（1 tick だけ） | 100ms |
| `run` | alert 後 / 追跡中 d > 停止半径 | 8 方向 × 2 フレーム交互 | 100ms (10fps) |
| `sleep` | idle が 10 秒継続 | 寝息 2 フレーム | 1000ms |
| `groom`（余裕があれば） | idle が 3 秒継続、確率遷移 | 毛づくろい 2 フレーム | 250ms |

### スプライトシート仕様（v2 のスキン追加を見据えて固定）

- PNG 1 枚、透過背景、**32×32px セルのグリッド**（Retina 用に @2x = 64px 実寸で作成し 32pt 表示）
- 行 = 状態、列 = フレーム。マッピングは `pet.json`（状態名 → 行/列/フレーム数/ループ有無）で宣言
- v1 は猫 1 種をアプリバンドルに同梱。`pet.json` 形式を README に書いておけば、v2 で「フォルダに置くだけスキン」に拡張できる
- 素材は**自作またはコミッション（CC0/MIT で公開）**。原作 oneko の BMP は「public domain とされる」が一次証跡が弱いため同梱しない（§9 リスク）

### CPU を極小にする工夫

- 描画は「窓の移動 + レイヤー画像差し替え」のみ。毎フレームの再レンダリングなし
- 状態別 tick 間隔（上表）: 追跡中でも 10fps（oneko 原作準拠のカクカク感はむしろ味）
- sleep 中は 1fps、かつカーソル座標が前回と同一なら描画・移動をスキップ
- スプライトは起動時に全フレーム CGImage 化してキャッシュ。tick 内でのデコードなし
- 目標: 追跡中 CPU < 2%、sleep 中 ~0%（Activity Monitor 実測を README に載せてネタにする）

---

## 6. UI/UX

### 常駐 UI（メニューバーのみ、設定ウィンドウなし）

```
🐈 (NSStatusItem)
├─ Pause / Resume          ← ペットの一時停止
├─ Speed  ▸ Normal / Zoomies
├─ ───────────────
├─ Launch at Login  [✓]    ← SMAppService
├─ About cursorpets…       ← バージョン + GitHub リンク
└─ Quit
```

- 設定の永続化は `UserDefaults` の 3 キー（paused / speed / 初回起動済み）だけ
- Dock アイコンなし（`LSUIElement`）。「アプリを起動した感」を消し、猫が住み着いた感を出す

### 初回起動体験（勝負どころ）

1. 起動 → **権限ダイアログなし**で即座に猫が画面中央に出現し、カーソルへ走ってくる（ここまで 3 秒）
2. 同時にメニューバーに猫アイコンが点灯。初回のみ小さな吹き出し風 popover で
   「Your pet lives in the menu bar → 🐈」と 1 文だけ表示（チュートリアルはこれで終わり）
3. 終了方法が分からない事故を防ぐため、About とメニューに Quit を明示

### エッジケースの割り切り（v1）

- 全画面アプリ（ゲーム・動画）の上には出ない（`.statusBar` レベルの仕様として許容）
- スクリーンショット・画面共有には写る（それも含めてネタ。README に明記）
- カーソルがペット真上に来たら idle のまま（捕まえた判定などは作らない）

---

## 7. 配布方法

| 項目 | v1 の方針 | 理由 |
|---|---|---|
| 一次配布 | GitHub Releases に Universal 2 の `.zip`（.app 同梱） | dmg 作成の手間より tag push → 自動リリースを優先 |
| Homebrew | 自前 tap: `brew install --cask Saber5656/tap/cursorpets` | 本家 cask リポジトリは知名度要件があるため、まず tap。README の 1 行インストールに使う |
| 署名 | Developer ID Application 証明書で codesign（要 Apple Developer Program $99/年） | 未署名だと macOS 15+ では右クリック開封すら不可になり、非開発者に配れない |
| Notarization | `notarytool` + staple を CI（GitHub Actions）に組み込み | Gatekeeper 警告ゼロが「非開発者に面白がってもらう」の必要条件 |
| 署名なし fallback | Developer Program 未加入の間は README に `xattr -dr com.apple.quarantine` 手順を明記 | v0.x の開発者向け配布用。**v1.0 公開の前提条件は署名+公証**とする |
| 自動更新 | **なし**。メニューの About に「Check for Updates (GitHub)」リンクのみ | Sparkle は依存とサーバ管理を増やす。brew upgrade で十分 |
| サンドボックス | App Sandbox なし（Mac App Store 非対応と割り切り） | 配布は GitHub + brew に限定。MAS 審査コストを回避 |

---

## 8. README 構成案（英語）

```markdown
<バナー画像: ドット絵の猫がロゴを追いかける横長 PNG>

# cursorpets 🐈
> A modern take on classic oneko-style pets that chase your cursor.

<デモ GIF: 実デスクトップで猫がカーソルを追い、寝るまでの 8 秒ループ>

## Install
brew install --cask Saber5656/tap/cursorpets
(or grab the .app from Releases)

## What it does
- 3 行の箇条書き: chases / sleeps / stays out of your way (click-through)
- "Zero permissions. Zero network. ~0% CPU while sleeping."

## Why another oneko?
- oneko.js は Web ページ内、これはデスクトップ全体、の 2 文

## Controls
- メニューバー項目の表（Pause / Speed / Launch at Login / Quit）

## Make your own pet (soon)
- スプライトシート仕様 (32px grid + pet.json) への短い言及と CONTRIBUTING リンク

## Credits & License
- MIT。スプライトのライセンス明記（CC0/自作）。原作 oneko (1989) へのリスペクト表明
```

ポイント: デモ GIF を最上部に置き、インストールコマンドまでスクロールなしで到達できること。
バッジは license / release / downloads の 3 つまで。

---

## 9. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| P0 | 小窓の高頻度 `setFrameOrigin` の負荷と残像 | 10fps で窓を動かした際の WindowServer CPU とティアリング | 100 行のプロトタイプを最初に書く（Issue #1） |
| P0 | `NSEvent.mouseLocation` の権限不要性の実機確認 | 調査上は不要だが、macOS 15/26 実機で TCC ダイアログが出ないこと・座標系(左下原点)とマルチディスプレイ負座標の扱いを確認 | 同プロトタイプ |
| P1 | 署名・notarization パイプライン | Developer Program 加入、CI からの notarytool 実行、staple 後の初回起動確認 | クリーンな Mac (または新規ユーザ) で DL→起動 |
| P1 | スプライトの権利 | 原作スプライトは同梱しない方針の堅持。新規素材のライセンス表記（CC0 か、作者クレジット付き MIT か）を作成時に確定 | 素材完成時にレビュー |
| P2 | Space 切替・全画面・スクリーンセーバ復帰時の窓挙動 | `.canJoinAllSpaces` でも稀に窓が消える報告があるため、復帰時に orderFrontRegardless する保険を検討 | 手動テスト表を作る |
| P2 | 低電力モード・ProMotion 環境での Timer 精度 | tick 間隔が伸びても破綻しない設計か（位置積分を Δt ベースにする） | Instruments で確認 |
| P3 | 名前空間の衝突 | "cursorpets" と類似の既存プロダクト・商標の簡易確認 | リリース前に検索 |

**最重要リスク**: P0 の 2 件。ここが崩れると方式変更（全画面透過窓 or イベント駆動）になるため、
本実装前に捨てられるプロトタイプで必ず確認する。

---

## 10. v1 Issue 分割案（8 個）

- **#1 `Spike: prototype click-through pet window and cursor polling`** — ラベル: `design`, `spike`
  ボーダーレス透過 NSWindow + `ignoresMouseEvents` + `NSEvent.mouseLocation` ポーリングの 100 行プロトタイプを作り、P0 リスク（窓移動負荷、権限ダイアログ非表示、マルチディスプレイ座標）を検証する。
  受け入れ条件: 四角い矩形がカーソルを 10fps で追い、TCC ダイアログが出ず、追跡中 CPU < 5% を Activity Monitor で確認。結果を Issue コメントに記録。

- **#2 `Set up app skeleton with menu bar item and launch-at-login`** — ラベル: `enhancement`
  `LSUIElement` の Swift アプリ雛形、NSStatusItem メニュー（Pause/Speed/Launch at Login/About/Quit）、UserDefaults 永続化、SMAppService 連携を実装する。
  受け入れ条件: Dock 非表示で常駐し、全メニュー項目が機能し、再起動後も設定が保持される。

- **#3 `Implement pet engine with state machine and variable tick`** — ラベル: `enhancement`
  idle/alert/run/sleep の状態機械、8 方向量子化、Δt ベースの位置積分、状態別 tick 間隔（run 100ms / idle 250ms / sleep 1000ms）を実装する。スプライト完成前はプレースホルダ矩形で動かす。
  受け入れ条件: 状態遷移が §5 の表どおり動き、sleep 時 CPU がほぼ 0% で、カーソル静止中に描画がスキップされる。

- **#4 `Create original cat spritesheet and pet.json format`** — ラベル: `design`, `assets`
  32×32 グリッド（@2x）の新規猫スプライト（idle/alert/run×8 方向/sleep）を自作し、状態→フレームのマッピングを `pet.json` として定義する。ライセンス（CC0 推奨）を LICENSE-assets に明記する。
  受け入れ条件: 全状態のフレームが揃い、原作 oneko 由来の画像を一切含まず、ライセンス表記が完備。

- **#5 `Render sprites and integrate with pet engine`** — ラベル: `enhancement`
  スプライトシートの起動時スライス・CGImage キャッシュ、CALayer への表示、状態機械との結線、Retina 対応を実装する。
  受け入れ条件: 実スプライトで追跡→停止→睡眠の一連が滑らかに動き、追跡中 CPU < 2%。

- **#6 `Add first-run experience and edge-case handling`** — ラベル: `enhancement`, `ux`
  初回起動 popover（メニューバー案内 1 文）、Space 切替・ディスプレイ構成変更・スリープ復帰時の窓再表示の保険処理を実装する。
  受け入れ条件: 初回のみ案内が出ること、Space 切替とディスプレイ抜き差し後もペットが表示され続けること。

- **#7 `Set up signed and notarized release pipeline with Homebrew tap`** — ラベル: `infra`
  GitHub Actions で tag push 時に Universal 2 ビルド → codesign → notarytool → staple → Releases に zip 添付。`Saber5656/homebrew-tap` に cask を追加する。
  受け入れ条件: クリーン環境で `brew install --cask` から警告なしで起動できる。

- **#8 `Write README with banner, demo GIF, and one-line install`** — ラベル: `docs`
  §8 の構成で英語 README を作成。バナー画像とデモ GIF（8 秒ループ）を撮影・作成し、oneko.js との違いとゼロ権限・ゼロネットワークを明記する。
  受け入れ条件: GIF とインストールコマンドがファーストビューに収まり、ライセンス/クレジット節がある。

推奨着手順: #1 → (#2, #4 並行) → #3 → #5 → #6 → (#7, #8 並行)。

---

## 参考資料（技術検証の根拠）

- クリックスルー/透過窓: [Apple Developer Forums — transparent window click-through](https://developer.apple.com/forums/thread/737584), [tauri-apps/tauri#13070（透明部クリックスルーは未解決 FR）](https://github.com/tauri-apps/tauri/issues/13070), [loomhq/ElectronMacOSClickThrough（Electron の既知問題の workaround）](https://github.com/loomhq/ElectronMacOSClickThrough), [Tauri Window Customization](https://v2.tauri.app/learn/window-customization/)
- マウス座標と権限: [NSEvent.mouseLocation — Apple Developer Documentation](https://developer.apple.com/documentation/appkit/nsevent/1533380-mouselocation), [CGEventTap と Accessibility 要件 — Apple Developer Forums](https://developer.apple.com/forums/thread/92761)
- oneko の権利状況: [glreno/oneko（現行メンテ版）](https://glreno.github.io/oneko/), [tie/oneko（oneko-sakura ミラー）](https://github.com/tie/oneko), [GitHub topic: oneko（クローン一覧）](https://github.com/topics/oneko)
- フレームワーク比較: [OpenReplay — Comparing Electron and Tauri](https://blog.openreplay.com/comparing-electron-tauri-desktop-applications/), [Tauri vs Electron 2026 benchmarks](https://tech-insider.org/tauri-vs-electron-2026/)
