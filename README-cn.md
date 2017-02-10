JsonRPC 2.0 Client and Server
=============================

[![Build Status](https://travis-ci.org/rryqszq4/php-JsonRPC.svg?branch=master)](https://travis-ci.org/rryqszq4/php-JsonRPC)

轻量级,高性能 JsonRPC 2.0 客户端和服务端的php扩展，基于 multi_curl + epoll的并行客户端。Jsonrpc_Client使用libcurl库的并行接口调取服务，使用IO多路复用的epoll去监听curl的IO事件，使用协程可以同步返回rpc的数据，但底层其实是异步的。Jsonrpc_Server支持php-fpm或swoole。遵守[http://www.jsonrpc.org/](http://www.jsonrpc.org/)协议规范。
[English](https://github.com/rryqszq4/JsonRPC/blob/master/README.md)

[jsonrpc in php7](https://github.com/rryqszq4/php7-ext-jsonrpc)

特性
-----------
* JSON-RPC 2.0协议规范
* 并发curl与epoll结合的并行客户端
* php-fpm中持久化epoll
* php-fpm中持久化curl_multi队列
* 默认使用YAJL解析JSON
* 服务端支持请求与通知
* Linux系统(需要支持epoll)

PHP环境
-----------
- PHP 5.3.*
- PHP 5.4.* 
- PHP 5.5.* 
- PHP 5.6.* 

安装
-----------
```
$/path/to/phpize
$./configure --with-php-config=/path/to/php-config
$make && make install
```

服务端
-----------
**接口**
- Jsonrpc_Server::__construct(mixed $payload, array $callbacks, array $classes)
- Jsonrpc_Server::register(string $name, mixed $closure)
- Jsonrpc_Server::bind(string $procedure, mixed $classname, string $method)
- Jsonrpc_Server::jsonformat()
- Jsonrpc_Server::rpcformat(mixed $payload)
- Jsonrpc_Server::executeprocedure(string $procedure, array $params)
- Jsonrpc_Server::executecallback(object $closure, array $params)
- Jsonrpc_Server::executemethod(string $class, string $method, array $params)
- Jsonrpc_Server::execute(boolean $response_type)

**注册函数**
```php
<?php

$server = new Jsonrpc_Server();

// style one function variable
$add1 = function($a, $b){
	return $a + $b;
};
$server->register('addition1', $add1);

// style two function string
function add2($a, $b){
  return $a + $b;
}
$server->register('addition2', 'add2');

// style three function closure
$server->register('addition3', function ($a, $b) {
    return $a + $b;
});

//style four class method string
class Api 
{
  static public function add($a, $b)
  {
    return $a + $b;
  }
}
$server->register('addition4', 'Api::add');

echo $server->execute();

//output >>>
//{"jsonrpc":"2.0","id":null,"error":{"code":-32700,"message":"Parse error"}}

?>
```

**绑定方法**
```php
<?php

$server = new Jsonrpc_Server();

class Api
{
  static public function add($a, $b)
  {
    return $a + $b;
  }

  public function newadd($a,$b){
    return $a + $b;
  }
}

$server->bind('addition5', 'Api', 'add');

$server->bind('addition6', $a=new Api, 'newadd');

echo $server->execute();

//output >>>
//{"jsonrpc":"2.0","id":null,"error":{"code":-32700,"message":"Parse error"}}

?>

```

**swoole jsonrpc server**
```php
<?php

$http = new swoole_http_server("127.0.0.1", 9501);

function add($a, $b){
  return $a + $b;
}

$http->on('Request', function($request, $response){
  if ($request->server['request_uri'] == "/jsonrpc_server"){
    $payload = $request->rawContent();
    
    $jsr_server = new Jsonrpc_Server($payload);
    $jsr_server->register('addition', 'add');
    $res = $jsr_server->execute();
    $response->end($res);

    unset($payload);
    unset($jsr_server);
    unset($res);
  }else {
          $response->end("error");
  }
});
$http->start();
?>
```

客户端
------------
**接口**
- Jsonrpc_Client::__construct(boolean $persist)
- Jsonrpc_Client::call(string $url, string $procedure, array $params, mixed $id)
- Jsonrpc_Client::connect(string $url)
- Jsonrpc_Client::__call(string $procedure, array $params)
- Jsonrpc_Client::execute(boolean $response_type)
- Jsonrpc_Client::authentication(string $username, string $password)
- Jsonrpc_Client::__destruct()

**持久化**
> Jsonrpc_client(1) 参数为1的时候，将epoll和curl_multi队列两个资源进行持久化，默认使用非持久化。

**直接调用**
```php
<?php

$client = new Jsonrpc_Client(1);
$client->connect('http://localhost/server.php');
$client->addition1(3,5);
$result = $client->execute();

?>
```

**并行调用**
```php
<?php

$client = new Jsonrpc_Client(1);
$client->call('http://localhost/server.php', 'addition1', array(3,5));
$client->call('http://localhost/server.php', 'addition2', array(10,20));

/* ... */
$result = $client->execute();

var_dump($result);

//output >>>
/*
array(2) {
  [0]=>
  array(3) {
    ["jsonrpc"]=>
    string(3) "2.0"
    ["id"]=>
    int(110507766)
    ["result"]=>
    int(8)
  }
  [1]=>
  array(3) {
    ["jsonrpc"]=>
    string(3) "2.0"
    ["id"]=>
    int(1559316299)
    ["result"]=>
    int(30)
  }
  ...
}
*/
?>
```
**自定义 id**
```php
<?php

$client = new Jsonrpc_client(1);
$client->call('http://localhost/server.php', 'addition', array(3,5),"custom_id_001");
$result = $client->execute();
var_dump($result);

//output >>>
/*
array(1) {
  [0]=>
  array(3) {
    ["jsonrpc"]=>
    string(3) "2.0"
    ["id"]=>
    string(13) "custom_id_001"
    ["result"]=>
    int(8)
  }
}
*/
?>
```

YAJL 生成/解析
-------------------
**Interface**
- Jsonrpc_Yajl::generate(array $array)
- Jsonrpc_Yajl::parse(string $json)

**生成**
```php
<?php

$arr = array(
    1,
    "string",
    array("key"=>"value")
);

var_dump(Jsonrpc_Yajl::generate($arr));

/* ==>output
string(28) "[1,"string",{"key":"value"}]";
*/

?>
```

**解析**
```php
<?php

$str = '[1,"string",{"key":"value"}]';

var_dump(Jsonrpc_Yajl::parse($str));

/* ==>output
array(3) {
  [0]=>
  int(1)
  [1]=>
  string(6) "string"
  [2]=>
  array(1) {
    ["key"]=>
    string(5) "value"
  }
}
*/

?>
```

常见错误信息
--------------
**jsonrpc 2.0 错误信息**
```javascript
// 语法解析错误
{"jsonrpc":"2.0","id":null,"error":{"code":-32700,"message":"Parse error"}}

// 无效请求
{"jsonrpc":"2.0","id":null,"error":{"code":-32600,"message":"Invalid Request"}}

// 找不到方法
{"jsonrpc":"2.0","id":null,"error":{"code":-32601,"message":"Method not found"}}

// 无效的参数
{"jsonrpc":"2.0","id":null,"error":{"code":-32602,"message":"Invalid params"}}

//
```

**HTTP协议错误信息**
```javascript
// 400
{"jsonrpc":"2.0","id":null,"error":{"code":-32400,"message":"Bad Request"}}
// 401
{"jsonrpc":"2.0","id":null,"error":{"code":-32401,"message":"Unauthorized"}}
// 403
{"jsonrpc":"2.0","id":null,"error":{"code":-32403,"message":"Forbidden"}}
// 404
{"jsonrpc":"2.0","id":null,"error":{"code":-32404,"message":"Not Found"}}

// 500
{"jsonrpc":"2.0","id":null,"error":{"code":-32500,"message":"Internal Server Error"}}
// 502
{"jsonrpc":"2.0","id":null,"error":{"code":-32502,"message":"Bad Gateway"}}
...

// unknow
{"jsonrpc":"2.0","id":null,"error":{"code":-32599,"message":"HTTP Unknow"}}
```

**curl错误信息**
```javascript
// 1 CURLE_UNSUPPORTED_PROTOCOL
{"jsonrpc":"2.0","id":null,"error":{"code":-32001,"message":"Curl Unsupported Protocol"}}

// 2 CURLE_FAILED_INIT
{"jsonrpc":"2.0","id":null,"error":{"code":-32002,"message":"Curl Failed Init"}}

// 3 CURLE_URL_MALFORMAT
{"jsonrpc":"2.0","id":null,"error":{"code":-32003,"message":"Curl Url Malformat"}}

// 4
{"jsonrpc":"2.0","id":null,"error":{"code":-32004,"message":"Curl Not Built In"}}

// 5 CURLE_COULDNT_RESOLVE_PROXY
{"jsonrpc":"2.0","id":null,"error":{"code":-32005,"message":"Curl Couldnt Resolve Proxy"}}

// 6 CURLE_COULDNT_RESOLVE_HOST
{"jsonrpc":"2.0","id":null,"error":{"code":-32006,"message":"Curl Couldnt Resolve Host"}}

// 7 CURLE_COULDNT_CONNECT
{"jsonrpc":"2.0","id":null,"error":{"code":-32007,"message":"Curl Couldnt Connect"}}
...

// CURL ERROR UNKNOW
{"jsonrpc":"2.0","id":null,"error":{"code":-32099,"message":"Curl Error Unknow"}}
```






