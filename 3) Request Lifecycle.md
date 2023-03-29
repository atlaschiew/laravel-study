# Request Lifecycle
1) 首先public/index.php就是每个用户请求的入口处。
```sh
<?php
# https://github.com/laravel/framework/tree/8.x/src/Illuminate/Contracts
# 包里包含了全部laravel framework的interface
# interface的用意就是可以让开发人员可以自己客制化和扩展。
use Illuminate\Contracts\Http\Kernel;

# https://github.com/laravel/framework/blob/8.x/src/Illuminate/Http/Request.php
# http request包
use Illuminate\Http\Request;

# 没发现任何用处
define('LARAVEL_START', microtime(true));

# php artisan down将会生成storage/framework/maintenance.php
# php artisan up则会删除maintenance.php 
if (file_exists($maintenance = __DIR__.'/../storage/framework/maintenance.php')) {
    require $maintenance;
}

# composer的auto loader
require __DIR__.'/../vendor/autoload.php';

# 获得一个app对象。
  # 同时3个底层ServiceProvider也被注册,分别为EventServiceProvider, LogServiceProvider和RoutingServiceProvider
$app = require_once __DIR__.'/../bootstrap/app.php';

# 使用make方法，获取Illuminate\Contracts\Http\Kernel的实例 (instance)。
# You may use the make method to resolve a class instance from the container. 
# The make method accepts the name of the class or interface you wish to resolve.
$kernel = $app->make(Kernel::class);

# 前面发生的都是铺垫，接下来kernel->handle就是动真格了，下篇说。
$response = $kernel->handle(
    $request = Request::capture()
)->send();

$kernel->terminate($request, $response);
```

2) 话说一旦执行`$kernel->handle`时，以下的事情将发生。
```sh
# 最底层(foundation)的这些bootstrappers (https://github.com/laravel/framework/blob/8.x/src/Illuminate/Foundation/Http/Kernel.php#L36) 将被boot()

# 最底层bootstrapper的执行顺序是
$response = $kernel->handle($request = Illuminate\Http\Request::capture());
$response = $this->sendRequestThroughRouter($request);
$this->bootstrap();
$this->app->bootstrapWith($this->bootstrappers());
$this->make($bootstrapper)->bootstrap($this);

# LoadConfiguration (第二个) 加载全部在project/config/*.php里的设置。
# RegisterFacades (第四个) 注册facade, 简单理解facade就是static化的helper，方便调用
# RegisterProviders (最后第二个)注册cached services (@bootstrap/cache/services.php)
# BootProviders(最后一个)回报app关于底层bootstrapper已完成,是时候boot services。
$app->boot() 

# boot每个已注册的services
array_walk($this->serviceProviders, function ($p) {
    $this->bootProvider($p);
});

# 目前我们所在的位置是$kernel->handle,再看看接下来又发生了什么事。
# 在sendRequestThroughRouter($request);有个诡异的东西叫pipeline, 先发个链接
# https://medium.com/@jeffochoa/understanding-laravel-pipelines-a7191f75c351，日后再了解！

# sendRequestThroughRouter主要的作用

  # 启动每个middleware来过滤request, 主要逻辑在pipeline里的carry (https://github.com/laravel/framework/blob/1b52a876131611b1297446c24964d8727fe5aef8/src/Illuminate/Pipeline/Pipeline.php#L97)
  # 其中$passable表示$request, $stack表示存中间件的栈, $pipe表示当前的中间件。

  # router的执行顺序 (https://github.com/laravel/framework/blob/8.x/src/Illuminate/Routing/Router.php)
  $this->router->dispatch($request)# caller将请求分发到路由
  $this->dispatchToRoute($request) # 将请求分发到路由，并返回响应
  $this->findRoute($request)       # 从 RouteCollection 路由集合中查找出当前请求 URI（$request）匹配的路由
  # 在运行路由闭包或控制器方法时，将采用类似 HTTP kernel 的 handle 执行方式去运行当前路由适用的局部中间件
  # 在最终的 then 方法内部会执行 $route->run() 方法运行路由，$route（Illuminate\Routing\Route）
  $this->runRoute($request, $routeInstance/*findRoute返回结果*/) 
  # 生成 HTTP 响应（由 prepareResponse 方法完成）。

  # routeInstance的执行顺序 (https://github.com/laravel/framework/blob/8.x/src/Illuminate/Routing/Route.php)
  # 运行路由闭包或控制器并对接(bridge)去你写的controller，最后返回响应(response)结果。
```

理解源码再来看以下的图就会更加明白。
<img src="https://user-images.githubusercontent.com/45816141/226101690-60a46d6b-687f-49f6-87ea-8793e2b3cca4.jpg"/>
