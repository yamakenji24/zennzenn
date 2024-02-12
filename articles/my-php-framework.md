---
title: "FEエンジニアがPHPでRoutingとDiコンテナを触ってみた"
emoji: "🍕"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["PHP", "php8", "フレームワーク", "DIコンテナ"]
published: false
---

# はじめに
アプリケーションエンジニアの@yamakenjiです。
元々フロントエンドエンジニアでしたが、最近はPHPやAWS周りを触っており、何エンジニアと名乗れば良いのか悩んでいたら会社から正式にアプリケーションエンジニアと外向けの肩書きが決まったので、これからはそう名乗っていこうと思います。

PHPを触る機会も増えたので、ルーティングやDIコンテナなどを自分なりにフレームワークを試しに実装してみることにしました。
簡易的に実装したものなので、改善の余地はたくさんありますが、今後何か開発をするときに改善していけたらなと思います。特に脆弱性対策や例外処理などは今回は対象外としています。

## 作成物
https://github.com/yamakenji24/pizza
に公開しています。pizza(🍕)という名前はそのときにピザを食べたいなと思ったのでそう名付けています。
- PHP 8.1
- guzzlehttp/psr7@2.6
- psr/container@2
- phpunit/phpunit@10
- phpstan/phpstan@1.1


## 環境構築
環境はDockerを用いて構築します
php-fpmとnginxで構成しており、docker-composeで立ち上げます。

Dockerfileでは、php-fpmのimageをベースに、composerのinstallとdump-autoloadを行なっています。
::::details docker-conf/php-fpm/Dockerfile
```docker: Dockerfile
FROM php:8.1-fpm
RUN apt-get update && apt-get install -y \
    git \
    zip \
    unzip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*


RUN curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer

WORKDIR /var/www/html
COPY ../../composer.json ../../composer.lock ./
RUN composer install
RUN composer dump-autoload

COPY ./ ./

EXPOSE 9000
```
::::

nginxの設定は、/apiに来たときはindex.phpにルーティングするように設定します
::::details docker-conf/nginx/conf.d/pizza.local.conf
```nginx: pizza.local.conf
server {
    listen 80;
    server_name pizza.local.com;
    root /var/www/html/public;
    index index.php index.html index.htm;

    location /api {
        index index.php;
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        include fastcgi_params;
        fastcgi_index index.php;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;

        fastcgi_param SERVER_NAME $http_host;
        fastcgi_param Host $http_host;
        fastcgi_param Accept-Encoding $http_accept_encoding;
        fastcgi_param SERVER_ADDR $server_addr;
        fastcgi_param X-Server-Address $server_addr;

        fastcgi_pass pizza-php:9000;
    }
}
```
::::
最初のルーティングパスであるindex.phpを設定しておきます

```php: public/index.php
<?php declare(strict_types=1);

echo "pizza";
```

最後に、docker-compose.yamlを設定すれば、PHPサーバーが立ち上がります。
:::details docker-compose.yaml
```yaml: docker-compose.yaml
version: '3.7'

services:
  pizza-nginx:
    image: nginx:latest
    container_name: pizza-nginx
    ports:
      - "80:80"
    volumes:
      - ./docker-conf/nginx/nginx.conf:/etc/nginx/nginx.conf
      - ./docker-conf/nginx/conf.d/pizza.local.conf:/etc/nginx/conf.d/pizza.local.conf
      - ./public:/var/www/html/public
      - ./src:/var/www/html/src
    depends_on:
      - pizza-php

  pizza-php:
    build: 
      context: .
      dockerfile: docker-conf/php-fpm/Dockerfile
    container_name: pizza-php
    volumes:
      - ./public:/var/www/html/public
      - ./src:/var/www/html/src

```
:::

# Routingの実装
ルーティングを実装するにあたって、主に以下を参考にしました
https://route.thephpleague.com/

実装したいポイントとしては次のとおりです
- 特定のエンドポイントに対してControllerのマッピングを定義できる
- リクエストに対して、定義したマッピングに基づいて適切に処理が行われる


Routerクラスには、「マッピングの登録」と「どの処理を実行するかを選択する」という2つの責務があります。
マッピングの登録は、`addRoute`メソッドにて行います。どのHTTPメソッドの、どのエンドポイントに対して、どのコントローラーを実行するかという情報を配列として保持しています。
これに対して、実際にきたリクエストを`resolve`メソッドに渡して、どの処理を行うかを取得して実行しています。
```php: src/Routes/Router.php
<?php declare(strict_types=1);

namespace App\Routes;

use Psr\Http\Message\ResponseInterface;
use Psr\Http\Message\ServerRequestInterface;
use GuzzleHttp\Psr7\Response;

class Router
{
    /**
     * @var array<string, array<string, callable|string>>
     */
    private array $routes = [];

    public function addRoute(string $method, string $route, mixed $handler): void {
        $this->routes[$method][$route] = $handler;
    }

    public function resolve(ServerRequestInterface $request): ResponseInterface {
        $method = $request->getMethod();
        $route = $request->getUri()->getPath();

        $handler = $this->routes[$method][$route] ?? null;

        if ($handler) {
            if (is_string($handler) && class_exists($handler)) {
                $class = new $handler();
                return $class($request);
            } else {
                return $handler($request);
            }
        } else {
            return new Response(404, [], 'Not found');
        }
    }
}
```


実際にマッピングの登録を行うのは、`api.php`で定義しています。
例えば、GET /api/sample というルーティングは以下のようになります。
```php: src/Routes/api.php
<?php declare(strict_types=1);

use App\Controller\ApiController;
use App\Routes\Router;
use GuzzleHttp\Psr7\ServerRequest;

return function (): void {
    $request = ServerRequest::fromGlobals();
    
    $router = new Router();

    $router->addRoute('GET', '/api/sample', ApiController::class);

    $response = $router->resolve($request);

    http_response_code($response->getStatusCode());
    echo $response->getBody();
};
```
これらを最初のエントリーポイントである`index.php`で実行しておきます。
```php: public/index.php
<?php declare(strict_types=1);

require __DIR__ . '/../vendor/autoload.php';

call_user_func(require_once __DIR__ . '/../src/Routes/api.php');
```

# Diコンテナの実装

# 終わりに