---
title: "【2026年版】macOS 開発者が最初にやる初期設定まとめ（defaults 編）"
emoji: "🛠"
type: "tech"
topics: ["macos", "開発環境", "terminal", "zsh", "初期設定"]
published: true
---

# macOS 開発者向け 初期設定手順（defaults 編）

この記事は、**新しいMacに買い替えた開発者**や、  
**macOSを開発向けに最適化したい人**向けの初期設定まとめです。  
「毎回同じことを手でやるのが面倒」「再現できる形で残したい」人に刺さります。

この記事でやることは次のとおりです。

- Finder / キーボード / Dock / スクショ周りを「開発者向け」に整える
- すべてターミナル（defaults）で設定し、再現性を持たせる
- 設定が入ったか `defaults read` で検証できるようにする

## TL;DR（先に結論）
- Finder は「リスト表示 + パス/ステータスバー + 隠しファイル表示」が最速
- キーリピート高速化と長押し無効化は体感が段違い
- Dock は自動で隠す + 高速化で作業領域が増える
- スクショは保存先を固定して散らばりを防ぐ

## 前提
- macOS 27.0（この環境で設定値の反映を `defaults read` で確認済み）
- ターミナル（zsh）

---

## 0. まとめて入れたい人向け（コピペ用）
必要な人だけ実行してください。**実行前に中身を確認してください。** 細かく見たい人は次のセクションへ。

```bash
# Finder
defaults write com.apple.finder AppleShowAllFiles -bool true
defaults write -g AppleShowAllExtensions -bool true
defaults write com.apple.finder ShowPathbar -bool true
defaults write com.apple.finder ShowStatusBar -bool true
defaults write com.apple.finder _FXSortFoldersFirst -bool true
defaults write com.apple.finder FXPreferredViewStyle -string "Nlsv"
killall Finder

# Keyboard
defaults write -g InitialKeyRepeat -int 15
defaults write -g KeyRepeat -int 2
defaults write -g ApplePressAndHoldEnabled -bool false

# Dock
defaults write com.apple.dock autohide -bool true
defaults write com.apple.dock autohide-delay -float 0
defaults write com.apple.dock autohide-time-modifier -float 0.15
killall Dock

# Screenshot
mkdir -p ~/Desktop/Screenshots
defaults write com.apple.screencapture location ~/Desktop/Screenshots
defaults write com.apple.screencapture name "screenshot"
defaults write com.apple.screencapture disable-shadow -bool true
killall SystemUIServer

# UI
defaults write NSGlobalDomain AppleShowScrollBars -string "Always"
defaults write NSGlobalDomain AppleKeyboardUIMode -int 3

# Text input
defaults write -g NSAutomaticCapitalizationEnabled -bool false
defaults write -g NSAutomaticQuoteSubstitutionEnabled -bool false
defaults write -g NSAutomaticDashSubstitutionEnabled -bool false

# Network
defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true
```

※ 反映には再ログインが必要な項目があります（後述）。

## 1. Finder・ファイル管理系

### 隠しファイルを表示
```bash
defaults write com.apple.finder AppleShowAllFiles -bool true
killall Finder
```
※ 不要になったら Finder 上で `Cmd + Shift + .` を押すと、表示と非表示を切り替えられます。

### 拡張子を常に表示（事故防止）
```bash
defaults write -g AppleShowAllExtensions -bool true
```

### パスバーを表示
```bash
defaults write com.apple.finder ShowPathbar -bool true
killall Finder
```

### ステータスバーを表示
```bash
defaults write com.apple.finder ShowStatusBar -bool true
killall Finder
```

### フォルダを先に表示（node_modules が下に行く）
```bash
defaults write com.apple.finder _FXSortFoldersFirst -bool true
killall Finder
```

### Finder のデフォルト表示をリスト表示に
```bash
defaults write com.apple.finder FXPreferredViewStyle -string "Nlsv"
killall Finder
```

---

## 2. キーボード・入力系（重要）

### キーリピートを高速化
```bash
defaults write -g InitialKeyRepeat -int 15
defaults write -g KeyRepeat -int 2
```
※ ログアウト → ログイン後に反映

### 長押しアクセント入力を無効化（aaaa 連打できる）
```bash
defaults write -g ApplePressAndHoldEnabled -bool false
```
※ ログアウト → ログイン後に反映  
※ 長押しで `é` などのアクセント文字を入力できなくなります。必要な場合は `defaults delete -g ApplePressAndHoldEnabled` で戻せます。

---

## 3. Dock・UI

### Dock を自動的に隠す
```bash
defaults write com.apple.dock autohide -bool true
killall Dock
```

### Dock の表示・非表示を高速化
```bash
defaults write com.apple.dock autohide-delay -float 0
defaults write com.apple.dock autohide-time-modifier -float 0.15
killall Dock
```

---

## 4. スクリーンショット設定

### 保存先を Desktop/Screenshots に変更
```bash
mkdir -p ~/Desktop/Screenshots
defaults write com.apple.screencapture location ~/Desktop/Screenshots
killall SystemUIServer
```
※ iCloud の「デスクトップと書類フォルダ」を有効にしている場合、スクリーンショットも iCloud に同期されます。同期したくない場合は `~/Pictures/Screenshots` などに変更してください。

### ファイル名をシンプルに
```bash
defaults write com.apple.screencapture name "screenshot"
killall SystemUIServer
```

### スクリーンショットの影を無効化（UI確認用）
```bash
defaults write com.apple.screencapture disable-shadow -bool true
killall SystemUIServer
```

※ ⌘ + Shift + 5 → オプション → 保存先  
（UI 設定が優先される場合があるため要確認）

---

## 5. UI・視認性

### スクロールバーを常に表示
```bash
defaults write NSGlobalDomain AppleShowScrollBars -string "Always"
```

### フルキーアクセス有効（Tab でボタン移動）
```bash
defaults write NSGlobalDomain AppleKeyboardUIMode -int 3
```

---

## 6. 入力・テキスト編集系

コードやコマンドをメモアプリなどに貼ったとき、記号が勝手に変換されるのを防ぎます。

### 自動大文字化を無効化
```bash
defaults write -g NSAutomaticCapitalizationEnabled -bool false
```

### スマート引用符を無効化（`"` が `“ ”` に変わらない）
```bash
defaults write -g NSAutomaticQuoteSubstitutionEnabled -bool false
```

### スマートダッシュを無効化（`--` が `—` に変わらない）
```bash
defaults write -g NSAutomaticDashSubstitutionEnabled -bool false
```

※ 開いているアプリには、再起動またはログアウト → ログイン後に反映されます。

---

## 7. ネットワーク

### ネットワーク上に .DS_Store を作らせない
```bash
defaults write com.apple.desktopservices DSDontWriteNetworkStores -bool true
```

---

## 8. 設定確認コマンド

```bash
defaults read com.apple.finder AppleShowAllFiles
defaults read -g AppleShowAllExtensions
defaults read com.apple.finder ShowPathbar
defaults read com.apple.finder ShowStatusBar
defaults read com.apple.finder _FXSortFoldersFirst
defaults read com.apple.finder FXPreferredViewStyle
defaults read -g InitialKeyRepeat
defaults read -g KeyRepeat
defaults read -g ApplePressAndHoldEnabled
defaults read com.apple.dock autohide
defaults read com.apple.dock autohide-delay
defaults read com.apple.dock autohide-time-modifier
defaults read com.apple.screencapture location
defaults read com.apple.screencapture name
defaults read com.apple.screencapture disable-shadow
defaults read NSGlobalDomain AppleShowScrollBars
defaults read NSGlobalDomain AppleKeyboardUIMode
defaults read -g NSAutomaticCapitalizationEnabled
defaults read -g NSAutomaticQuoteSubstitutionEnabled
defaults read -g NSAutomaticDashSubstitutionEnabled
defaults read com.apple.desktopservices DSDontWriteNetworkStores
```

## 9. 反映タイミングの注意
- Finder / Dock / SystemUIServer は `killall` で即時反映
- キーボード系は**ログアウト → ログイン後**に反映されることが多い
- 「反映されない」ときは、再起動すると確実です

---

## 10. 元に戻したい場合

```bash
defaults delete <domain> <key>
```

例：
```bash
defaults delete com.apple.finder AppleShowAllFiles
killall Finder
```

---

## 今後について

今後は、以下の内容も順次まとめていく予定です。

- iTerm2 / zsh / starship の最小構成
- Node / Python のバージョン管理（mise を使った実務向け構成）
- 新しいMac用のセットアップスクリプト（1コマンドで完了）

実際に自分の環境で使っている設定を元に、引き続きまとめていきます。
続編を出したらこの記事にも追記するので、必要ならブックマークしておいてください。
