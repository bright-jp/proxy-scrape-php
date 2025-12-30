[![Promo](https://brightdata.jp/static/github_promo_15.png?md5=105367-daeb786e)](https://brightdata.jp/?promo=github15) 
# PHP Proxy Server: PHPでプロキシを設定する方法

cURL、`file_get_contents()`、Symfony を使用して PHP でプロキシを設定する方法を学びます。また、WebスクレイピングとIPローテーションのために、PHPでBright Dataの[residential proxies](https://brightdata.jp/proxy-types/residential-proxies)を使用する方法もご紹介します。このガイドは[Bright Data blog](https://brightdata.jp/blog/how-tos/php-proxy-servers)でもご覧いただけます。

- [要件](#requirenments)
  - [Apacheでローカルプロキシサーバーをセットアップする](#setting-up-a-local-proxy-server-in-apache)
- [PHPでプロキシを使用する](#using-proxies-in-php)
  - [cURLでのプロキシ統合](#proxy-integration-with-curl)
  - [`file_get_contents()`を使用したプロキシ統合](#proxy-integration-using-file_get_contents)
  - [Sympfonyでのプロキシ統合](#proxy-integration-in-symfony)
- [PHPでのプロキシ統合をテストする](#testing-proxy-integration-in-php)
- [PHPでのBright Dataプロキシ統合](#bright-data-proxy-integration-in-php)
  - [レジデンシャルプロキシのセットアップ](#residential-proxy-setup)
  - [認証付きプロキシ経由のWebスクレイピング例](#web-scraping-example-through-an-authenticated-proxy)
  - [IPローテーションのテスト](#testing-ip-rotation)

## Requirenments

お使いのマシンに[PHP 8+](https://www.php.net/downloads.php)、[Composer](https://getcomposer.org/download/)、[Apache](https://httpd.apache.org/download.cgi)がインストールされていることを確認してください。インストールされていない場合は、上記リンクをクリックしてインストーラーをダウンロードし、起動して手順に従ってください。

Apacheサービスが起動して稼働していることを確認してください。

PHPプロジェクト用のフォルダーを作成し、そこに移動して、内部で新しいComposerアプリケーションを初期化します。

```bash
mkdir <PHP_PROJECT_FOLDER_NAME>
cd <PHP_PROJECT_FOLDER_NAME>
composer init
```

**Note**: Windowsでは、WSL（[Windows Subsystem for Linux](https://learn.microsoft.com/en-us/windows/wsl/install)）の使用を推奨します。

### Setting Up a Local Proxy Server in Apache

ApacheのローカルWebサーバーがフォワードプロキシサーバーとして動作するように設定します。

まず、次のコマンドで[`mod_proxy`](https://httpd.apache.org/docs/2.4/mod/mod_proxy.html)、[`mod_proxy_http`](https://httpd.apache.org/docs/2.4/mod/mod_proxy_http.html)、[`mod_proxy_connect`](https://httpd.apache.org/docs/2.4/mod/mod_proxy_connect.html)モジュールを有効化します。

```bash
sudo a2enmod proxy
sudo a2enmod proxy_http
sudo a2enmod proxy_connect
```

次に、`/etc/apache2/sites-available/`内に、デフォルトのバーチャルホスト設定ファイル`000-default.conf`をコピーして、`proxy.conf`という新しい[virtual host configuration file](https://httpd.apache.org/docs/2.4/vhosts/)を作成します。

```bash
cd /etc/apache2/sites-available/
sudo cp 000-default.conf proxy.conf
```

以下のプロキシ定義ロジックで`proxy.conf`を初期化します。

```
<VirtualHost *:80>
    # set the server name to localhost
    ServerName localhost
    # set the server admini email to admin@localhost
    ServerAdmin admin@localhost

    # if the SSL module is enabled
    <IfModule mod_ssl.c>
        # disable SSL to avoid certificate errors
        SSLEngine off
    </IfModule>

    # specify the error log file location
    ErrorLog ${APACHE_LOG_DIR}/error.log
    # specify the access log file location and format
    CustomLog ${APACHE_LOG_DIR}/access.log combined

    # enable proxy logic
    ProxyRequests On
    ProxyVia On

    # define a proxy for all requests
    <Proxy *>
        Order deny,allow
        Allow from all
    </Proxy>
</VirtualHost>
```

次のコマンドで、新しいApacheバーチャルホストを登録します。

```bash
sudo a2ensite proxy.conf
```

最後に、Apacheサーバーをリロードします。

```bash
service apache2 reload
```

これで、`http://localhost:80`で待ち受けるローカルプロキシサーバーが用意できました。

## Using Proxies in PHP

以下の技術に、PHPでプロキシを統合する方法を見ていきます。

- [cURL](https://www.php.net/manual/en/book.curl.php)
- [`file_get_contents()`](https://www.php.net/manual/en/function.file-get-contents.php)
- [Symfony](https://symfony.com/)

### Proxy Integration With cURL

cURLライブラリを使用してPHPでプロキシサーバーを指定するには、[`CURLOPT_PROXY`](https://curl.se/libcurl/c/CURLOPT_PROXY.html)オプションを使用します。以下の`curl_proxy.php`のスニペットのとおりです。

```php
// the URL of the proxy server
$proxyUrl = 'http://localhost:80';
// the URL of the target site
$targetUrl = 'https://httpbin.org/get';

// initialize a cURL session
$ch = curl_init();

// set the target URL
curl_setopt($ch, CURLOPT_URL, $targetUrl);
// set the proxy server to be used for routing the request
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
// return the response as a string
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);

// disable SSL certificate verification to avoid certificate errors
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);
curl_setopt($ch, CURLOPT_SSL_VERIFYHOST, false);

// execute the cURL request
$response = curl_exec($ch);

// if the response is not successful
if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    echo $response;
}

// close the cURL session
curl_close($ch);
```

### Proxy Integration Using `file_get_contents()`

`file_get_contents()`でプロキシサーバーを設定するには、`proxy`オプションを使用します。以下の`file_get_contents_proxy.php`のスニペットのとおりです。

```php
// define the proxy server to be used for HTTP/HTTPS requests
$options = [
    'http' => [
        'proxy' => 'tcp://localhost:80',
        // force the use of the full URI when making the request
        'request_fulluri' => true,
    ],
];
// create a stream context with the defined options
$context = stream_context_create($options);

// the URL of the target site
$url = 'https://httpbin.org/get';
// perform an HTTP request with the defined context
$response = file_get_contents($url, false, $context);

// if the response was not successful
if ($response === false) {
    echo "Failed to retrieve data from $url";
} else {
    echo $response;
}
```

**Note**: `proxy`オプションに指定するプロキシサーバーのプロトコルは、`http`ではなく`tcp`である必要があります。

### Proxy Integration in Symfony

Symfonyコンポーネントの[`BrowserKit`](https://symfony.com/doc/current/components/browser_kit.html)と[`HTTP Client`](https://symfony.com/doc/current/http_client.html)をインストールします。

```bash
composer require symfony/browser-kit symfony/http-client
```

`HttpBrowser`を使用してHTTPリクエストを行う際に、`HttpClient`の[`proxy`](https://symfony.com/doc/current/http_client.html#http-proxies)オプションでプロキシサーバーを指定します。以下の`symfony_proxy.php`のスニペットのとおりです。

```php
// include the Composer autoload file
require './vendor/autoload.php';

// load the required load Symfony components
use Symfony\Component\BrowserKit\HttpBrowser;
use Symfony\Component\HttpClient\HttpClient;

// define the proxy server and port
$proxyServer = 'http://localhost';
$proxyPort = '80';

// create an HTTP client with a proxy configuration
$client = new HttpBrowser(HttpClient::create(['proxy' => sprintf('%s:%s', $proxyServer, $proxyPort)]));

// make a GET request to the target URL
$client->request('GET', 'https://httpbin.org/get');

// get the content of the response
$content = $client->getResponse()->getContent();

// output the content
echo $content;
```

## Testing Proxy Integration in PHP

以下のコマンドを使用して、3つのPHPプロキシ統合スクリプトのいずれかを起動します。

```bash
php <PHP_SCRIPT_NAME>
```

どのスクリプトを実行しても、結果は次のようになります。

```json
{
  "args": {},
  "headers": {
    "Accept": "*/*",
    "Host": "httpbin.org",
    "X-Amzn-Trace-Id": "Root=1-661ab837-40de4746307643415ec9c659"
  },
  "origin": "XX.YY.ZZ.AA",
  "url": "https://httpbin.org/get"
}
```

プロキシによって行われたリクエストを追跡するApacheログファイル`access.log`を確認してみてください。

```
tail -n 50 /var/log/apache2/access.log
```

最終行は、リクエストが`httpbin.org`へ正常にプロキシされ、レスポンスのステータスコードが`200`であったことを示します。

```
::1 - - [13/Apr/2024:18:53:22 +0200] "CONNECT httpbin.org:443 HTTP/1.0" 200 6138 "-" "-"
```

## Bright Data Proxy Integration in PHP

Bright Dataは、出口IPを自動でローテーションする[premium proxies](https://brightdata.jp/proxy-types)を提供しています。cURLを使用したPHPスクリプトで、これらをWebスクレイピングに利用する方法を見ていきましょう。

### Residential Proxy Setup

無料トライアルを開始するために、[Bright Dataにサインアップ](https://brightdata.jp/cp/start)してください。「Proxies & Scraping Infrastructure」ダッシュボードに移動し、「Residential Proxy」カードの「Get Started」をクリックします。

手順に従ってレジデンシャルプロキシをセットアップし、次の認証情報を取得してください。

- `<BRIGHTDATA_PROXY_HOST>`
- `<BRIGHTDATA_PROXY_PORT>`
- `<BRIGHTDATA_PROXY_USERNAME>`
- `<BRIGHTDATA_PROXY_PASSWORD>`

### Web Scraping Example Through an Authenticated Proxy

Bright Dataの認証付きレジデンシャルプロキシを使用して、["Proxy server" Wikipedia page](https://en.wikipedia.org/wiki/Proxy_server)に接続し、[`DOMDocument`](https://www.php.net/manual/en/class.domdocument.php)でデータをスクレイピングします。以下の`curl_proxy_scraping.php`スニペットのとおりです。

```php
// Bright Data proxy details
$proxyUrl = '<BRIGHTDATA_PROXY_HOST>:<BRIGHTDATA_PROXY_PORT>';
$proxyUser = '<BRIGHTDATA_PROXY_USERNAME>:<BRIGHTDATA_PROXY_PASSWORD>';

// target scraping page
$targetUrl = 'https://en.wikipedia.org/wiki/Proxy_server';

// perform a GET request to the target page
// through the Bright Data proxy
$ch = curl_init();

curl_setopt($ch, CURLOPT_URL, $targetUrl);
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
curl_setopt($ch, CURLOPT_PROXYUSERPWD, $proxyUser);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

$response = curl_exec($ch);

if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    // parse the HTML document returned by the server
    $dom = new DOMDocument();
    @$dom->loadHTML($response);

    // extract the text content on the page
    $content = $dom->getElementById('mw-content-text')->textContent;

    // extract the titles from the H2 headings
    $headings = [];
    $headingsNodeList = $dom->getElementsByTagName('h2');
    foreach ($headingsNodeList as $heading) {
        $headings[] = $heading->textContent;
    }

    // extract the titles from the H3 headings
    $headingsNodeList = $dom->getElementsByTagName('h3');
    foreach ($headingsNodeList as $heading) {
        $headings[] = $heading->textContent;
    }

    // print the scraped data
    echo "Content:\n";
    echo $content . "\n\n";

    echo "Headings:\n";
    foreach ($headings as $index => $heading) {
        echo ($index + 1) . ". $heading\n";
    }
}

curl_close($ch);
```

出力は次のようになります。

```
Content:
Computer server that makes and receives requests on behalf of a user
.mw-parser-output .hatnote{font-style:italic}.mw-parser-output div.hatnote{padding-left:1.6em;margin-bottom:0.5em}.mw-parser-output .hatnote i{font-style:normal}.mw-parser-output .hatnote+link+.hatnote{margin-top:-0.5em}For Wikipedia's policy on editing from open proxies, please see Wikipedia:Open proxies. For other uses, see Proxy.


Communication between two computers connected through a third computer acting as a proxy server.
// omitted for brevity...

Headings:
1. Contents
2. Types[edit]
3. Uses[edit]
// omitted for brevity...
```

### Testing IP Rotation

IPに関する情報を返す特別なエンドポイントである[`http://lumtest.com/myip.json`](http://lumtest.com/myip.json)をターゲットにしたPHPプロキシスクリプト`curl_proxy_brightdata.php`を実行します。

```php
<?php

$proxyUrl = '<BRIGHTDATA_PROXY_HOST>:<BRIGHTDATA_PROXY_PORT>';
$proxyUser = '<BRIGHTDATA_PROXY_USERNAME>:<BRIGHTDATA_PROXY_PASSWORD>';

$targetUrl = 'http://lumtest.com/myip.json';

$ch = curl_init();

curl_setopt($ch, CURLOPT_URL, $targetUrl);
curl_setopt($ch, CURLOPT_PROXY, $proxyUrl);
curl_setopt($ch, CURLOPT_PROXYUSERPWD, $proxyUser);
curl_setopt($ch, CURLOPT_RETURNTRANSFER, true);
curl_setopt($ch, CURLOPT_SSL_VERIFYPEER, false);

$response = curl_exec($ch);

if (curl_errno($ch)) {
    echo 'cURL Error: ' . curl_error($ch);
} else {
    echo $response;
}

curl_close($ch);
```

スクリプトを複数回実行してください。毎回、異なる場所からの異なるIPが表示されます。