# 概要

React.jsで複数アプリ（ドメイン）をホストさせる方法

## 注意事項

- 構成管理が複雑になるので、理由がなければプロジェクト単位でアプリ分けた方がいいっすよ。

## 基本知識

React.jsの開発はcreate-react-appで一気に開発環境作ってくれる。これはWebpack等の必要な設定もしてくれる。
要約すると面倒な設定を一括自動で行っている。。そのためプロジェクト作成の基本となるケースが多い。

さて、気になるのは一括設定はどこにあるのか？である。
create-react-appの実行結果にはそれらしい設定ファイルが一切見当たらない。

```
yarn eject
```

上記のコマンド実行後、設定ファイルが出力される。
**しかし元に戻せない。** いわゆる「良い感じ」に動いている部分を全て自分で設定する必要がある。
本当に自分たちで保守が必要か見極めて、必要に迫られてから実行すること。

## 手順

### 環境構築

- npm、yarn、npxのうち好きなやつを使えるようにしておく

```
yarn create react-app プロジェクト名
cd プロジェクト名
yarn start
```

- http://localhost:3000　が開けること

### eject

```
yarn eject
```

プロジェクト直下に `config` フォルダが生成される。

### 複数アプリ作成

例として、一般ユーザ向けアプリ、管理者向けアプリを作成する。

`src` フォルダを以下のように編集。各ファイルのはデフォルトのものをコピっただけ。

```
src
 - admin
     - App.css
     - App.js
     - App.test.js
     - index.css
     - index.js
     - logo.svg
     - serviceWorker.js
     - setupTests.js
 - user
     - App.css
     - App.js
     - App.test.js
     - index.css
     - index.js
     - logo.svg
     - serviceWorker.js
     - setupTests.js
```

差分が確認したい場合は `App.js` の中身を書き換えておく。

`public` フォルダを以下のように編集。各ファイルのはデフォルトのものをコピっただけ。

```
public
 - admin
     - favicon.ico
     - index.html
     - logo192.png
     - 1ogo512.png
     - manifest.json
     - robots.txt
 - user
     - favicon.ico
     - index.html
     - logo192.png
     - 1ogo512.png
     - manifest.json
     - robots.txt
```

`config/paths.js` の中身を以下のように編集

```
// config after eject: we're in ./config/

const ENTRY_NAME = process.env.ENTRY_NAME || 'user'  //追記

module.exports = {
  dotenv: resolveApp('.env'),
  appPath: resolveApp('.'),
  appBuild: resolveApp('build/' + ENTRY_NAME), //編集
  appPublic: resolveApp('public/' + ENTRY_NAME), //編集
  appHtml: resolveApp(`public/${ENTRY_NAME}/index.html`), //編集
  appIndexJs: resolveModule(resolveApp, `src/${ENTRY_NAME}/index`), //編集
  appPackageJson: resolveApp('package.json'),
  appSrc: resolveApp('src'),
  appTsConfig: resolveApp('tsconfig.json'),
  appJsConfig: resolveApp('jsconfig.json'),
  yarnLockFile: resolveApp('yarn.lock'),
  testsSetup: resolveModule(resolveApp, 'src/setupTests'),
  proxySetup: resolveApp('src/setupProxy.js'),
  appNodeModules: resolveApp('node_modules'),
  publicUrlOrPath,
};
```

`package.json` を以下のように編集

```
  "scripts": {
    "start": "node scripts/start.js",
    "build": "node scripts/build.js",
    "test": "node scripts/test.js",
    "start:admin": "ENTRY_NAME='admin' PORT=3001 node scripts/start.js",
    "build:admin": "ENTRY_NAME='admin' node scripts/build.js",
    "test:admin": "ENTRY_NAME='admin' node scripts/test.js"
  },
```

### 動作確認

ターミナルを2つ用意。

ターミナル1
```
コマンド：yarn start
Compiled successfully!

You can now view プロジェクト名 in the browser.

  Local:            http://localhost:3000
  On Your Network:  http://192.168.3.2:3000

Note that the development build is not optimized.
To create a production build, use yarn build.

```

ターミナル2
```
コマンド：yarn start:admin

Compiled successfully!

You can now view プロジェクト名 in the browser.

  Local:            http://localhost:3001
  On Your Network:  http://192.168.3.2:3001

Note that the development build is not optimized.
To create a production build, use yarn build.
```

それぞれアクセスしてみて別の画面が出ること。

- http://localhost:3000
- http://localhost:3001

## リリース

さて開発環境はうまく動かせることがわかった。しかし、このまま2つデプロイしようとしたらどうなるのか？

dockerを使い、nginxを例に解説する。

ざっくり説明すると、
1. buildしたアプリを配備しnginxのドキュメントルートに設定
2. アプリの数だけバーチャルホスト設定

### 設定ファイル

プロジェクト直下に以下のように作成

※サンプルなので、配置が気に入らねえって場合は適宜変更すること

```
nginx
 - default.conf
docker-compose.yml
```

`nginx/default.conf` を編集

```
server {
    listen       80;
    server_name admin.local;

    location / {
        root   /var/www/admin;
        index  index.html index.htm;
        try_files $uri /index.html;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}

server {
    listen       80;
    server_name user.local;

    location / {
        root   /var/www/user;
        index  index.html index.htm;
        try_files $uri /index.html;
    }
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

バーチャルホスト設定でadmin.local、user.localに2つのアプリを配備させる。

`docker-compose.yml` を編集

```
version: '3'
services:
  nginx:
    image: nginx
    container_name: nginx
    ports:
      - 8080:80
    volumes:
      - ./build:/var/www  #buildしたファイルをdockerのnginx内に反映
      - ./nginx/:/etc/nginx/conf.d/ #ローカルの設定ファイルをnginxに反映
```

### 名前解決

自PCのhostsに追加（名前解決できればなんでもOK

```
127.0.0.1 admin.local
127.0.0.1 user.local
```

### デプロイ

```
docker-compose build
docker-compose up
```

それぞれアクセスして別画面が出ること。

- http://admin.local:8080/
- http://user.local:8080/
