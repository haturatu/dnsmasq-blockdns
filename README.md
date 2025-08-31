# dnsmasq-blockdns

このリポジトリは、StevenBlackのホストリストを使用してdnsmasq用のadblockするためのconfigファイルを自動生成するGitHub Actionsワークフローです。

## 機能

-  **自動更新**: 週1回、上流のホストリストをチェックして更新
-  **即座の反映**: `userconf`ファイル変更時の自動再生成
-  **統計表示**: ブロック対象ドメイン数の自動カウント
-  **差分検出**: ハッシュベースの変更検出で無駄な更新を回避

## 生成されるファイル

### `adblock.conf`
dnsmasq用のコンフィギュレーションファイル。以下の形式でドメインブロックルールを記述：

```
address=/.example.com/0.0.0.0
address=/.ads.google.com/0.0.0.0
address=/.doubleclick.net/0.0.0.0
```

### `hosts_current`
前回取得したホストファイルのコピー（差分検出用）

## 使用方法

### 1. 初期セットアップ

1. このリポジトリをフォークまたはクローン
2. `.github/workflows/update-adblock.yml`ファイルが存在することを確認
3. 必要に応じて`userconf`ファイルを作成（カスタムルール用）

### 2. カスタムルールの追加

独自のブロックルールを追加したい場合は、`userconf`ファイルを作成・編集：

```bash
# userconfファイルの例
address=/.custom-blocked-site.com/0.0.0.0
address=/.another-blocked-domain.net/0.0.0.0
```

ファイルを保存してmainブランチにpushすると、自動的にワークフローが実行されます。

### 3. dnsmasqでの使用

生成された`adblock.conf`をdnsmasqサーバーで使用：

```bash
# dnsmasq設定ファイル（例: /etc/dnsmasq.conf）に追記
conf-file=/path/to/adblock.conf

# dnsmasqサービスの再起動
sudo systemctl restart dnsmasq
```

FreeBSDの場合
```bash
# /usr/local/etc/dnsmasq.d/に配置
sudo cp adblock.conf /usr/local/etc/dnsmasq.d/
sudo service dnsmasq restart

# doasの場合
doas cp adblock.conf /usr/local/etc/dnsmasq.d/
doas service dnsmasq restart
```

## ワークフローのトリガー

このワークフローは以下の条件で自動実行されます：

### 定期実行（週次）
- **タイミング**: 毎週日曜日 日本時間 11:00 (UTC 02:00)
- **動作**: 上流ホストファイルの変更をチェック
- **更新条件**: ファイルハッシュが変更された場合のみ

###  userconf変更時
- **トリガー**: `userconf`ファイルがmainブランチにpush
- **動作**: 即座にワークフローを実行
- **更新**: 上流ファイルに変更がなくても強制的に再生成

### 手動実行
- **方法**: GitHub ActionsのUIから「Run workflow」ボタンをクリック
- **用途**: 緊急時やテスト目的での実行

## コミットメッセージの形式

ワークフローは更新理由に応じて異なるコミットメッセージを生成：

```
# userconf変更時
Update adblock.conf - userconf modified - 2025-09-01 - 45123 domains

# 上流更新時  
Update adblock.conf - upstream hosts updated - 2025-09-01 - 45456 domains

# 初回実行時
Initial adblock.conf generation - 2025-09-01 - 44998 domains
```

## ファイル構成

```
.
├── .github/
│   └── workflows/
│       └── update-adblock.yml    # ワークフロー定義
├── README.md                     # このファイル
├── adblock.conf                 # 生成されるdnsmasq設定（自動更新）
├── hosts_current                # 差分検出用ファイル（自動更新）
└── userconf                     # カスタムルール（オプション）
```

## dnsmasq設定例

### 基本設定
```bash
# /etc/dnsmasq.conf
port=53
domain-needed
bogus-priv
conf-file=/path/to/adblock.conf
cache-size=1000
```

### systemd設定
```bash
# dnsmasqの有効化
sudo systemctl enable dnsmasq

# 設定変更後の再起動
sudo systemctl restart dnsmasq

# ステータス確認
sudo systemctl status dnsmasq
```

### データソース
- **メインソース**: [StevenBlack/hosts](https://github.com/StevenBlack/hosts/blob/master/hosts)
- **更新頻度**: 上流リポジトリの更新に依存（通常は週次〜月次）

### 変換処理
```bash
# hostsファイル形式からdnsmasq形式への変換
cat hosts | awk '/^0.0.0.0/ {print "address=/."$2"/0.0.0.0"}' 
```

### ハッシュベース差分検出
```bash
# SHA256ハッシュを使用した変更検出
sha256sum hosts_file | cut -d' ' -f1
```

## 関連リンク

- [StevenBlack/hosts](https://github.com/StevenBlack/hosts) - アップストリームのホストリスト
- [dnsmasq Documentation](http://www.thekelleys.org.uk/dnsmasq/doc.html) - dnsmasq公式ドキュメント
