# d6e Docker STF 開発ガイド（非エンジニア向け）

> **対象読者**: プログラミング経験がないコンサルタント・ビジネス担当者
> **目的**: AIコーディングツールを使って、d6eプラットフォーム用のDockerイメージを開発・公開する手順

---

## この作業で何ができるようになるのか

d6eプラットフォームでは「STF（State Transition Function）」と呼ばれるデータ処理の部品を使って業務を自動化します。STFはDockerイメージとしてパッケージ化できる仕組みになっているので、自分が使うDockerイメージを開発するのがこのガイドのゴールです。

AIコーディングツール（Claude Code や Cursor）を使えば、プログラミングの知識がなくても「こういう処理がしたい」と日本語で指示するだけでコードを生成できます。この手法を**バイブコーディング**と呼びます。

---

## 既に公開されているSTFスイート（参考・そのまま利用可）

d6eでは、すぐに使えるDocker STFスイートが公開されています。ゼロから作る前に、まずこれらを確認してください。自分の業務に合うものがあれば、そのまま利用できます。また、これらをベースにカスタマイズすることも可能です。

### 法務STFスイート（d6e-docker-stf-legal）

法務業務を自動化するための6つのSTFが含まれています。

| STF名 | できること |
|--------|-----------|
| **NDA Triage** | NDA（秘密保持契約）を受け取り、リスクをGREEN/YELLOW/REDに自動分類 |
| **Contract Review** | 契約書をレビューし、標準からの逸脱を検出。修正案（レッドライン）も生成 |
| **Legal Risk Assessment** | 重大度×可能性のマトリクスでリスクを評価・スコアリング |
| **Compliance** | GDPR等のプライバシー規制対応、DPA（データ処理契約）レビュー |
| **Canned Responses** | 法的問い合わせへのテンプレート回答を自動生成 |
| **Meeting Briefing** | 会議ブリーフィング資料の作成、アクションアイテム追跡 |

👉 https://github.com/d6e-ai/d6e-docker-stf-legal

### 財務・経理STFスイート（d6e-docker-stf-finance）

財務・経理業務を自動化するための5つのSTFが含まれています。

| STF名 | できること |
|--------|-----------|
| **Financial Statements** | 損益計算書・貸借対照表・キャッシュフロー計算書・試算表の生成 |
| **Journal Entry** | 仕訳帳の作成・検証、減価償却計算、未払費用の計上 |
| **Variance Analysis** | 予算対実績の差異分析、ウォーターフォールチャート生成 |
| **Reconciliation** | 銀行照合、GL対補助元帳照合、会社間照合 |
| **Close Management** | 月次決算のタスク管理、進捗追跡、ブロッカー特定 |

👉 https://github.com/d6e-ai/d6e-docker-stf-finance

> **ポイント**: 上記のスイートはいずれもDockerイメージとしてGHCR（GitHub Container Registry）で公開されています。例えば `ghcr.io/d6e-ai/stf-nda-triage:latest` のように、イメージ名を指定するだけでd6eから呼び出せます。自分でSTFを開発する際の構成やコードの書き方の参考にもなります。

---

## 全体の流れ（4ステップ）

```
① プロジェクトフォルダを作る
     ↓
② AIに「d6eの作り方」を教える（Agent Skillsのインストール）
     ↓
③ AIと対話しながらコードを書く（バイブコーディング）
     ↓
④ 完成品をインターネットに公開する（GHCR）
```

---

## 事前準備（初回のみ）

### 1. Homebrewをインストールする

Homebrew はMac/Linux用のパッケージ管理ツールです。以下の公式サイトの手順に従ってインストールしてください。

👉 https://brew.sh/

> **Windowsの場合**: WSL2（Windows Subsystem for Linux）を先にインストールしてから、その中でHomebrewを使います。

### 2. 必要なツールをインストールする

Homebrewが入ったら、以下のコマンドを順に実行します。

```bash
# Node.js（コマンドツールの実行に必要）
brew install node

# Git（ソースコード管理）
brew install git

# GitHub CLI（GitHubの操作をターミナルから行う）
brew install gh

# Docker Desktop（コンテナの作成・テスト）
brew install --cask docker
```

### 3. GitHub CLIでログインする

```bash
gh auth login
```

対話形式で聞かれるので、以下のように選択します。

1. **「What account do you want to log into?」** → `GitHub.com`
2. **「What is your preferred protocol?」** → `HTTPS`
3. **「Authenticate Git with your GitHub credentials?」** → `Yes`
4. **「How would you like to authenticate?」** → `Login with a web browser`

ブラウザが開くので、GitHubにログインして認証を完了してください。

### 4. AIコーディングツールを選ぶ

以下の **どちらかの組み合わせ** を選んでインストールしてください。

| 選択肢 | ツール | インストール方法 | 特徴 |
|--------|--------|------------------|------|
| **A** | **Claude Code** + **VS Code** | 下記参照 | ターミナル操作が中心。パワフル |
| **B** | **Cursor** | 下記参照 | VS Code風のエディタ。視覚的で直感的 |

#### 選択肢A: Claude Code + VS Code

```bash
# VS Code（コードエディタ）
brew install --cask visual-studio-code

# Claude Code（AIコーディングツール）
brew install claude-code
```

> Claude Code の利用には Anthropic のアカウントが必要です。https://claude.ai/ でサインアップしてください。

#### 選択肢B: Cursor

```bash
brew install --cask cursor
```

> Cursor は VS Code をベースにAI機能を統合したエディタです。https://cursor.com/ でアカウントを作成してください。

### 5. GitHubアカウントを持っていない場合

https://github.com/ でアカウントを作成してください（無料）。

---

## ステップ① プロジェクトフォルダを作る

ターミナル（Macは「ターミナル」、Windowsは「PowerShell」）を開いて、以下を1行ずつ入力してEnterを押します。

```bash
# 作業フォルダを作成して移動
mkdir my-d6e-stf
cd my-d6e-stf

# Gitで管理を開始
git init
```

**何をしているか**: `mkdir` は「フォルダを作る」、`cd` は「そのフォルダに移動する」、`git init` は「このフォルダをGitで管理する」という意味です。

---

## ステップ② Agent Skillsをインストールする

### Agent Skillsとは？

AIコーディングツールに「d6e用のDockerイメージの正しい作り方」を教える説明書のようなものです。これをインストールすると、AIが d6e の仕様を理解した上でコードを生成してくれるようになります。

### インストール手順

ターミナルで以下を実行します。

```bash
# d6e専用のスキルをインストール
npx skills add d6e-ai/d6e-docker-stf-skills
```

実行すると対話形式で質問されます。

1. **「Which skills would you like to install?」** → すべて選択（スペースキーで選択、Enterで確定）
2. **「Which agents?」** → 使うツールを選択
   - Claude Codeを使う場合 → `claude-code`
   - Cursorを使う場合 → `cursor`
   - 両方使う場合 → 両方選択
3. **「Installation method?」** → `Symlink`（推奨）を選択

### 確認方法

インストールが完了すると、プロジェクト内に以下のようなフォルダが作られます。

```
my-d6e-stf/
├── .claude/skills/     ← Claude Code用
│   └── d6e-docker-stf-development/
│       └── SKILL.md
└── .cursor/skills/     ← Cursor用
    └── d6e-docker-stf-development/
        └── SKILL.md
```

> **補足**: Vercel公式のスキルも追加したい場合は以下のコマンドも実行できます。
> ```bash
> npx skills add vercel-labs/agent-skills
> ```

---

## ステップ③ バイブコーディング（AIと対話してコードを書く）

### バイブコーディングとは

従来のプログラミングでは、人間がコードの一行一行を自分で書く必要がありました。バイブコーディングでは、AIに「こういうものを作って」と日本語で伝え、AIがコードを生成します。人間の役割は「何を作りたいか」を明確に伝え、できあがったものを確認・修正指示することです。

### Claude Code + VS Codeの場合

VS Codeでプロジェクトフォルダを開いた状態で、VS Code内のターミナル（Ctrl+`）で以下を実行してClaude Codeを起動します。

```bash
claude
```

起動したら、日本語でやりたいことを伝えます。

**指示の例**:

```
d6eプラットフォーム用のDocker STFを作ってください。

機能: CSVファイルを読み込んで、指定されたカラムの合計値を計算する
入力: CSVファイルのパスと、合計したいカラム名
出力: 計算結果のJSON

Dockerfileも作成して、ローカルでテストできるようにしてください。
```

AIが以下のファイルを自動生成してくれます。

| ファイル | 役割 |
|----------|------|
| `Dockerfile` | Dockerイメージの設計図 |
| `index.js` (または `.ts`) | 実際の処理ロジック |
| `package.json` | 使用するライブラリの一覧 |
| `README.md` | 使い方の説明 |

### Cursorの場合

1. Cursorアプリを開く
2. File → Open Folder で `my-d6e-stf` フォルダを開く
3. 右側のチャット欄（Cmd+L / Ctrl+L）で同様に日本語で指示する

### ローカルでテストする

AIが生成したコードが動くか確認します。ターミナルで以下を実行。

```bash
# Dockerイメージをビルド（組み立て）
docker build -t my-d6e-stf .

# テスト実行
docker run my-d6e-stf
```

エラーが出た場合は、エラーメッセージをそのままAIに貼り付けて「これを直して」と伝えればOKです。

---

## ステップ④ GHCR（GitHub Container Registry）でDockerイメージを公開する

### GHCRとは？

GitHubが提供するDockerイメージの保管・配布サービスです。ここに公開すると、d6eプラットフォームや他のユーザーがイメージを取得して使えるようになります。

### 手順

#### 4-1. GHCRにログインする

事前準備で `gh auth login` 済みなら、以下のコマンドだけでDockerのログインも完了します。

```bash
gh auth token | docker login ghcr.io -u YOUR_GITHUB_USERNAME --password-stdin
```

> `YOUR_GITHUB_USERNAME` は自分のGitHubユーザー名に置き換えてください。

#### 4-2. イメージにタグを付ける

```bash
# 形式: ghcr.io/あなたのGitHubユーザー名/イメージ名:バージョン
docker tag my-d6e-stf ghcr.io/YOUR_GITHUB_USERNAME/my-d6e-stf:latest
docker tag my-d6e-stf ghcr.io/YOUR_GITHUB_USERNAME/my-d6e-stf:1.0.0
```

**タグとは**: イメージに名札を付ける作業です。`latest` は「最新版」、`1.0.0` はバージョン番号です。

#### 4-3. GHCRにプッシュ（アップロード）する

```bash
docker push ghcr.io/YOUR_GITHUB_USERNAME/my-d6e-stf:latest
docker push ghcr.io/YOUR_GITHUB_USERNAME/my-d6e-stf:1.0.0
```

#### 4-4. 公開設定を確認する

1. GitHubの自分のプロフィールページを開く
2. 「Packages」タブをクリック
3. アップロードしたイメージが表示されていることを確認
4. 必要に応じて「Package settings」から公開範囲（Public / Private）を設定

---

## よくあるトラブルと解決法

| 症状 | 原因 | 対処法 |
|------|------|--------|
| `npx skills` が動かない | Node.jsが入っていない | `brew install node` を実行 |
| `docker build` でエラー | Docker Desktopが起動していない | Docker Desktopを起動してから再実行 |
| `docker login` で認証エラー | `gh auth login` が未実施 | `gh auth login` を実行してから再試行 |
| `docker push` で403エラー | イメージ名のユーザー名が違う | `ghcr.io/自分のユーザー名/...` になっているか確認 |
| AIが変なコードを生成した | 指示が曖昧だった | より具体的に「入力・出力・処理内容」を伝え直す |

---

## 用語集

| 用語 | 説明 |
|------|------|
| **Homebrew** | Mac/Linux用のパッケージ管理ツール。`brew install ○○` でツールを簡単にインストールできる |
| **GitHub CLI (gh)** | GitHubの操作をターミナルから行うツール。ログインやリポジトリ管理がコマンド1つでできる |
| **Docker** | アプリケーションを「コンテナ」という箱に入れて、どの環境でも同じように動かせる技術 |
| **Dockerイメージ** | コンテナの設計図。レシピのようなもの |
| **コンテナ** | Dockerイメージから作った実行中のアプリ。レシピから作った料理のようなもの |
| **GHCR** | GitHub Container Registry。Dockerイメージを保管・共有するGitHubのサービス |
| **STF** | State Transition Function。d6eでデータを処理する部品 |
| **Agent Skills** | AIコーディングツールに専門知識を教える仕組み |
| **バイブコーディング** | AIに自然言語（日本語）で指示してコードを生成させる開発手法 |
| **npx** | Node.jsのコマンド実行ツール。インストール不要でツールを実行できる |
| **Git** | ファイルの変更履歴を管理するツール。いつでも過去の状態に戻せる |
| **ターミナル** | コマンド（テキストの命令文）でPCを操作する画面 |
| **ビルド** | ソースコードからDockerイメージを組み立てること |
| **プッシュ** | ローカルのファイルやイメージをサーバーにアップロードすること |

---

## 参考リンク

- [Homebrew](https://brew.sh/)
- [Agent Skills 公式サイト](https://skills.sh)
- [Agent Skills CLI（vercel-labs/skills）](https://github.com/vercel-labs/skills)
- [d6e Docker STF Skills](https://github.com/d6e-ai/d6e-docker-stf-skills)
- [d6e Docker STF Legal（法務STFスイート）](https://github.com/d6e-ai/d6e-docker-stf-legal)
- [d6e Docker STF Finance（財務・経理STFスイート）](https://github.com/d6e-ai/d6e-docker-stf-finance)
- [GHCR ドキュメント](https://docs.github.com/en/packages/working-with-a-github-packages-registry/working-with-the-container-registry)
- [GitHub CLI](https://cli.github.com/)
- [Claude Code](https://claude.ai/code)
- [VS Code](https://code.visualstudio.com/)
- [Cursor](https://cursor.com)
