---
title: LaravelもViteもVueもわかんないから、デプロイまで簡単にまとめる
tags:
  - Laravel
  - Docker
  - Vue.js
  - vite
private: false
updated_at: '2024-12-03T22:13:47+09:00'
id: c484c8fda52872fc36d2
organization_url_name: null
slide: false
ignorePublish: false
---

# この記事で書くこと

## Laravel11 + Vite (Vue.js) のインストール

> フォルダの作成

```
myapp(プロジェクトフォルダ)
├── docker
│   ├── app
│   │   ├── Dockerfile
│   │   └── php.ini
│   ├── db
│   │
│   └── nginx
│       ├── Dockerfile
│       ├── default.conf
│       └── logs
├── docker-compose.yaml
└── my.cnf
```

- こうなります 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/532dc358-7885-291b-12d2-b957c7cadc3d.jpeg" width="200px">

### Docker, docker-compose, 各 conf

> docker-compose の編集

```yaml:docker-compose.yaml
services:
  nginx:
    container_name: "nginx"
    build:
      context: ./docker/nginx
    depends_on:
      - app
    ports:
      - 80:80
    volumes:
      - ./:/src

  app:
    container_name: "app"
    build:
      context: ./docker/app
    depends_on:
      - db
    ports:
      - 5173:5173
    volumes:
      - ./:/src
      - /src/node_modules
      - /src/vendor
      - ./docker/app/php.ini:/usr/local/etc/php/php.ini

  db:
    image: mysql:8.0.37
    container_name: "db"
    volumes:
      - ./docker/db:/var/lib/mysql
      - ./my.cnf:/etc/mysql/conf.d/my.cnf
    environment:
      - MYSQL_ROOT_PASSWORD=root
      - MYSQL_DATABASE=myapp
    ports:
      - 3306:3306

  redis:
    image: redis:alpine
    container_name: "redis"
    ports:
      - 16379:6379
```

- 高速化のため以下のフォルダはマウントから<b>除外</b>（これをしないと vite のロードに 1 分かかる）
  - /src/node_modules
  - /src/vendor
- そのため docker-compose down のたびに下記コマンド必須。

  - composer install
  - npm install

> my.cnf の編集

```conf:./my.cnf
[mysqld]
max_allowed_packet = 32505856
lower_case_table_names = 2
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci

[client]
default-character-set=utf8mb4
```

> nginx の Dockerfile

```docker:docker/nginx/Dockerfile
FROM nginx:1.27
COPY ./default.conf /etc/nginx/conf.d/default.conf
```

> nginx の default.conf

```conf:docker/nginx/default.conf
server {

    listen 80;
    server_name _;

    client_max_body_size 1G;

    root /src/public;
    index index.php;

    access_log /src/docker/nginx/logs/access.log;
    error_log  /src/docker/nginx/logs/error.log;

    location / {
        try_files $uri $uri/ /index.php$is_args$args;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass app:9000;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param PATH_INFO $fastcgi_path_info;
    }

}
```

> app の dockerfile

```docker:docker/app/Dockerfile
FROM php:8.3-fpm
EXPOSE 5173
RUN apt-get update \
  && apt-get install -y \
  git \
  zip \
  unzip \
  vim \
  libfreetype6-dev \
  libjpeg62-turbo-dev \
  libmcrypt-dev \
  libpng-dev \
  libfontconfig1 \
  libxrender1

RUN docker-php-ext-configure gd --with-freetype --with-jpeg
RUN docker-php-ext-install gd
RUN docker-php-ext-install bcmath
RUN docker-php-ext-install pdo_mysql mysqli exif
RUN cd /usr/bin && curl -s http://getcomposer.org/installer | php && ln -s /usr/bin/composer.phar /usr/bin/composer

ENV NODE_VERSION=20.15.0
RUN curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
ENV NVM_DIR=/root/.nvm
RUN . "$NVM_DIR/nvm.sh" && nvm install ${NODE_VERSION}
RUN . "$NVM_DIR/nvm.sh" && nvm use v${NODE_VERSION}
RUN . "$NVM_DIR/nvm.sh" && nvm alias default v${NODE_VERSION}
ENV PATH="/root/.nvm/versions/node/v${NODE_VERSION}/bin/:${PATH}"
RUN node --version
RUN npm --version

WORKDIR /src
ADD . /src/storage
RUN chown -R www-data:www-data /src/storage
```

> app の php.ini の編集

```ini:docker/app/php.ini
upload_max_filesize=256M
post_max_size=256M
```

### Laravel, 初回 migrate

> app の起動

```bash
> docker compose up -d
```

- file not found が正しいです 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/f974f84c-123d-be77-e883-dff596cff76c.jpeg" width="200px">

> 一時フォルダを作成して Laravel をインストール

```bash
> docker compose exec app bash
```

```bash
src# mkdir temp
src# cd temp
src/temp# composer create-project "laravel/laravel=11.*" . --prefer-dist
```

- 画面上はこうなってます 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/db915309-0123-1e72-dd98-ef9b5c753fba.jpeg" width="400px">

> 中身をプロジェクトフォルダに移動して、一時フォルダを削除する

```bash
src/temp# cd ..
src# mv l11dev_tmp/* ./
src# mv l11dev_tmp/.* ./
src# rm l11dev_tmp -rf
```

> 依存関係のインストール

```bash
src# composer install
src# npm install
```

- フォルダ構成 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/b99a2850-ec60-5f93-6b89-d9e7006595b3.jpeg" width="200px">

> The stream or file "/src/storage/logs/laravel.log" could not be opened in append mode の防止

- root 以外に www-data が書き込みできるように変更
- Laravel の公式に指定された、strage のシンボリックリンクを設置

```bash
src# chmod -R guo+w storage
src# php artisan storage:link
```

> DB との接続

- env ファイルでコメントアウトされている箇所を変更

```env:.env
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=myapp
DB_USERNAME=root
DB_PASSWORD=root
```

- .env 上記の変更箇所 👇️
<div>
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/3b291fcd-9d0d-478e-986d-afc09069cd9d.jpeg" width="200px">
<div></div>
<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/cdc3ed56-a1e9-3c14-5d9d-1d6a8b3b9725.jpeg" width="200px">
</div>

- migration

```bash
src:# php artisan migrate
```

> アクセステスト

- 共通 #src npm run dev をしてからアクセステスト。
- 共通 Ctrl + C で抜けてから構築作業

* http://localhost にアクセスしてみる
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/33224b2a-84aa-3000-20dd-471e245107f9.jpeg" width="400px">

### Vite

> welcome.blade.php の確認

- welcome.blade.php に以下の記述があるか確認 -> ない場合は/head の前に記述する

```php:resources/views/welcome.blade.php
@vite(['resources/css/app.css', 'resources/js/app.js'])
```

- この記述を確認する 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/4f858d97-7b4f-4a04-3aa5-6c87253533e3.jpeg" width="400px">

> vite.config.js の変更

- 以下の画像の位置にコードを追加する

```js:vite.config.js
    server: {
        host: true,
        hmr: {
            host: 'localhost',
        },
    },
```

- 上記の位置 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/8f04ef79-e8e1-b196-1b4d-04026aa5ca3a.jpeg" width="400px">

> Vite の起動

```bash
/src# npm run dev
```

- http://localhost にアクセス 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/a7c53e83-569a-1116-92bd-e2b67825fdba.jpeg" width="400px">

> 設定を変えて再起動するとき

```bash
> docker compose down
> docker compose up -d --build
> docker compose exec app bash
src:# composer install
src:# npm install
```

> vue のインストール

```bash
src:# npm install vue @vitejs/plugin-vue
```

> app.js の編集

```js:resources/js/app.js
import "./bootstrap";
import { createApp } from "vue";
import ExampleComponent from './components/ExampleComponent.vue';

const app = createApp({});
app.component('example-component', ExampleComponent);
app.mount('#app');
```

> resources/js/components/ExampleComponent.vue 　の作成

```vue:resources/js/components/ExampleComponent.vue
<template>
    <div>
        <h1>Hello, Vue with Vite!</h1>
    </div>
</template>

<script>
export default {
    name: 'ExampleComponent',
};
</script>

<style scoped>
h1 {
    color: blue;
}
</style>
```

> welcome.blade.php の編集

- body の中をすべて削除して記入

```php:resources/views/welcome.blade.php
<body>
    <div id="app">
        <example-component></example-component>
    </div>
</body>
```

> Vue の動作確認

- vite.config.js の編集

```js:vite.config.js
import { defineConfig } from "vite";
import laravel from "laravel-vite-plugin";
import vue from "@vitejs/plugin-vue";

export default defineConfig({
    plugins: [
        laravel({
            input: ["resources/css/app.css", "resources/js/app.js"],
            refresh: true,
        }),
        vue(),
    ],
    resolve: {
        alias: {
            vue: "vue/dist/vue.esm-bundler.js",
        },
    },
    server: {
        host: true,
        hmr: {
            host: "localhost",
        },
    },
});
```

> http://localhost の確認

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/1bfea4de-85a7-7a00-2b84-e2d0018072e5.jpeg" width="400px">

## Laravel の MVC

### Controller, View, web.php(routing)

> controller の作成

```bash
/src# php artisan make:controller SampleController
```

- controller の編集

```php:app/Http/Controllers/SampleController.php
namespace App\Http\Controllers;

use Illuminate\Http\Request;

class SampleController extends Controller
{
    public function index()
    {
        return view('sample'); // resources/views/sample.blade.php
    }
}
```

> view の作成

```bash
src# touch resources/views/sample.blade.php
```

- view の編集

```php:resources/views/sample.blade.php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Sample Page</title>
</head>
<body>
    <h1>Hello from Sample View!</h1>
</body>
</html>
```

> web.php の編集

```php:routes/web.php
<?php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\SampleController;

Route::get('/', function () {
    return view('welcome');
});

Route::resource('samples', SampleController::class);
```

- これで自動的にルートが生成される 👇️
  <img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/9c35fbb2-4e38-22a3-232c-0e6d05577df0.jpeg" width="400px">

> http://localhost/samples にアクセス

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/7a7058e0-7569-56a2-e59f-c84e0c1c6b22.jpeg" width="400px">

### migration, seeder

> migration ファイルの作成

- create\_複数形\_table

```bash
src# php artisan make:migration create_items_table
```

- migration ファイルの編集(title は 100 文字制限)

```php:database/migrations/2024_11_30_123816_create_items_table.php
...
public function up(): void
{
    Schema::create('items', function (Blueprint $table) {
        $table->id();
        $table->string('title', 100);
        $table->text('content');
        $table->timestamps();
    });
}
...
```

> migration

```bash
src# php artisan migrate
```

> Seeder ファイルの作成

```bash
src# php artisan make:seeder ItemsTableSeeder
```

- Seeder ファイルの編集

```php:database/seeders/ItemsTableSeeder.php
...
use Illuminate\Support\Facades\DB;

class itemsTableSeeder extends Seeder
{
  public function run(): void
  {
      DB::table('items')->insert([
          'title' => 'タイトル１',
          'content' => 'テストデータ１',
      ],
      [
          'title' => 'タイトル２',
          'content' => 'テストデータ２',
      ]);
  }
...
}
```

- DatabaseSeeder への登録

```php:database/seeders/DatabaseSeeder.php
...
public function run(): void
{
    $this->call(ItemsTableSeeder::class);
    ...
}
...
```

> seeder の実行

```bash
src# php artisan db:seed
```

### Model

> model の作成

```bash
src# php artisan make:model Item
```

app/Models/item.php 　が生成される

> db からデータの取り出し

```php:app/Http/Controllers/SampleController.php
...
use App\Models\Item;

class SampleController extends Controller
{
  public function index(Request $request)
  {
    ...
    $title = Item::find(1)->title;
    return view("sample", ["title" => $title]);
    ...
  }
...
}
```

> sample.blade.php の編集

```php:resources/views/sample.blade.php
...
<body>
    <h1>Hello from Sample View!</h1>
    <?= $title ?>
</body>
...
```

> http://localhost/samples 　にアクセス

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/aff8d788-5221-2818-e3c1-84c8e777d310.jpeg" width="400px">

## AWS のデプロイ

### EC2 起動, security-group

> EC2 を開く

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/bd214860-8734-d68e-ecfb-4ae608340e84.jpeg" width="400px">

<img src="https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/48b75fcd-3d97-3de3-fe83-76ea32340f73.jpeg" width="400px">

![スクリーンショット 2024-12-02 12.09.27.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/e803307b-b427-5189-309c-d4ba0d985545.jpeg)

![スクリーンショット 2024-12-02 12.09.36.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/a1ad6115-0157-02da-df86-95aa189d3895.jpeg)

> 途中キーペアを作成した場合はダウンロードしておく（既存のでも OK）

![スクリーンショット 2024-12-02 12.09.52.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/d7efed7d-526e-dbc5-ec4e-5fa06a4b005b.jpeg)

> 起動したインスタンスのパブリック IPv4DNS を確認

![スクリーンショット 2024-12-02 12.13.16.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/be3b4831-316c-ace8-0f8b-e73f53e422c9.jpeg)

> ssh 接続

- キーペアファイルはよく使うのでなくさないように保存
- 保存したフォルダに cd していれば、フルパスじゃなくて、キーペアファイル名のみで OK

```bash
> ssh ec2-user@<パブリック IPv4DNSアドレス> -i <キーペアファイルまでのフルパス>
```

> docker のインストール

```bash
sudo su
sudo dnf update
sudo dnf install -y docker
sudo systemctl start docker
sudo gpasswd -a $(whoami) docker
sudo chgrp docker /var/run/docker.sock
sudo service docker restart
sudo systemctl enable docker

sudo mkdir -p /usr/local/lib/docker/cli-plugins
sudo curl -SL https://github.com/docker/compose/releases/download/v2.21.0/docker-compose-linux-x86_64 -o /usr/local/lib/docker/cli-plugins/docker-compose
sudo chmod +x /usr/local/lib/docker/cli-plugins/docker-compose

mkdir -p /work/docker
sudo chown -R ec2-user /work/docker
sudo chmod -R 775 /work/docker
```

### sftp

> プロジェクトのアップロード

- プロジェクトを zip 化して、c/projects/docker に置く

```bash
sftp -oPort="22" -oIdentityfile="<キーペアのフルパス>" <パブリックIPv4DNS>
sftp> cd /work/docker
sftp> lcd ~/projects/docker
sftp> put myapp.zip
```

### install

```bash
ssh ec2-user@<インスタンスのパブリックDNS> -i <キーペアファイル>
cd /work/docker
sudo su
unzip myapp.zip
rm -rf __MACOSX
rm -rf <zip ファイル>
cd myapp
docker compose up -d

docker compose exec app bash
src# composer install
src# npm install
src# npm run build
```

> public アドレスにアクセスして表示確認

![スクリーンショット 2024-12-03 21.37.14.jpg](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/3822203/f8c7a9cd-6a24-af1b-8b6b-0a2ff5e2da4b.jpeg)

# 参考にした記事

https://qiita.com/hitotch/items/2e816bc1423d00562dc2

https://qiita.com/Masahiro111/items/f6201b1e89fb6cfddc09
