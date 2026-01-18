---
title: "Hugo でブログを作ってみた"
description: 
slug: create-hugo-blog
date: 2026-01-11T21:18:16+09:00
image: 
categories:
  - Hugo
tags: [Hugo]
---

## はじめに

ブログを作りたかったけどフロントエンド云々が分からないので、
そういった技術に疎くても編集しやすく、かつデザインがよさそうなテンプレートを探していました。
そうして出会った [Hugo](https://gohugo.io/) の [Stack](https://themes.gohugo.io/themes/hugo-theme-stack/) テーマが気に入ったので、
ブログを構築してみました。

本記事はその備忘録です。OS とアーキテクチャは Ubuntu 24.04, amd64 を想定していますので、それ以外の環境の場合は適宜読み替えていただければと思います。

## やったこと

### Hugo のインストール

`apt install` 等してもよいですがバージョンが古いため、今回は[リリースページ](https://github.com/gohugoio/hugo/releases/)から最新版 (執筆時点では v0.154.4) の `.deb` をダウンロードしてきてインストールする方法をとりました。バージョン違いやアーキテクチャ違い (arm64 など) は適宜読み替えてください。

Hugo には通常版と extended 版がありますが、必要な機能の都合上 extended 版を選択します。

```bash
wget https://github.com/gohugoio/hugo/releases/download/v0.154.4/hugo_extended_0.154.4_linux-amd64.deb
sudo apt install ./hugo_extended_0.154.4_linux-amd64.deb
rm ./hugo_extended_0.154.4_linux-amd64.deb
```

### ブログ用ディレクトリの初期化

Hugo 公式の [Quick start](https://gohugo.io/getting-started/quick-start/) をベースに、ブログ用ディレクトリを作成したり、テーマの設定をしたりします。

まず、ブログ用ディレクトリを作成します。`<sitename>` は任意のディレクトリ名です。

```bash
hugo new site <sitename>
cd <sitename>
git init
```

次に Stack をテーマとして設定します。

```bash
git submodule add https://github.com/CaiJimmy/hugo-theme-stack/ themes/hugo-theme-stack
```

テーマの中に設定ファイルやコンテンツの例があるので、それを `<sitename>` にコピーします。ただし、のちの便宜のために `hugo.yaml` は `config.yaml` にリネームしておきます (後の作業で異なるディレクトリに `hugo.yaml` ファイルが現れるため紛らわしいと思ったが故の措置であり、必須ではありません)。
また、元々存在していた `hugo.toml` は不要なので削除します。

```bash
cp -r \
  themes/hugo-theme-stack/exampleSite/hugo.yaml \
  themes/hugo-theme-stack/exampleSite/content \
  themes/hugo-theme-stack/archetypes \
  .
mv ./hugo.yaml ./config.yaml
rm ./hugo.toml
```

### `config.yaml` の編集

[こちらの記事](https://miiitomi.github.io/p/hugo/) の config の節を参考に、 `config.yaml` を編集します。

### コンテンツの整理と初期設定

やはり [こちらの記事](https://miiitomi.github.io/p/hugo/) のコンテンツの節を参考に作業します。
`<sitename>/content` には `hugo-theme-stack/exampleSite/content` 由来のテスト記事・テストカテゴリーがありますが、不要であれば (基本的には不要だと思いますが) 削除します。
また、About や記事のメタデータのフォーマットを編集します。

### テスト記事の作成とビルド

`<sitename>` ディレクトリにおいて、以下のコマンドで記事 `my-first-post` を作成します。

```bash
hugo new content content/posts/my-first-post.md
```

その後、ビルドを行い、`localhost:1313` からサーバーにアクセスし、作成した記事が適切に生成されていることを確認します。

```bash
hugo server
```

ここまでの作業が終わったら、 git commit と Github リポジトリへの push を行ってもよいかもしれません。

### Github Pages へのデプロイ

Github Pages にデプロイするために、従来はブランチを利用する方法 (サードパーティの `peaceiris/actions-hugo` や `peaceiris/actions-gh-pages` を利用する方法) が用いられていたようですが、ここでは Github Actions (`actions/upload-pages-artifact` や `actions/deploy-pages`) を用いたデプロイを行います。

手順は Hugo 公式の [Host on Github Pages](https://gohugo.io/host-and-deploy/host-on-github-pages/) に従いますが、具体的に行ったことを書いておきます。

まず Github リポジトリ `<github-username>.github.io` を作成し (`<github-username>` は Github のアカウント名)、Setting からサイドバー内の Pages を選択し、Build and deploy の Source を Github Actions に設定しておきます (設定は自動保存されます)。

続いて `<sitename>/config.yaml` に以下の記述を追加します。

```yaml
caches:
  assets:
    dir: :resourceDir/_gen
    maxAge: -1
  getcsv:
    dir: :cacheDir/:project
    maxAge: -1
  getjson:
    dir: :cacheDir/:project
    maxAge: -1
  getresource:
    dir: :cacheDir/:project
    maxAge: -1
  images:
    dir: :cacheDir/images
    maxAge: -1
  misc:
    dir: :cacheDir/:project
    maxAge: -1
  modules:
    dir: :cacheDir/modules
    maxAge: -1
```

さらに `.github/workflows/hugo.yaml` を作成します。

```bash
mkdir -p .github/workflows
touch .github/workflows/hugo.yaml
```

そして、作成した `hugo.yaml` に以下をそのまま記載します。

```yaml
name: Build and deploy
on:
  push:
    branches:
      - main
  workflow_dispatch:
permissions:
  contents: read
  pages: write
  id-token: write
concurrency:
  group: pages
  cancel-in-progress: false
defaults:
  run:
    shell: bash
jobs:
  build:
    runs-on: ubuntu-latest
    env:
      DART_SASS_VERSION: 1.97.1
      GO_VERSION: 1.25.5
      HUGO_VERSION: 0.154.2
      NODE_VERSION: 24.12.0
      TZ: Asia/Tokyo
    steps:
      - name: Checkout
        uses: actions/checkout@v5
        with:
          submodules: recursive
          fetch-depth: 0
      - name: Setup Go
        uses: actions/setup-go@v5
        with:
          go-version: ${{ env.GO_VERSION }}
          cache: false
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
      - name: Setup Pages
        id: pages
        uses: actions/configure-pages@v5
      - name: Create directory for user-specific executable files
        run: |
          mkdir -p "${HOME}/.local"
      - name: Install Dart Sass
        run: |
          curl -sLJO "https://github.com/sass/dart-sass/releases/download/${DART_SASS_VERSION}/dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          tar -C "${HOME}/.local" -xf "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          rm "dart-sass-${DART_SASS_VERSION}-linux-x64.tar.gz"
          echo "${HOME}/.local/dart-sass" >> "${GITHUB_PATH}"
      - name: Install Hugo
        run: |
          curl -sLJO "https://github.com/gohugoio/hugo/releases/download/v${HUGO_VERSION}/hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          mkdir "${HOME}/.local/hugo"
          tar -C "${HOME}/.local/hugo" -xf "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          rm "hugo_extended_${HUGO_VERSION}_linux-amd64.tar.gz"
          echo "${HOME}/.local/hugo" >> "${GITHUB_PATH}"
      - name: Verify installations
        run: |
          echo "Dart Sass: $(sass --version)"
          echo "Go: $(go version)"
          echo "Hugo: $(hugo version)"
          echo "Node.js: $(node --version)"
      - name: Install Node.js dependencies
        run: |
          [[ -f package-lock.json || -f npm-shrinkwrap.json ]] && npm ci || true
      - name: Configure Git
        run: |
          git config core.quotepath false
      - name: Cache restore
        id: cache-restore
        uses: actions/cache/restore@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: hugo-${{ github.run_id }}
          restore-keys:
            hugo-
      - name: Build the site
        run: |
          hugo \
            --gc \
            --minify \
            --baseURL "${{ steps.pages.outputs.base_url }}/" \
            --cacheDir "${{ runner.temp }}/hugo_cache"
      - name: Cache save
        id: cache-save
        uses: actions/cache/save@v4
        with:
          path: ${{ runner.temp }}/hugo_cache
          key: ${{ steps.cache-restore.outputs.cache-primary-key }}
      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: ./public
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    needs: build
    steps:
      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

その後 git commit および Github リポジトリへの push を行えば、デプロイが行われます。デプロイ結果の詳細は Github リポジトリ `<github-username>.github.io` のページの Actions から見ることができます。

## おわりに

本記事では Hugo + Stack テーマでブログを構築する手順を説明しました。

こんなに手軽に、いい感じのブログが作れるなんてすごいなあ、と思いながら作業していました。

これから記事を書いていくのが楽しみですね～～
