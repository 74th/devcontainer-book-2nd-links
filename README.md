# 同人誌『DevContainer実践ガイド』サンプルコード

<img src="./material/ebook.png" alt="DevContainer実践ガイド表紙" width="600"/>

このリポジトリは技術書典19にては [@74th](https://github.com/74th) が頒布した同人誌『DevContainer実践ガイド』の、本文掲載リンクとコードを収録しています。

販売先: Booth https://74th.booth.pm/items/7605652

# 2. DevContainerの構築を考える

## 2.3. DevContainerコンテナ構築のフェーズを理解する

### DevContainer Features(C3)

> devcontainers/images<br/>https://github.com/devcontainers/images

> Available Dev Container Templates<br/>https://containers.dev/templates

# 3. DevContainer設定と、各IDE、CLIでの使い方

## 3.1. DevContainer構築の設定ファイル

### devcontainer.json

#### 単一のコンテナイメージ

```json
// .devcontainer/devcontainer.json
{
  "name": "single-docker-image",
  "image": "mcr.microsoft.com/devcontainers/python:3.14-trixie"
}
```

#### Dockerfile

```json
// .devcontainer/devcontainer.json
{
  "name": "single-dockerfile",
  "build": {
    "context": "."
  },
  "dockerFile": "./Dockerfile"
}
```

#### Docker Compose

```yaml
# .devcontainer/compose.yml
services:
  dev:
    build:
      context: .
      dockerfile: ./Dockerfile
    # 起動したままにできるようにする
    command: sleep infinity
    volumes:
      # ワークスペースのマウント
      - ..:/workspace:cached
```

```json
// .devcontainer/devcontainer.json
{
  "name": "compose",
  // Docker Composeのマニフェストファイル
  "dockerComposeFile": "./compose.yml",
  // DevContainerとして起動するサービス名
  "service": "dev",
  // ワークスペースディレクトリ
  "workspaceFolder": "/workspace"
}
```

#### 環境変数


```json
{
  "remoteEnv": {
    "PATH": "/home/vscode/npm/bin:${containerEnv:PATH}",
    "CLAUDE_CODE_USE_BEDROCK": "1"
  },
}
```

#### 転送ポート

```json
{
  "forwardPorts": ["18080:8080", 5432]
}
```

#### マウント

```json
{
  "mounts": [
    {
      "source": "${localEnv:HOME}/.llm/claude/.claude.json",
      "target": "/home/vscode/.claude.json",
      "type": "bind"
    },
    "source=${localEnv:HOME}/.llm/claude/.claude,target=/home/vscode/.claude,type=bind,consistency=cached"
  ]
}
```

#### ライフサイクルスクリプト

> ライフサイクルスクリプトの表記法の公式ドキュメント<br/>https://containers.dev/implementors/spec/#parallel-exec

#### 各エディタ（主にVS Code）の独自項目

```json
{
  "customizations": {
    "vscode": {
      "extensions": [
        "Anthropic.claude-code"
      ],
      "settings": {
        "claude-code.environmentVariables": [
          {
            "name": "CLAUDE_CODE_USE_VERTEX",
            "value": "1"
          }
        ]
  // 略
```

## 3.5. DevContainer CLI

```bash
# ビルド
devcontainer build --workspace-folder .
# 起動
devcontainer up --workspace-folder .
# コマンドの実行 (bash を開く)
devcontainer exec --workspace-folder . bash
```

## 3.7. Docker Desktop以外のコンテナランタイムアプリケーションは使えるか

> Docker Desktop<br/>https://docs.docker.com/desktop/

> Podman Desktop<br/>https://podman-desktop.io/

> Rancher Desktop<br/>https://rancherdesktop.io/

> Finch<br/>https://github.com/runfinch/finch

> Colima<br/>https://github.com/abiosoft/colima

> Apple Container<br/>https://github.com/apple/container

```bash
# Rancher Desktopの場合
docker context use rancher-desktop
# Colimaの場合
docker context use colima
```

```json
{
  // User/settings.json
  // Podman
  "dev.containers.dockerPath": "/opt/podman/bin/podman",
  "dev.containers.dockerComposePath": "/opt/podman/bin/podman compose",
  // finch
  "dev.containers.dockerPath": "/usr/local/bin/finch",
  "dev.containers.dockerComposePath": "/usr/local/bin/finch compose"
}
```

# 4. DevContainerの細かい使い方

## 4.1. ファイルシステムへのアクセスが遅い問題への対処

### macOSの場合

#### Docker Desktopを使わない場合

> 1: https://github.com/74th/test-devcontainer/tree/main/20251025-fio

#### LinuxVM上で動かす

> 1: https://github.com/74th/test-devcontainer/tree/main/20251025-fio

### Windowsの場合

```bash
# Dockerのインストール
sudo apt update
sudo apt install -y docker.io

# ユーザ権限でdockerコマンドを実行できるようにする
sudo usermod -aG docker $USER
```

## 4.2. 非rootユーザを利用する

### common-utilsを使用する（第2章のE3）

```json
// .devcontainer/devcontainer.json
{
  "dockerFile": "Dockerfile",
  "features": {
    // common-utils
    "ghcr.io/devcontainers/features/common-utils:2": {}
  },
  // non-rootユーザの使用の指定
  "containerUser": "vscode",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh"
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.2-C3-add_non_root_user

### Dockerfileに記述を追加する（第2章のE2）

```Dockerfile
FROM python:3.13.7-trixie

# 非rootユーザの引数
ARG USERNAME=vscode
ARG USER_UID=1000
ARG USER_GID=$USER_UID

# 非rootユーザの追加
RUN groupadd --gid $USER_GID $USERNAME \
    && useradd --uid $USER_UID --gid $USER_GID -m $USERNAME
# sudo の追加
RUN apt-get update \
    && apt-get install -y sudo \
    && echo $USERNAME ALL=\(root\) NOPASSWD:ALL > /etc/sudoers.d/$USERNAME \
    && chmod 0440 /etc/sudoers.d/$USERNAME

# ユーザの切り替え
USER $USERNAME
```

> Add a non-root user to a container<br/>https://code.visualstudio.com/remote/advancedcontainers/add-nonroot-user

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.2-C2-add_non_root_user

## 4.3. Windowsで改行コードの差分が出る問題

```
git config --global core.autocrlf false
```

## 4.4. ホストに影響を与えること

### docker-outside-of-docker

```bash
docker run -it --rm -v /Users/nnyn:/Users/nnyn debian ls /Users/nnyn
```

### ネットワーク

```
ssh nnyn@host.docker.internal
```

```
ssh nnyn@$(ip route | awk '/default/ { print $3 }')
```

## 4.5. シークレットの持ち込み方

### ローカルの環境変数から、コンテナの環境変数に渡す（非推奨）

```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "remoteEnv": {
    "AWS_BEARER_TOKEN_BEDROCK": "${localEnv:AWS_BEARER_TOKEN_BEDROCK}"
  }
}
```

### envfileから、コンテナの環境変数に渡す

```json
// .devcontainer/devcontainer.json
{
    "image": "mcr.microsoft.com/devcontainers/base:bullseye",
    "runArgs": [
        "--env-file",
        ".devcontainer/devcontainer.env"
    ]
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.5-secret_from_env

### ディレクトリマウントで渡す

```bash
# .devcontainer/bashrc
# bedrock のシークレットを環境変数で読み込む
export AWS_BEARER_TOKEN_BEDROCK=$(cat /run/secrets/bedrock_api_key)
```

```bash
#!/bin/bash
# .devcontainer/postCreateCommand.sh
# bashrc に bedrock のシークレットを環境変数で読み込むようにbashrcを追記
cat .devcontainer/bashrc >> ~/.bashrc
```

```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "mounts": [
    "source=${localEnv:HOME}/.secrets/bedrock_api_key,⏎
     target=/run/secrets/bedrock_api_key,type=bind,consistency=cached",
  ],
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.5-secret_from_mount

### docker composeのシークレット機能を使う

```yaml
# .devcontainer/compose.yml
secrets:
  bedrock_api_key:
    file: ~/.secrets/bedrock_api_key

services:
  # devcontainerのコンテナ
  dev:
    image: mcr.microsoft.com/devcontainers/base:bullseye
    command: sleep infinity
    volumes:
      - ..:/workspace:cached
    secrets:
      - source: bedrock_api_key
        target: bedrock_api_key
        mode: 0444
```

```json
{
  "dockerComposeFile": "./compose.yaml",
  "service": "dev",
  "workspaceFolder": "/workspace",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh"
}
```


> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.5-secret_from_compose_secrets

### コンテナ起動時にシークレットを更新する

```py
# /// script
# requires-python = ">=3.13"
# dependencies = [
#     "aws-bedrock-token-generator",
# ]
# ///
# .devcontainer/generate_bedrock_shortterm_api_key_to_secrets.py
import pathlib
from aws_bedrock_token_generator import provide_token

output_path = pathlib.Path("~/.secrets/bedrock_api_key").expanduser()

token = provide_token(region="ap-northeast-1")

output_path.parent.mkdir(parents=True, exist_ok=True)
output_path.write_text(token)
```

```bash
#!/bin/bash
# .devcontainer/initializeCommand.sh
uv run .devcontainer/generate_bedrock_shortterm_api_key_to_secrets.py
```

```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "initializeCommand": ".devcontainer/initializeCommand.sh",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "mounts": [
    "source=${localEnv:HOME}/.secrets/bedrock_api_key,⏎
     target=/run/secrets/bedrock_api_key,type=bind,consistency=cached",
  ]
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.5-secret_from_env

## 4.6. GitHubの認証情報の持ち込み方

### GitHub CLIでコンテナ内で認証する

```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "features": {
    "ghcr.io/devcontainers/features/github-cli:1": {}
  }
}
```

```bash
gh auth login
```

```bash
git config --global user.name "Atsushi Morimoto (74th)"
git config --global user.email "74th.tech@gmail.com"
git commit -m 'foo' --allow-empty
git push
```

> https://github.com/74th/devcontainer-book-2nd-samples-github-auth/tree/main/.devcontainer/gh

### GitHub Personal Access Tokenを環境変数渡しで使う

```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "remoteEnv": {
    "GH_TOKEN": "${localEnv:GH_TOKEN}"
  }
}
```

```bash
git config --global credential.helper \
  '!f() { echo username=x; echo password=${GH_TOKEN}; }; f'
git config --global credential.useHttpPath true
```

> https://github.com/74th/devcontainer-book-2nd-samples-github-auth/blob/main/.devcontainer/token_with_env

### GitHub Personal Access Tokenをマウントして利用する

```bash
#!/bin/bash
# .devcontainer/initializeCommand.sh
# ホスト側で実行される
gh auth token > ~/.secrets/gh_token
```

```bash
#!/bin/bash
# .devcontainer/postCreateCommand.sh
git config --global credential.helper '!f() { echo username=x; ⏎
  echo password=$(cat /run/secrets/gh_token 2>/dev/null); }; f'
git config --global credential.useHttpPath true
```

```json
// .devcontainer/devcontainer.json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "postCreateCommand": "./.devcontainer/postCreateCommand.sh",
  "initializeCommand": "./.devcontainer/initializeCommand.sh",
  "mounts": [
    "source=${localEnv:HOME}/.secrets/gh_token,⏎
     target=/run/secrets/gh_token,type=bind"
  ]
}
```

> https://github.com/74th/devcontainer-book-2nd-samples-github-auth/tree/main/.devcontainer/gh_token_with_secret_mount

## 4.7. dotfilesの利用

```json
// User/settings.json
{
  "dotfiles.repository": "https://github.com/74th/dotfiles.git",
  "dotfiles.installCommand": "install_devcontainer.sh"
}
```

```bash
#!/bin/bash
# ~/dotfiles/install_devcontainer.sh
if ! grep -q "source ~/dotfiles/bashrc/devcontainer.bashrc" ~/.bashrc; then
    echo "source ~/dotfiles/bashrc/devcontainer.bashrc" >> ~/.bashrc
fi
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.7-git_config_from_dotfiles

## 4.8. 個人の設定を使う

### Docker Compose のオーバーライドを使う

```json
// .devcontainer/devcontainer.json
{
    "dockerComposeFile": "compose.yml",
    "service": "dev",
    "workspaceFolder": "/workspace"
}
```

```yaml
# .devcontainer/compose.yml
services:
  redis:
    image: redis:latest
    # ポートの設定
    ports:
      - "6379:6379"
  dev:
    image: mcr.microsoft.com/devcontainers/base:bullseye
    volumes:
      - ..:/workspace
    command: /bin/sh -c "while sleep 1000; do :; done"
```

```yaml
# .devcontainer/private/compose.override.yml
services:
  redis:
    # !override がないとポートの上書きではなく、追加になる
    ports: !override
      - "16379:6379"
```

```json
// .devcontainer/private/devcontainer.json
{
    "dockerComposeFile": ["../compose.yml", "./compose.override.yml"],
    "service": "dev",
    "workspaceFolder": "/workspace"
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/4.8-override_settings

# 5. DevContainer Features

## 5.2. Featuresの使い方

### 基本

```json
{
  "name": "docker-in-docker",
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "features": {
    "ghcr.io/devcontainers/features/docker-in-docker:2": {}
  }
}
```

> Available Dev Container Features<br/>https://containers.dev/features

### Featuresのオプション

```json
{
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "features": {
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.11"  // Pythonのバージョンを指定
    }
  }
}
```

## 5.3. 有用なFeatures

#### AWS CLI

```json
"features": {
  "ghcr.io/devcontainers/features/aws-cli:1": {}
}
```

> https://github.com/devcontainers/features/tree/main/src/aws-cli

#### Docker (Docker-in-Docker)

```json
"features": {
  "ghcr.io/devcontainers/features/docker-in-docker:2": {}
}
```

> https://github.com/devcontainers/features/tree/main/src/docker-in-docker

#### Docker (docker-outside-of-docker)

```json
"features": {
    "ghcr.io/devcontainers/features/docker-outside-of-docker:1": {}
}
```

> https://github.com/devcontainers/features/tree/main/src/docker-outside-of-docker

#### Common Utilities

```json
"features": {
    "ghcr.io/devcontainers/features/common-utils:2": {}
}
```

> https://github.com/devcontainers/features/tree/main/src/common-utils

#### Python

```json
"features": {
    "ghcr.io/devcontainers/features/python:1": {
      "version": "3.12"
    }
}
```

> https://github.com/devcontainers/features/tree/main/src/python

#### Node.js

```json
"features": {
    "ghcr.io/devcontainers/features/node:1": {
      "version": "lts"
    }
}
```

> https://github.com/devcontainers/features/tree/main/src/node

## 5.4. Featuresを自作する

> https://github.com/74th/devcontainer-book-2nd-samples-devcontainer-features

### テンプレートを利用する

> A bootstrap repo for self-authoring Dev Container Features<br/>https://github.com/devcontainers/feature-starter

### マニフェストファイルの記述

> devcontainer-feature.json properties<br/>https://containers.dev/implementors/features/#devcontainer-feature-json-properties

```json
// src/flyway-8.1.0/devcontainer-feature.json
{
  "name": "flyway-8.1.0",
  "id": "flyway-8.1.0",
  "version": "0.0.1",
  "description": "A flyway feature",
  "options": {},
  "containerEnv": {
    "PATH": "/flyway:${PATH}"
  },
  "dependsOn": {
    "ghcr.io/devcontainers/features/java:1": {}
  },
  "installsAfter": ["ghcr.io/devcontainers/features/common-utils"]
}
```

#### options

```json
"options": {
  "version": {
    "type": "string",
    "proposals": [
      "latest",
      "none",
      "1.24",
      "1.23"
    ],
    "default": "latest",
    "description": "Select or enter a Go version to install"
  },
  "golangciLintVersion": {
    "type": "string",
    "default": "latest",
    "description": "Version of golangci-lint to install"
  }
},
```

#### containerEnv

```json
"containerEnv": {
  "PATH": "/opt/flyway/bin:${PATH}"
},
```

#### dependsOn

```json
"dependsOn": {
  "ghcr.io/devcontainers/features/java:1": {}
},
```

#### installsAfter

```json
"installsAfter": ["ghcr.io/devcontainers/features/common-utils"]
```

#### Lifecycle Hooks

```json
"postStartCommand": "/opt/embedding_bedrock_token/postStartCommand.sh"
```

### インストールスクリプトの記述

```bash
#!/bin/sh
set -e

# optionのバージョンを受け取る
FLYWAY_VERSION="${VERSION:-"8.1.0"}"

echo "Activating feature 'flyway'"

# 重複インストールの判定
if [ -d "/usr/local/lib/flyway" ]; then
  echo "Flyway is already installed"
  exit 0
fi

# インストール作業に必要なパッケージのインストール
if ! command -v curl >/dev/null; then
  echo "curl is not installed, installing it now"
  # ここではDebian系のみをサポート
  apt-get update && apt-get install -y curl
fi


# /usr/local/lib/flywayにインストール
mkdir -p /usr/local/lib/flyway
cd /usr/local/lib/flyway
curl -L https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/⏎
  ${FLYWAY_VERSION}/flyway-commandline-${FLYWAY_VERSION}.tar.gz ⏎
  -o flyway-commandline-${FLYWAY_VERSION}.tar.gz
tar -xzf flyway-commandline-${FLYWAY_VERSION}.tar.gz --strip-components=1
rm flyway-commandline-${FLYWAY_VERSION}.tar.gz
# 読み込み権限を付与
chmod 755 /usr/local/lib/flyway/flyway
ln -s /usr/local/lib/flyway/flyway /usr/local/bin/flyway
```

#### パッケージのインストールのためのディストリビューションの判別

> https://github.com/devcontainers/features/blob/feature_common-utils_2.5.4/src/common-utils/main.sh#L348-L361

#### Featuresのその他のファイルの参照

```bash
exec /bin/bash "$(dirname $0)/main.sh"
```

### テストする

```json
{
  "test": {
    "image": "mcr.microsoft.com/devcontainers/base:ubuntu",
    "features": {
      "flyway": {
        "version": "10.0.0"
      }
    }
  }
}
```

```bash
#!/bin/bash
# test/flyway/test.sh
set -e

# テスト用のツールを読み込む
source dev-container-features-test-lib

# これで終了コードが0であればインストール成功とみなす
check "execute command" bash -c "flyway --version"

# 最後に結果を報告
reportResults
```

```bash
#!/bin/bash
# test/flyway/duplicate.sh
set -e

source dev-container-features-test-lib

check "execute command" bash -c "flyway --version | grep '8.1.0'"

reportResults
```

```
# すべてのFeaturesのテストを実行
devcontainer features test

# 特定のFeatureのテストのみを実行
devcontainer features test -f <Feature名>

# 特定のFeatureの特定のシナリオのみ実行
# --skip-autogenerated --skip-duplicated オプションがないと、
# 重複インストール、自動生成のテストはスキップされない
devcontainer features test -f <Feature名> --filter <シナリオ名> \
  --skip-autogenerated --skip-duplicated
```

# 6. DevContainer構築例

## 6.1. Go

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/6.1-golang_with_language_official_image

### ランタイムとツールの導入方法

> https://github.com/devcontainers/features/tree/main/src/go

### DevContainer設定ファイルの作成

```Dockerfile
# .devcontainer/Dockerfile
# 言語公式イメージの利用
# https://hub.docker.com/_/golang
FROM golang:1.25.1-trixie

# ここではGo以外に必要なツールがあれば導入する
```

```yaml
# .devcontainer/compose.yml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ..:/workspace:cached
      # ビルドキャッシュをコンテナ外に永続化
      - ~/.cache/devcontainer/go/cache-go-build:/home/vscode/.cache/go-build
      # gomodのキャッシュを永続化
      - ~/.cache/devcontainer/go/pkg-mod:/go/pkg/mod
      - ~/.cache/devcontainer/go/pkg-sumdb:/go/pkg/sumdb
    command: /bin/sh -c "while sleep 1000; do :; done"
```

```bash
#!/bin/bash
# .devcontainer/initializeCommand.sh
# マウント対象のディレクトリを予め作成しておく
# Dockerランタイムによっては存在しないディレクトリをマウントできなかったり、
# root権限で作成される場合があるため
mkdir -p ~/.cache/devcontainer/go/cache-go-build
mkdir -p ~/.cache/devcontainer/go/pkg-mod
mkdir -p ~/.cache/devcontainer/go/pkg-sumdb
```

```bash
#!/bin/bash
# .devcontainer/postCreateCommand.sh
# Goの各ツールのインストール
# ユーザ権限でインストールするので、ユーザが更新できる
go install golang.org/x/tools/gopls@latest
go install github.com/cweill/gotests@latest
go install github.com/josharian/impl@latest
go install github.com/go-delve/delve/cmd/dlv@latest

# golangci-lintのバージョンを固定してインストール
GOLANGCI_LINT_VERSION=v1.64.8
curl -sSfL \
  https://raw.githubusercontent.com/golangci/golangci-lint/HEAD/install.sh \
  | sh -s -- -b $(go env GOPATH)/bin $GOLANGCI_LINT_VERSION

# 依存パッケージのインストール
go get ./...
```

```json
// .devcontainer/devcontainer.json
{
  "dockerComposeFile": "compose.yml",
  "service": "dev",
  "workspaceFolder": "/workspace",
  "features": {
    // 非rootユーザを作る
    "ghcr.io/devcontainers/features/common-utils:2": {}
  },
  "initializeCommand": ".devcontainer/initializeCommand.sh",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "remoteUser": "vscode",
  "updateRemoteUserUID": true
}
```

## 6.2. Python

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/6.2-python_with_devcontainer_official_image

### uvを入れたイメージの作成

> https://github.com/devcontainers/images/tree/main/src/python

```Dockerfile
# .devcontainer/Dockerfile
# DevContainer公式イメージの利用
# https://github.com/devcontainers/images/tree/main/src/python
FROM mcr.microsoft.com/devcontainers/python:3.13

# uv のインストール
COPY --from=ghcr.io/astral-sh/uv:0.8.22 /uv /uvx /bin/
RUN mkdir -p /var/uv/cache; \
    chmod 777 /var/uv/cache
ENV UV_COMPILE_BYTECODE=1 \
    UV_SYSTEM_PYTHON=true \
    UV_CACHE_DIR=/var/uv/cache \
    UV_LINK_MODE=copy
```

### DevContainer設定ファイルの作成

```yaml
# .devcontainer/compose.yml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ..:/workspace:cached
      # uvのキャッシュを永続化
      - ~/.cache/devcontainer/python/uv-cache:/var/uv/cache:cached
    command: /bin/sh -c "while sleep 1000; do :; done"
```

```bash
#!/bin/bash
# .devcontainer/initializeCommand.sh
# マウント対象のディレクトリを予め作成しておく
# Dockerランタイムによっては存在しないディレクトリをマウントできなかったり、
# root権限で作成される場合があるため
mkdir -p ~/.cache/devcontainer/python/uv-cache
```

```bash
#!/bin/bash
# .devcontainer/postCreateCommand.sh
# sudo権限の剥奪
sudo rm -rf /etc/sudoers.d/*

# 依存パッケージのインストール
uv sync --all-groups
```

```json
// .devcontainer/devcontainer.json
{
  "dockerComposeFile": "compose.yml",
  "service": "dev",
  "workspaceFolder": "/workspace",
  "initializeCommand": ".devcontainer/initializeCommand.sh",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "remoteUser": "vscode"
}
```

## 6.3. Node.js

```json
// .devcontainer/devcontainer.json
{
  "dockerComposeFile": "compose.yml",
  "service": "dev",
  "workspaceFolder": "/workspace",
  "features": {
    "ghcr.io/devcontainers/features/common-utils:2": {},
    // Node.jsを追加インストール
    "ghcr.io/devcontainers/features/node:1": {},
  },
  "initializeCommand": ".devcontainer/initializeCommand.sh",
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "remoteUser": "vscode"
}
```

> https://github.com/devcontainers/features/tree/main/src/node

```yaml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    volumes:
      - ..:/workspace:cached
      # キャッシュを永続化
      # npm
      - ~/.cache/devcontainer/node/npm-cache:/home/vscode/.npm:cached
      - ~/.cache/devcontainer/node/npm-config:/home/vscode/.config/npm:cached
      # node-gyp / node-pre-gyp
      - ~/.cache/devcontainer/node/node-gyp:/home/vscode/.cache/node-gyp:cached
      - ~/.cache/devcontainer/node/node-pre-gyp:/home/vscode/.cache/node-pre-gyp:cached
      # pnpm
      - ~/.cache/devcontainer/node/pnpm-store:/home/vscode/.pnpm-store:cached
    command: /bin/sh -c "while sleep 1000; do :; done"
```

```bash
#!/bin/bash
mkdir -p ~/.cache/devcontainer/node/npm-cache
mkdir -p ~/.cache/devcontainer/node/npm-config
mkdir -p ~/.cache/devcontainer/node/node-gyp
mkdir -p ~/.cache/devcontainer/node/node-pre-gyp
mkdir -p ~/.cache/devcontainer/node/pnpm-store
```

```bash
# postCreateCommand.sh
pnpm config set store-dir /home/vscode/.pnpm-store
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/6.3-append_nodejs

## 6.4. DBをDocker Composeで立ち上げる

### Redis

```
# .devcontainer/redis/redis.conf
user default off
user app on >redis-password allcommands allkeys
```

```yaml
# .devcontainer/compose.yml
services:
  redis:
    image: redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf
    command:
      - redis-server
      - /usr/local/etc/redis/redis.conf
  dev: # 略
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/6.4-with_redis

### MySQL

```yaml
# .devcontainer/compose.yml
services:
  mysql:
    image: mysql:9.5.0
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root_password
      MYSQL_USER: mysql_user
      MYSQL_PASSWORD: mysql_password
      MYSQL_DATABASE: app
  dev: # 略
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/6.4-with_mysql

### PostgreSQL

```yaml
# .devcontainer/compose.yml
services:
  postgres:
    image: postgres:16
    ports:
      - "5432:5432"
    environment:
      POSTGRES_USER: app_user
      POSTGRES_PASSWORD: app_password
      POSTGRES_DB: app
      POSTGRES_INITDB_ARGS: "-E UTF8"
  dev: # 略
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/6.4-with_postgresql

# 7. ネットワーク隔離環境

## 7.1. LLMコーディングエージェントとセキュリティリスクの議論

> 「OWASP Top 10 for Large Language Model Applications」2025年版のリスク概説<br/>https://www.trendmicro.com/ja_jp/jp-security/25/e/expertview-20250522-01.html

## 7.3. Docker Composeでネットワークを作り、ホストと分離する

```yml
services:
  dev:
    build:
      context: .
      dockerfile: Dockerfile
    command: sleep infinity
    volumes:
      - ..:/workspace:cached
    networks:
      - dev-network
  sidecar:
    image: nginx
    networks:
      - dev-network

networks:
  dev-network:
    driver: bridge
    internal: true
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/7.3-network_dedicated_container

## 7.4. IP制限をする

### 方法1: iptablesでFirewallを構築しIPで制限する

> https://github.com/anthropics/claude-code/blob/main/.devcontainer/devcontainer.json

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/7.4-firewall_by_iptables

### 方法2: HTTPプロキシでドメインで制限する

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/7.4-http_proxy

# 8. ローカルでのLLMコーディングエージェントでのDevContainerの利用

## 8.2. Claude Code

Claude Codeは、Anthropicが提供するLLMコーディングエージェントです。

#### AWS Bedrock認証情報の渡し方

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/8.2-claudecode_with_bedrock

#### Google Cloud Vertex AI認証情報の渡し方

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/8.2-claudecode_with_vertexai

### DevContainerの設定

```json
// .devcontainer/devcontainer.json
{
  "name": "claude-code",
  "image": "mcr.microsoft.com/devcontainers/base:bullseye",
  "features": {
    // 公式のDevContainer Featureを利用
    "ghcr.io/anthropics/devcontainer-features/claude-code:1": {}
  },
  "mounts": [
    // 永続化
    "source=${localEnv:HOME}/.llm/claude/.claude.json,⏎
     target=/home/vscode/.claude.json,type=bind,consistency=cached",
    "source=${localEnv:HOME}/.llm/claude/.claude,⏎
     target=/home/vscode/.claude,type=bind,consistency=cached"
  ],
  "remoteEnv": {
    // npmのパスを通す
    "PATH": "/home/vscode/npm/bin:${containerEnv:PATH}"
  },
  "customizations": {
    "vscode": {
      "extensions": [
        // VS Code拡張機能
        "Anthropic.claude-code"
      ]
    }
  }
}
```

## 8.3. Codex CLI


### 永続化

### DevContainerの設定

```Dockerfile
# .devcontainer/Dockerfile
FROM mcr.microsoft.com/devcontainers/base

# node.jsをnodesource.comからインストール
RUN apt-get update && \
    apt-get install -y curl && \
    curl -fsSL https://deb.nodesource.com/setup_22.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

# ユーザ権限でnpmでインストールするためのパス設定
ENV PATH="/home/vscode/npm/bin:${PATH}"
```

```bash
# codex CLIをグローバルインストール
npm config set prefix ~/npm
npm install -g @openai/codex
```

```json
// .devcontainer/devcontainer.json
{
  "name": "codex-cli",
  "build": {
    "dockerfile": "./Dockerfile"
  },
  "mounts": [
    // 永続化
    "source=${localEnv:HOME}/.llm/codex/.codex,⏎
     target=/home/vscode/.codex,type=bind,consistency=cached"
  ],
  "remoteEnv": {
    // npmのパスを通す
    "PATH": "/home/vscode/npm/bin:${containerEnv:PATH}"
  },
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "customizations": {
    "vscode": {
      "extensions": [
        "openai.chatgpt"
      ]
    }
  }
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/8.3-codex_cli

## 8.4. Gemini CLI

### DevContainerの設定

```bash
# gemini CLIをグローバルインストール
#!/bin/bash
npm config set prefix ~/npm
npm install -g @google/gemini-cli
```

```json
{
  "name": "gemini-cli",
  "build": {
    "dockerfile": "./Dockerfile"
  },
  // Gemini CLIのインストールコマンド
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "mounts": [
    // 永続化
    "source=${localEnv:HOME}/.llm/gemini/.gemini,⏎
     target=/home/vscode/.gemini,type=bind,consistency=cached",
    "source=${localEnv:HOME}/.llm/gemini/.config/configstore,⏎
     target=/home/vscode/.config/configstore,type=bind,consistency=cached"
  ],
  "remoteEnv": {
    // npmのパスを通す
    "PATH": "/home/vscode/npm/bin:${containerEnv:PATH}"
  },
  "customizations": {
    "vscode": {
      "extensions": [
        "Google.gemini-cli-vscode-ide-companion"
      ]
    }
  }
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/8.4-gemini_cli

## 8.5. Copilot CLI

### DevContainerの設定

```bash
#!/bin/bash
npm config set prefix ~/npm
npm install -g @github/copilot
```

```json
{
  "name": "copilot-cli",
  "build": {
    "dockerfile": "./Dockerfile"
  },
  "postCreateCommand": ".devcontainer/postCreateCommand.sh",
  "mounts": [
    // 永続化
    "source=${localEnv:HOME}/.llm/copilot/.copilot,⏎
     target=/home/node/.copilot,type=bind,consistency=cached",
    "source=${localEnv:HOME}/.llm/copilot/.config/configstore,⏎
     target=/home/node/.config/configstore,type=bind,consistency=cached"
  ]
}
```

> https://github.com/74th/devcontainer-book-2nd-samples/tree/main/8.5-copilot_cli

## 8.6. Container Useを使う

> Container Use<br/>https://container-use.com

```json
{
  "workdir": "/workdir",
  "base_image": "mcr.microsoft.com/devcontainers/base:latest",
  "setup_commands": [
    "curl -LsSf https://astral.sh/uv/install.sh | sh",
    "sudo ln -sf $HOME/.local/bin/uv /usr/bin/uv"
  ]
}
```

> Agent-Integrations<br/>https://container-use.com/agent-integrations

> https://github.com/74th/devcontainer-book-2nd-samples-container-use

# 9. クラウドLLMコーディングエージェントの環境セットアップ方法

## 9.1. GitHub Copilot Coding Agent

> GitHub Copilot コーディング エージェント用の開発環境のカスタマイズ<br/>https://docs.github.com/ja/copilot/how-tos/use-copilot-agents/coding-agent/customize-the-agent-environment

```yaml
# .github/workflows/copilot-setup-steps.yml
name: "Copilot Setup Steps"

on:
  workflow_dispatch:
  push:
    paths:
      - .github/workflows/copilot-setup-steps.yml
  pull_request:
    paths:
      - .github/workflows/copilot-setup-steps.yml

jobs:
  copilot-setup-steps:
    runs-on: ubuntu-latest
    permissions:
      contents: read

    steps:
      - name: Checkout code
        uses: actions/checkout@v5

      # uvの公式GitHub Actionを使ってインストール
      - name: Install uv
        uses: astral-sh/setup-uv@v6
```

## 9.2. Codex

> https://github.com/openai/codex-universal

```bash
#!/bin/bash
# setup_script_for_codex.sh
curl -LsSf https://astral.sh/uv/install.sh | sh
```

```bash
./setup_script_for_codex.sh
```

## 9.3. Jules

> Environment setup | Jules<br/>https://jules.google/docs/environment/

# おわりに

## 著者

- [X(旧Twitter)](https://twitter.com/74th)、[GitHub](https://github.com/74th): 74th
- [BOOTH https://74th.booth.pm/に電子工作キットも出しています](https://74th.booth.pm/)
- 制作物ショーケース https://showcase.74th.tech/

#### 著書

- [商業誌『改訂新版 Visual Studio Code 実践ガイド』（技術評論社, 2024）](https://gihyo.jp/book/2024/978-4-297-13909-4)
- [同人誌『CH32V003カイハツガイドブック』技術書典 18](https://74th.booth.pm/items/6934072)
- [同人誌『自宅IoTリモコン全部作る』技術書典 17](https://74th.booth.pm/items/6201064)
- [同人誌『USB完全に理解した−通信仕様とコントローラ編−』技術書典16](https://74th.booth.pm/items/5826037)
- [同人誌『WCHのICを活用する電子工作の本』技術書典15](https://74th.booth.pm/items/5261331)
- [同人誌『マイコンさんに知らないプロトコルを喋らせる技術』技術書典14](https://74th.booth.pm/items/4799571)
- [同人誌『土曜日の Raspberry Pi Pico』技術書典13](https://74th.booth.pm/items/4161550)
- [同人誌『VS Code デバッグ技術 2nd Edition』技術書典11](https://74th.booth.pm/items/3338895)

#### 製作ガジェット・電子工作ツール

- [SparrowS v3: 左右分割自作キーボードキット](https://74th.booth.pm/items/6655442)
- [SparrowDial: M5StackCore2、M5Dialをトラックパッドとして搭載可能なGH60互換ケース対応自作キーボードキット](https://74th.booth.pm/items/5525751)
- [RP2040 Large: RP2040の手はんだ実装に挑戦する開発ボードキット](https://74th.booth.pm/items/3929664)
- [CH32V003 ProMicro Like: CH32V003 ProMicroサイズ開発ボードキット](https://74th.booth.pm/items/4645948)
- [USB-C Solder Tester v2: Type-Cソケット実装を確認するツール](https://74th.booth.pm/items/5812941)
