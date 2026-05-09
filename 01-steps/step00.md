# Step 0: 環境構築

## 目的

### これは何か

このPoCで使うソフトウェア（プログラミング言語、ランタイム、コンテナ環境など）を自分のPCにインストールする作業。

### なぜやるのか

以降の全ステップはこれらのツールが入っていることを前提にしている。最初にまとめて入れておくことで、途中で「あれが足りない」と中断されることを防ぐ。

### 何がうれしいのか

- 全員が同じツール・同じバージョンで作業するため、「自分の環境では動くのに」という問題が起きにくい
- バージョンを揃えておくことで、AIに質問したときの回答精度も上がる

## 完了条件

以下のコマンドを実行し、それぞれ期待される出力が得られること。

```bash
$ docker --version
Docker version 27.x.x, build xxxxxxx   # 27以上

$ docker compose version
Docker Compose version v2.x.x          # v2以上

$ python --version
Python 3.12.x                           # 3.12以上

$ uv --version
uv 0.x.x                               # 表示されればOK

$ node --version
v22.x.x                                # 22 LTS以上

$ pnpm --version
9.x.x                                  # 9以上

$ gitleaks version
v8.x.x                                 # 表示されればOK

$ trivy --version
Version: 0.x.x                          # 表示されればOK
```

---

## Python 環境

### 1. Python インストール

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y python3.12 python3.12-venv

# macOS（pyenv推奨）
brew install pyenv
pyenv install 3.12
pyenv global 3.12

# 確認
python --version
```

### 2. uv インストール（パッケージ管理）

`uv` は Rust 製の高速Pythonパッケージマネージャ。`pip` + `venv` を置き換える。

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# 確認
uv --version
```

---

## Node.js 環境（フロントエンド）

### 1. Node.js インストール

```bash
# fnm（高速なバージョンマネージャ）推奨
curl -fsSL https://fnm.vercel.app/install | bash
fnm install 22
fnm use 22

# 確認
node --version
```

### 2. pnpm インストール

```bash
npm install -g pnpm

# 確認
pnpm --version
```

---

## 共通ツール

### Docker

```bash
# Docker Engine + Docker Compose（公式手順に従う）
# https://docs.docker.com/engine/install/

# 確認
docker --version
docker compose version
```

### gitleaks

```bash
# macOS
brew install gitleaks

# Linux
wget https://github.com/gitleaks/gitleaks/releases/latest/download/gitleaks_linux_x64 -O gitleaks
chmod +x gitleaks
sudo mv gitleaks /usr/local/bin/

# 確認
gitleaks version
```

### Trivy

```bash
# macOS
brew install trivy

# Linux
sudo apt-get install -y trivy

# 確認
trivy --version
```

---

## プロジェクト構成（全体像）

```
sales-management/
├── backend/                   # Python + FastAPI
│   ├── src/
│   │   ├── domain/            # 型定義 + 純粋関数（ドメインロジック）
│   │   ├── infra/             # DB リポジトリ（SQLAlchemy）
│   │   └── api/               # FastAPI ルーティング
│   ├── tests/
│   │   ├── domain/            # ドメインロジックのPBT
│   │   └── api/               # APIの統合テスト
│   ├── alembic/               # DBマイグレーション
│   ├── pyproject.toml
│   └── alembic.ini
├── frontend/                  # React + TypeScript + Vite
│   ├── src/
│   │   ├── domain/            # TypeScript 型定義
│   │   ├── api/               # APIクライアント（openapi-typescript生成）
│   │   └── pages/             # React ページコンポーネント
│   ├── tests/
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml
└── ci.sh
```

---

## 次のステップ

Step 0が完了したら [Step 1: Hello World API](./step01.md) へ進む。
