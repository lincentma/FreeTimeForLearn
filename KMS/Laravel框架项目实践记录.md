# Laravel框架项目实践记录
> 在Laravel框架实践过程中对于composer、mysql、redis队列、OpenID和JWT的实践总结

>  从入职到现在，自己一直在学习和掌握Laravel框架的方方面面的知识，自己将项目开发中遇到的问题整理，希望可以对Laravel框架的基础知识和相关使用经验进行总结，能够所有帮助。

## 创建项目——composer
> composer是一种PHP包管理工具，Laravel采用composer来管理项目的代码依赖。

### composer 安装镜像

  在梳理项目过程中，通常采用以下几个镜像地址获取第三方依赖的具体信息。通常镜像地址为国外镜像和国内镜像：
  - composer : https://packagist.org
  - phpcomposer ：https://packagist.phpcomposer.com
  - laravel-china : https://packagist.laravel-china.org
 
  由于从国内直接访问国外官方镜像地址下载速度缓慢，由热心个人以及社区维护的国内镜像成为项目更好的选择。其中最开始采用phpcomposer国内镜像，但是由于其不再维护，访问失败。所以laravel-china是目前访问速度以及可靠性最优的选择。

  composer全局配置生效命令为

  ```
  $ composer config -g repo.packagist composer https://packagist.laravel-china.org
  ```

  **需要注意的是，composer尽量不要选择root用户安装，即使root用户操作很“方便”。因为composer管理的项目的码依赖中，第三方代码可以通过composer中的install、exec等命令访问当前运行composer的用户。所以基于安全问题，不要使用root用户进行composer的操作**

  同时还有一个值得注意的地方是，在composer.json文件中是否存在**repositories (root-only)**，例如

  ```
  "repositories": {
      "packagist": {
          "type": "composer",
          "url": "https://packagist.org"
      }
  }
  ```
   如果项目配置的repositories (root-only)的镜像地址与全局配置的composer镜像地址不一致，则按照项目中repositories (root-only)的设置地址进行依赖下载。

### composer 更换镜像地址

  原有项目中的composer.lock文件中的下载地址是phpcomposer镜像地址,需要更换为laravel-china镜像下载地址。

  在刷新composer.lock文件时，之前自己采用的是直接将composer.lock文件删除，然后通过配置新的composer镜像地址进行重新安装。这种做法不好的地方有以下两点：

  1. composer.json文件中对于第三方依赖的版本号规则不尽相同，当没有确定的版本号时，采用版本号范围（逗号、或运算符）以及版本号通配符（*）都会导致安装的具体版本与之前的composer.lock文件中的版本号不一致，会影响正式环境的稳定性。
  2. composer install会重新安装所有的第三方依赖，会导致内存占用过大，composer进行被killed。

  采用composer update nothing可以刷新替换composer.lock文件内的镜像下载地址而无需删除原有的composer.lock文件。
  ```
  $ composer update nothing
  ```

### composer install过程中killed问题

自己开发环境在VMware虚拟机Debian中，通过添加swap交换空间大小以及增加PHP运行时的内存限制来解决killed问题。

- 添加swap交换空间
```
# dd if=/dev/zero of=/var/swap bs=1M count=1024
# mkswap /var/swap
# swapon /var/swap
```
- 设置PHP进程运行内存限制
```
$ php -d memory_limit=-1 /path/to/composer install composer install -vvv --profile
```
  
> 总结Tips

1. 经过多次composer install 被killed的经验后，composer install的内存占用大于composer update，每一次添加项目依赖时，通过composer require + 具体版本号码是更好的选择。
2. 直接将其他机器中项目的vendor目录拷贝过来进行重新安装也是一种简单粗暴的方法。

## 业务数据——MySQL查询

> 业务逻辑处理离不开SQL查询，从一个分组查询最新记录来看SQL语句查询差异。

### 数据查询需求

按照页面、空间、用户等维度统计最新访问时间。不同的数据查询逻辑导致不同的接口响应时间。

### 数据库表结构

精简后的相关表结构如下所示：

字段 | 类型 | 注释
---|---|---
id | bigint(20) | 主键ID
page_id | int(10) | 页面ID
page_type | tinyint(4) | 页面类型
view_time | int(10) | 浏览时间
space_key | varchar(255) | 空间KEY

- 索引设置：
  - index_page_id
  - index_user_name
  - index_space_key
  - index_page_type

- MySQL版本：5.5.52-0+deb8u1-log

- **整体思路：这是一个分组取最新的查询问题。根据不同的字段进行分组，查询各自分组的最新一条数据。主要是对比不同查询方式的查询时间的优劣。保证接口响应时间。**

### 方式1——GROUP_CONTACT + SUBSTRING
  - 通过将每一个分组的浏览时间通过GROUP_CONTACT函数进行组内排序，再通过SUBSTRING函数取第一个即为分组内的最新的浏览数据。
  - 优点：简单直接。
  - 缺点：GROUP_CONTACT默认长度为1024, 数据量增大容易超过限制；全表扫描，数据量增大导致查询时间增大。
 

```
SELECT
    *
FROM
    `confluence_page_view`
WHERE
    `id` IN (
SELECT
    SUBSTRING_INDEX( GROUP_CONCAT( id ORDER BY `view_time` DESC ), ',', 1 )
FROM
    `confluence_page_view`
WHERE
    `space_key` IN (
    'LeiHuo',
    ...
    )
    AND `view_time` BETWEEN '1514736000'
    AND '1544543999'
GROUP BY
    `space_key`
    )
ORDER BY
    `view_time` DESC
 
+----+--------------------+----------------------+-------+-----------------+-----------------+---------+------+------+-----------------------------+
| id | select_type        | table                | type  | possible_keys   | key             | key_len | ref  | rows | Extra                       |
+----+--------------------+----------------------+-------+-----------------+-----------------+---------+------+------+-----------------------------+
|  1 | PRIMARY            | confluence_page_view | ALL   | NULL            | NULL            | NULL    | NULL | 7724 | Using where; Using filesort |
|  2 | DEPENDENT SUBQUERY | confluence_page_view | range | index_space_key | index_space_key | 768     | NULL | 7561 | Using where                 |
+----+--------------------+----------------------+-------+-----------------+-----------------+---------+------+------+-----------------------------+
```

### 方式2——LEFT JOIN
  - 通过将自身与自身进行左外连接，根据分组维度进行WHERE条件查询和浏览时间比较，右表中没有匹配的行即为分组内最大记录。
  - 优点：通过左外连接，减少了总查询行数。
  - 缺点：逻辑稍显复杂，同样会全表扫描，数据量增大导致查询时间增大。

```
SELECT
    `confluence_v1`.`space_key`,
    `confluence_v1`.`view_time`
FROM
    `confluence_page_view` AS `confluence_v1`
    LEFT JOIN `confluence_page_view` AS `confluence_v2` ON `confluence_v2`.`space_key` = `confluence_v1`.`space_key`
    AND `confluence_v1`.`id` < `confluence_v2`.`id`
WHERE
    `confluence_v2`.`id` IS NULL
    AND `confluence_v1`.`space_key` IN (
    'tfweb',
    ...
    )
    AND `confluence_v1`.`view_time` BETWEEN '1514736000'
    AND '1545062399'
ORDER BY
    `confluence_v1`.`view_time` DESC;
 
 
+----+-------------+-------+------+-------------------------+-----------------+---------+---------------------------+------+--------------------------------------+
| id | select_type | table | type | possible_keys           | key             | key_len | ref                       | rows | Extra                                |
+----+-------------+-------+------+-------------------------+-----------------+---------+---------------------------+------+--------------------------------------+
|  1 | SIMPLE      | c1    | ALL  | index_space_key         | NULL            | NULL    | NULL                      | 8248 | Using where; Using filesort          |
|  1 | SIMPLE      | c2    | ref  | PRIMARY,index_space_key | index_space_key | 768     | confluence_p.c1.space_key |   55 | Using where; Using index; Not exists |
```

### 方式3——SUB QUERY + MAX
  - 通过子查询以及聚合函数MAX获取分组内最新时间记录，再与自身内连接根据最新时间进行条件查询。
  - 优点：通过子查询避免了全表扫描。
  - 缺点：暂无，目前查询时间最少的方法。

```
SELECT
    `space_key`,
    `view_time`
FROM
    `confluence_page_view`
    INNER JOIN (
SELECT
    MAX( id ) AS id
FROM
    `confluence_page_view`
WHERE
    `view_time` BETWEEN '1514736000'
    AND '1830355199'
    AND `space_key` IN (
    'tfweb',
    ...
    )
GROUP BY
    `space_key`
    ) AS confluence_v1 ON `confluence_v1`.`id` = `confluence_page_view`.`id`
ORDER BY
    `view_time` DESC
 
 
+----+-------------+----------------------+--------+-----------------+-----------------+---------+------------------+-------+---------------------------------+
| id | select_type | table                | type   | possible_keys   | key             | key_len | ref              | rows  | Extra                           |
+----+-------------+----------------------+--------+-----------------+-----------------+---------+------------------+-------+---------------------------------+
|  1 | PRIMARY     | <derived2>           | ALL    | NULL            | NULL            | NULL    | NULL             |    33 | Using temporary; Using filesort |
|  1 | PRIMARY     | confluence_page_view | eq_ref | PRIMARY         | PRIMARY         | 8       | confluence_v1.id |     1 |                                 |
|  2 | DERIVED     | confluence_page_view | index  | index_space_key | index_space_key | 768     | NULL             | 28102 | Using where                     |
+----+-------------+----------------------+--------+-----------------+-----------------+---------+------------------+-------+---------------------------------+
```

> 总结Tips

1. GROUP BY执行顺序：Where > Group By > Order By。

2. 通过不断实践验证不同的方案在具体应用场景下的查询效果，通过EXPALIN命令分析数据查询语句执行过程，提高查询执行效率，避免慢SQL的产生。

## 业务优化——队列

> 队列是异步解耦长耗时的操作的有效方式，提高响应时间和保证失败重试的常用方法，以数据导出Excel发送邮件功能为例。将多表多字段的数据导出功能，通过异步队列的方式通过邮件发送等方式减少用户的页面响应的等待时间。

> Laravel 队列为不同的后台队列服务提供统一的API，比如Beanstalk，Amazon SQS，Redis。结合项目实际已有的predis，采用基于Redis的Laravel队列

### 基于Redis的Laravel队列

1. 配置Laravel Redis队列
  在**config/queue.php**配置文件中配置redis的队列connection信息。
```
'redis' => [
    'driver' => 'redis',
    'connection' => 'kvstore',
    'queue' => 'data_export',
    'retry_after' => 90,
],
```
2. 创建任务类

  默认在**app/Jobs**目录下创建新的任务类。
```
$ php artisan make:job SendExportDataMailJob
```

3. 推送任务至任务队列

  在业务逻辑逻辑代码中将耗时较长的数据导出Excel的同步代码功能，推送到对应的任务队列。伪代码如下。
```
[...]
SendExportDataMailJob::dispatch($this, $this->mail_service, $this->user_service);
[...]
```
4. 开启任务队列的监听
可以配置队列的指定队列(--queue)，超时时间(--timeout)、睡眠时间(--sleep)、失败任务重试的次数(--tries)等参数,命令如下。
```
$ php artisan queue:work redis --queue=data_export --timeout=60 --sleep=5 --tries=3
```
### 保证队列读取最新代码

Laravel自带队列在生产环境中使用存在的问题有以下2点：
1. 队列监听命令queue:work与queue:listen均会在当前终端窗口运行中，由于使用Ctrl+C命令将进程关闭。
2. queue:work在队列启动后，不会再读取代码的修改；queue:listen仍然会不断读取加载最新的代码，但其效率不如queue:work，同时官方文档中也没有再推荐使用queue:listen。

对应的解决方法是：
1. 对于Larvel队列后台运行的问题，推荐采用supervisor实现队列监听的后台运行。但由于在测试以及正式环境中supervisor需要root权限进行添加对应的后台任务以及对于队列代码修改后，需要重启Supervisor的队列管理才能生效。对于代码调试和频发发布不是很友好。
**采用crontab定时任务**的方式手动保证代码修改过后及时重启队列。


- 主要思路是通过记录执行队列监听命令的进程PID，通过文件保存，每一次定时任务启动时通过查询对应的PID来判断队列监听进程是否存在，如果不存在则手动启动并保存对应的进程PID。
```
*/1 * * * * cd /srv/sites/phpapi/consumemgr.leihuo.netease.com && php artisan data_export_queue_checkup
```
```
/**
 * Check if the queue listener is running.
 *
 * @return bool
 */
private function isQueueListenerRunning() {
    if (!$pid = $this->getLastQueueListenerPID()) {
        return false;
    }
    $process                = exec("ps -p $pid -opid=,cmd=");
    $processIsQueueListener = !empty($process); 
    return $processIsQueueListener;
}

/**
 * Get any existing queue listener PID.
 *
 * @return bool|string
 */
private function getLastQueueListenerPID() {
    if (!file_exists(LOG_DIR . '/queue.pid')) {
        return false;
    }
    return file_get_contents(LOG_DIR . '/queue.pid');
}
```

- 同时，通过判断代码目录的最新修改时间，在每一次定时任务时对比当前时间与最新代码修改提交时间的间隔，进行手动重启队列。

```
/**
 * Check and restart the queue listener.
 *
 * @return void
 */
private function checkQueueRestart() {
    // 查看app代码修改日期
    $code_modify_time = $this->findNewestFileModifyTime(app_path());
    $now = time();
    // 每分钟运行一次，判断代码修改时间与当前时间是否在60秒内，在则重启队列
    if (($now - $code_modify_time) < 60) {
        $restart_command = 'php artisan queue:restart';
        exec($restart_command);;
    }
}
    
/**
 * Find the newest modify time.
 *
 * @param $directory
 * @return string
 */
private function findNewestFileModifyTime($directory) {
    $last_modified_time = 0;
    $dir_m_time         = filemtime($directory);
    foreach (glob("$directory/*") as $file) {
        if (is_file($file)) {
            $file_m_time = filemtime($file);
        } else {
            $file_m_time = $this->findNewestFileModifyTime($file);
        }
        $last_modified_time = max($file_m_time, $dir_m_time, $last_modified_time);
    }
    return $last_modified_time;
}
```

> 总结Tips

1. 通过对于开发环境的具体场景，对比不同实现方案的优劣，选择相对最优的解决方案来解决问题。
2. 队列任务的配置需要在发布系统中对.env文件中的队列配置更新为**redis**，本地开发环境可以配置为**sync**同步执行便于调试。


## 业务对外——OpenID + JWT
> 系统对内通常采用OpenID完成系统登录，系统对外API接口采用基于Json Web Token完成接口权限验证。

> 以confluence系统页面登录以及对外数据接口需求为例，总结记录项目中OpenID以及JWT的实践过程。

### 基于OpenID-Connect-PHP的OpenID登录认证

> [OpenID开发文档](https://login.netease.com/download/oidc_docs/sdk/readme.html) 推荐采用OpenID-Connect-PHP。OpenID-Connect-PHP能够通过composer安装，使用便捷简单。

1. composer 安装

```
$ composer require jumbojett/openid-connect-php
```

2. 通过服务提供器(Service providers)注册OpenIDConnectClient

通过OPENID系统的站点接入添加对应系统登录页面的域名地址，获取对应的client id、client secret等信息。

```
[...]
$this->app->singleton(OpenIDConnectClient::class, function () {
    // 网易OPENID验证地址
    $netease_openid_url = config('openid.open_id_url');
    $client_id          = config('openid.open_id_client_id');
    $client_secret      = config('openid.open_id_client_secret');
    return new OpenIDConnectClient($netease_openid_url, $client_id, $client_secret);
});
[...]
```

3. OpenID认证流程

流程伪代码如下。
```
[...]
    // 初始化OpenIDConnectClient
    $oidc = app(OpenIDConnectClient::class);
    // 添加openid scope，默认申请括nickname、email、fullname
    // 敏感权限empno、title、dep需要自行申请
    $oidc->addScope(["openid", "fullname", "nickname", "email"]);
    // 设置回调地址，默认是回调请求接口本身，也可以设置为其他地址进行callback处理。
    // 需要注意的是，在Chrome等浏览器请求Header中会包含Upgrade-Insecure-Requests: 1, 初始化OpenIDConnectClient会判断自动转换返回HTTPS域名地址。可以通过手动设置回调地址来避免本地开发中的HTTPS的问题。
    $http_type          = ((isset($_SERVER['HTTPS']) && $_SERVER['HTTPS'] == 'on') || (isset($_SERVER['HTTP_X_FORWARDED_PROTO']) && $_SERVER['HTTP_X_FORWARDED_PROTO'] == 'https')) ? 'https://' : 'http://';
    $redirect_login_url = $http_type . $_SERVER['HTTP_HOST'] . $_SERVER['REQUEST_URI'];
    $oidc->setRedirectURL($redirect_login_url);
    
    try {
        // openid认证，调用该函数会使用302重定向认证页面进行认证
        $oidc->authenticate();
        // 获取认证返回参数，openid 返回scope属性
        $user_name = $oidc->requestUserInfo('nickname');
        } catch (OpenIDConnectClientException $e) {
            // 认证错误处理
            return $this->jsonFail(-1, 'open_id login fail');
        } catch (ErrorException $e) {
            // 认证错误处理
            return $this->jsonFail(-1, 'open_id login fail');
        }
[...]
```

在开发过程中，遇到偶发[undefined index $_SESSION['openid_connect_state']](https://github.com/jumbojett/OpenID-Connect-PHP/issues/93)问题，问题定位于在回调接口地址时无法获取到302跳转前存储在session中的openid_connect_state属性。后续可以通过将[OpenID的session单独存储处理](https://github.com/jumbojett/OpenID-Connect-PHP/pull/134/commits)，便于读取和设置。


### 基于jwt-auth的API接口认证

> JWT与传统 Web 的 Cookies 或者 Session 方式的认证不同的是，JWT 是无状态的，服务器上不需要对 token 进行存储，也不需要和客户端保持连接。jwt-auth

1. composer安装
```
$ composer require tymon/jwt-auth 1.*@rc
```
2. 添加服务提供器
```
 [...]
    Tymon\JWTAuth\Providers\LaravelServiceProvider::class,
 [...]
```
3. 发布配置文件
```
  $ php artisan vendor:publish --provider="Tymon\JWTAuth\Providers\LaravelServiceProvider"
```
4. 生成加密密钥
```
  $ php artisan jwt:secret
```
5. 自定义模型

> 常见官方文档示例和参考教程中，都是对于默认的User模型进行更新完成jwt-auth的认证实现，但是在项目中，由于User模型已经被使用，只能采用自定义的模型进行实现，整理总结自己的实现过程。

- 在自定义的模型中需要实现JWTSubject接口，定义JWTIdentifier以及JWTCustomClaims方法
- 由于jwt-auth调用默认的UserProvider，其中密码字段固定采用password。可以通过自定义实现UserProvider更改jwt-auth验证字段。

```
class DAHR extends Model implements AuthenticatableContract, AuthorizableContract,  AuthenticatableUserContract {
    use Authenticatable, Authorizable, CanResetPassword;

    protected $table = 'dahr_user';
    protected $connection = 'consume_mgr';
    protected $primaryKey = 'id';
    protected $fillable = ['username', 'password', 'corp_mail', 'create_time'];
    protected $hidden = ['id', 'password', 'remember_token'];

    public $timestamps = false;

    /**
     * @return mixed
     */
    public function getJWTIdentifier() {
        return $this->getKey();
    }

    /**
     * @return array
     */
    public function getJWTCustomClaims() {
        return [];
    }
}
```

6. 修改auth.php 

在config/auth.php文件中配置对应的自定义guard

```
'guards' => [
    [...]
    'dahr' => [
        'driver' => 'jwt',
        'provider' => 'dahr'
    ]
    [...]
],
```

7. 创建auth相关路由以及对应的控制器方法

以登录接口为例。

```
Route::group(['prefix' => 'auth'], function () {
    [...]
      // 登录
        Route::post('/login', 'DAHRAuthController@login');
        Route::options('/login', 'DAHRAuthController@login');
        [...]
});
```

```
/**
 * Get a JWT via given credentials.
 *
 * @param Request $request
 * @return \Illuminate\Http\JsonResponse
 */
public function login(Request $request) {
    $username = $request->input('username');
    $password = $request->input('password');

    try {
        $token = Auth::guard('dahr')->attempt(['username' => $username, 'password' => $password]);
        if (!$token) {
            return response()->json(['error' => 'invalid_credentials'], 401);
        }
    } catch (\Tymon\JWTAuth\Exceptions\Exceptions $e) {
        return response()->json(['error' => 'could_not_create_token'], 500);
    }
    
    return response()->json(compact('token'));
}
```

8. 创建auth相关中间件

- jwt-auth默认提供了jwt.auth和jwt.refresh两种中间件。简单介绍下自定义实现jwt接口认证的中间件，伪代码如下。

```
class JwtAPIAuth extends BaseMiddleware {
    /**
     * Handle an incoming request.
     *
     * @param  \Illuminate\Http\Request $request
     * @param  \Closure $next
     * @return mixed
     */
    public function handle($request, Closure $next) {

        if (!auth()->guard('dahr')->parser()->setRequest($request)->hasToken()) {
            return response()->json(['status' => 'Token not Provided'], 422); //缺少令牌
        }
        try {
            if (!$user = auth()->guard('dahr')->userOrFail()) {
                return response()->json(['status' => 'User not Found'], 401); //无用户
            }
        } catch (TokenExpiredException $e) {
            return response()->json(['status' => 'Token is Expired'], 401); //令牌过期
        } catch (JWTException $e) {
            return response()->json(['status' => 'Token is Invalid'], 400); //令牌无效
        }

        return $next($request);
    }
}
```

```
    [...]
    protected $routeMiddleware = [
        [...]
        // 添加jwt中间件
        'jwt.api.auth' => \App\Http\Middleware\JwtAPIAuth::class,
        [...]
    ];
    [...]
```

> 总结Tips

1. 采用OpenID-Connect-PHP实现OPENID认证，可以在一个接口内完成页面认证以及回调处理重定向等操作，更加简单高效。

## REF

- [如何从 phpcomposer 的 Composer 镜像迁移到 Laravel China 维护的镜像？](https://laravel-china.org/wikis/16722)
- [composer killed while updating](https://stackoverflow.com/questions/20667761/composer-killed-while-updating)
- [EnsureQueueListenerIsRunning](https://gist.github.com/ivanvermeyen/b72061c5d70c61e86875)
- [Laravel and JWT](https://blog.pusher.com/laravel-jwt/)

## 最后

自己在项目开发中遇到的问题，经过思考、搜索、请教之后实践解决，整理记录下来是一次最好的复盘和总结。自己从刚接触Laravel框架到现在，每一次问题的解决都是一次新的收获，希望自己可以在业务开发的实践过程中不断找到更加高效创新的解决方案。