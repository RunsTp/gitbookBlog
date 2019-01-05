# Swoole\Coroutine\Channel 实现并发控制
---

### 序
---
某日遇到朋友需要爬取大量图片资源，但是PHP的 `Curl`，`fopen` 等都是阻塞的，网上提供的很多方案都会将图片资源读进 `buffer`，即便使用 `yield` 来实现非阻塞，一但并发数提高可能就会导致爆内存...  ~~其实都不是主要问题，主要是带宽卡着。~~  

于是就那一刻就想当然的觉得可以使用 `[Swoole\Coroutine\Http\Client](https://wiki.swoole.com/wiki/page/p-coroutine_http_client.html)` 来解决这个问题，和 `Curl` 不同， 协程Client 提供了非阻塞的请求和直接写入磁盘下载文件的特性。

### 依赖
---
- [Swoole\Coroutine\Http\Client](https://wiki.swoole.com/wiki/page/p-coroutine_http_client.html) 不多说，本期主角
- [Saber](https://github.com/swlib/saber) 高性能协程Http客户端 (作者是Swoole开发组成员，我很喜欢这个库，也很崇拜作者.jpg)
- [Composer](https://www.phpcomposer.com/) 使用Composer安装 `Saber` 和自动加载

### 上代码
---

**简单封装**
```php
<?php
require_once "vendor/autoload.php";

use Swlib\SaberGM;
use Swlib\Http\ContentType;
use Swoole\Coroutine\Channel;

class Crawler
{
    private $page = 1;
    private $totalPage = 1;
    private $fileType = 'png';

    public function getTaskList()
    {
        if ($this->totalPage < $this->page) {
            return false;
        }

        // 获取图片列表
        $response = $this->postRequest($this->getListUrl($this->page));
        $this->page++;
        if (!isset($response['status']) || $response['status'] !== 'success') {
            return $this->getTaskList();
        }

        // 获取图片总页数
        $this->totalPage = $response['total_page'];

        $imageUrlArray = Array();
        foreach ($response['data_list'] as $tagItem) {
            $imgIdsArr = explode(',', $tagItem['fids']);
            foreach ($imgIdsArr as $imgId) {
                $url = $this->createDownloadUri($tagItem['_id'] . '/' . $imgId);
                $filename = $this->createFileName($imgId);
                $path = $savePath = $this->createSavePath($tagItem['author_id'] . '/' . $tagItem['_id']. '/'. $filename);
                array_push($imageUrlArray, ['url' => $url, 'path' => $path]);
            }
        }

        return $imageUrlArray;
    }

    protected function getListUrl(int $page)
    {
        return "http://data.boluonvshen.com/UMETA/API/Logic/Connection/connection.php?json_data={\"sort_type\":\"hottest|newest\",\"page_size\":\"10\",\"obj\":\"photoGallery\",\"act\":\"get_mix_gallery_list\",\"page\":\"".$page."\"}";
    }

    protected function createSavePath(string $relativePaths)
    {
        return '/home/runs/Downloads/images/'. $relativePaths;
    }

    protected function createDownloadUri(string $fileName)
    {
        return 'http://youmitao.image.boluonvshen.com/'. $fileName;
    }

    protected function createFileName($fileId)
    {
        return $fileId. $this->fileType;
    }

    public function saveRemoteFile($saveArray)
    {
        $path = dirname($saveArray['path']);
        if (!is_dir($path)) {
            if (mkdir($path, 0755, true) === false) {
                throw new Exception('mkdir '. $path. 'failure!');
            }
        }
        SaberGM::download($saveArray['url'], $saveArray['path']);
    }

    protected function postRequest($url, $params = null, $options = [])
    {
        $opt = [
            'headers' => [
                'Content-Type' => ContentType::JSON,
                'Accept' => 'application/json',
                'Cache-Control' => 'no-cache',
                'Pragma' => 'no-cache'
            ]
        ] + $options;
        $response = SaberGM::post($url, $params, $opt);
        return $response->getParsedJsonArray();
    }
}
```

**执行**
```php
go(function (){
    $i = 0;
    $crawler = new Crawler();
    while (true) {
        $list = $crawler->getTaskList();
        // 完成
        if ($list === false) {
            echo "ok \n";
            break;
        }

        foreach ($list as $task) {
            go(function () use ($crawler, $task){
                $crawler->saveRemoteFile($task);
            });
            $i++;
            echo "download start $i \n";
        }
    }
});
```

此时我已经安奈不住心情，感觉自己已经要起飞！然而....
> Too many open files

经过分析下载文件时候会打开一个文件句柄，而非阻塞则会在网络IO时自动挂起然后继续执行后续代码(就会导致开非常非常多的文件句柄)然后就爆炸了。~~如果你网络非常非常好，可能并不会遭遇这种场景。~~

于是想到了并发控制..保证并发下载量在可控范围内。

### 并发控制
---
归根结底，并发控制就是确保单次任务投递数量可控。但是如何在协程环境中控制，还是第一次遇到。
随后在文档中找到了 [Swoole\Coroutine\Http\Client](https://wiki.swoole.com/wiki/page/p-coroutine_http_client.html)，一个底层封装的生产者消费者并且实现了自动协程切换调度。

**执行**
```php
$max = 50; // 并发控制
$chan = new Channel($max);
$crawler = new Crawler();

go(function () use ($chan, $crawler){
    $i = 1;
    while (true) {
        $list = $crawler->getTaskList();
        // 完成
        if ($list === false){
            break;
        }
        foreach ($list as $item){
            $chan->push($item);
            echo "push $i \n";
            $i++;
        }
    }
});
go(function () use ($chan, $crawler){
    $i = 1;
    while(true) {
        $data = $chan->pop(3);
        if ($data === false){
            break;
        }
        $crawler->saveRemoteFile($data);
        echo "download start $i \n";
        $i++;
    }
    $chan->close();
    echo "end \n";
});
```

### 简单讲解
---
`Channel` 可以理解成一个队列，当任何一个生产者投递到这个实例中的时候，他就会压入队列。
而队列的容量则是 `new Channel($容量)` 时设置的， 由于 `Channel` 只使用php的引用计数来保存变量，所以 `$容量 = 10` 的时候则代表只容纳10个变量。 其真实占用大小则是 `$容量 * zval大小`(php7中 zval占用16字节)

当队列满时，所有生产者(push操作)会被挂起，让出给消费者(pop操作)，以此保证并发数。
