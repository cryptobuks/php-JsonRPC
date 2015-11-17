JsonRPC 2.0 Client and Server
=============================

[![Build Status](https://travis-ci.org/rryqszq4/JsonRPC.svg)](https://travis-ci.org/rryqszq4/JsonRPC)

轻量级 JsonRPC 2.0 客户端和服务端的php扩展，基于 multi_curl + epoll的并行客户端，依据[jsonrpc](http://www.jsonrpc.org/)协议规范。
[en](https://github.com/rryqszq4/JsonRPC/blob/master/README-en.md)

特性
-----------
* JSON-RPC 2.0协议规范
* 并发curl与epoll结合的并行客户端
* php-fpm中持久化epoll
* 服务端支持请求与通知
* Linux系统

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
- Jsonrpc_Server::__construct
- Jsonrpc_Server::register
- Jsonrpc_Server::bind
- Jsonrpc_Server::jsonformat
- Jsonrpc_Server::rpcformat
- Jsonrpc_Server::executeprocedure
- Jsonrpc_Server::executecallback
- Jsonrpc_Server::getresponse
- Jsonrpc_Server::execute

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

****绑定方法****
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


客户端
------------
**接口**
- Jsonrpc_Client::__construct
- Jsonrpc_Client::call
- Jsonrpc_Client::execute

**持久化**
> Jsonrpc_client(1) 参数为1的时候，将epoll资源进行持久化，默认使用非持久化。

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

**自定义客户端错误信息**
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
```






