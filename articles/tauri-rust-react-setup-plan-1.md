---
title: "Tauriを用いた開発での、もろもろセットアップについて"
emoji: "🖥️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tauri", "rust", "react", "typescript", "bun", "sqlite", "tailwindcss"]
published: true
---

## はじめに

この記事は、[Tauri](https://tauri.studio/)を用いた開発における、もろもろセットアップについての記事です。
基本的には上記サイトの[必須要件以降](https://tauri.app/v1/guides/getting-started/prerequisites)を参考にすれば良いので、詰まった点などについて記載していきます。

  

## プロジェクト全般、Bunあたりの設定
  
### ホットリロードの設定

#### Rust側の設定（SQLiteを選択した場合）

DBにSQLiteを使う場合、ホットリロードを有効にするとDBファイルの更新によって、ホットリロードが乱発してしまう。
そのため、***.taurignore***を作成します。
この[GitHub Issue](https://github.com/tauri-apps/tauri/issues/4617)を参考にしました。
例えば、プロジェクトルートを

```
Application
```

だとすると、

```
Application/src-tauri
```

がRustのプロジェクトのルートになるので、

```
Application
├── src-tauri
└── src
```


driverというディレクトリにDBに関する機能を置き、driver/db配下にSQLiteのDBファイルを配置すると仮定すると、
```path
/Application
├── /src-tauri (Rust)
│   ├── /src
│   │   ├── /driver
│   │   │   ├── /db
│   │   │   │   ├── /data
│   │   │   │   │   ├── /db.sqlite
│   │   │   │   └── /initial_setup
│   │   │   └── main.rsとか
│   │   ├── Cargo.toml
│   │   ├── tauri.conf.json
│   │   └── .taurignore
└── /src (UI & Front)
```

と ***.taurignore***ファイルを作成します。
ファイル内には、以下のように記述します。  

```ignorelang
# .taurignore
/src/driver/db/data/*
```

これによりホットリロード時に、DBファイルの更新が発生して約１秒おきにリロードされて画面がちらつく問題が解決できました。  


### React、フロントエンド側（Tailwind CSS）の場合  

#### Tailwind CSSの設定  

```path
/Application
├── /src-tauri (Rust)
└── /src (UI & Front)
```

Application配下に通常のReactプロジェクト同様に実際のコードを記述していく形になり、package.jsonなども通常通りにApplicationの直下に生成されます。

全体像は以下のようになります。  

```path
/Application
│
├── /dist
├── /node_modules
├── /public
│
├── /src
│   └── (contents of /src not listed)
│
├── /src-tauri
│   ├── /src
│   │   └── (contents of /src-tauri/src not listed)
│   ├── Cargo.toml
│   ├── tauri.conf.json
│   └── .taurignore
│
├── package.json
├── package-lock.json
├── tsconfig.json
└── vite.config.ts

```

といった感じです。
TauriでReactを使った場合のディレクトリ構成で特徴的なのは、ただ ***/src-tauri***がRust自体のプロジェクトルートとなる点です。

では、本題のTailwind CSSの設定について。
Tailwind CSSの設定は他にも記事がたくさんあるので、そちらを参考にしていただくとして、問題は以下のコマンド実行時のホットリロードが機能するかだと思います。

```bash
cargo tauri dev
```

このコマンドでフロントエンド、つまりUIを含めたReact側とRust側の両方のコードをホットリロードできます。
しかし、Tailwind CSSの設定をする際には、CSSのコンパイルが必要になるので、以下のように ***npm run dev*** と ***npm run build*** コマンドを変更します。

```json
{
  "scripts": {
    "dev": "bun tailwindcss -i ./src/input.css -o ./dist/output.css && vite ",
    "build": "bun tailwindcss -i ./src/input.css -o ./dist/output.css && tsc && vite build",
  }
}
```

これにより、CSSのコンパイルが ***cargo tauri dev*** コマンドを実行した際に走るようになります。

### その他

#### Rustのマルチクレート化の方法について


「マルチクレート化」という表現が正しいのかわからないですが、要はモジュラモノリスのような形で、Rustのプロジェクトを分割していきたいシナリオで考えます。

例えば、以下のような構成にしたいときに役立ちます。  

```path
.
├── build.rs
├── Cargo.lock
├── Cargo.toml
├── src
│   ├── domain
│   │   ├── Cargo.lock
│   │   ├── Cargo.toml
│   │   └── src
│   │       ├── domain.rs
│   │       ├── lib.rs
│   │       └── rss_feed_site.rs
│   ├── driver
│   │   ├── Cargo.toml
│   │   ├── db
│   │   │   ├── data
│   │   │   └── initial_setup
│   │   └── src
│   │       ├── lib.rs
│   │       ├── register_driver.rs
│   │       └── sqlite_driver.rs
│   ├── main.rs
│   └── mod.rs
└── tauri.conf.json

```

このような構成にしたい場合は、各ライブラリを生成して（ *lib.rsが存在することが肝*）、プロジェクトルートの***Cargo.toml*** に以下のように記述します。


```toml
[workspace]
members = [
  "src/domain",
  "src/driver",
]

[dependencies]
## other dependencies
anyhow = "1.0"
futures = "0.3"
dotenv = "0.15.0"

## your dependencies
domain = { path = "src/domain", version = "0.1.0" }
driver = { path = "src/driver", version = "0.1.0" }
```

そして、例えばdomainをdriverから使いたい場合には、driverのCargo.tomlを以下のようにします。


```toml

[dependencies]
domain = { path = "../domain" }
```

これにより、domainをdriverから使うことができるようになるかと思います。  


### 以上

一旦これで今回の記事は終わりにします。
追加で便利なことや必須の情報が判明したら、追記するか別記事にする予定です。

