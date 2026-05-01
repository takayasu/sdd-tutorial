# Step 0: 環境構築

## 目的

### これは何か

このPoCで使うソフトウェア（プログラミング言語、ビルドツール、コンテナ環境など）を自分のPCにインストールする作業。

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

$ dotnet --version
8.0.xxx                                 # 8.0以上

$ gradle --version
Gradle 8.x                             # 8以上

$ java --version
openjdk 21.x.x                         # 21以上

$ gitleaks version
v8.x.x                                 # 表示されればOK

$ trivy --version
Version: 0.x.x                          # 表示されればOK

$ scc --version
scc version x.x.x                       # 表示されればOK
```

---

## F# 環境

### 1. .NET SDK インストール

```bash
# Ubuntu/Debian
sudo apt-get update
sudo apt-get install -y dotnet-sdk-8.0

# macOS
brew install dotnet-sdk

# 確認
dotnet --version
```

### 2. ツールのインストール

```bash
# Fantomas（フォーマッター）
dotnet tool install -g fantomas

# FSharpLint（リンター）
dotnet tool install -g dotnet-fsharplint

# 確認
dotnet fantomas --version
dotnet fsharplint --version
```

---

## Kotlin 環境

### 1. JDK インストール

```bash
# Ubuntu/Debian
sudo apt-get install -y openjdk-21-jdk

# macOS
brew install openjdk@21

# 確認
java --version
```

### 2. Gradle インストール

```bash
# SDKMAN推奨
curl -s "https://get.sdkman.io" | bash
source "$HOME/.sdkman/bin/sdkman-init.sh"
sdk install gradle

# 確認
gradle --version
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

### scc（F#用の複雑度予約）

```bash
# macOS
brew install scc

# Linux
go install github.com/boyter/scc/v3@latest

# 確認
scc --version
```

---

## 次のステップ

Step 0が完了したら [Step 1: Hello World API](./step01.md) へ進む。
