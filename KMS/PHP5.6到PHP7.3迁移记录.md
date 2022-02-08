# PHP5.6与PHP7.3差异分析

> 由于系统运行环境同时有多个其他系统共同运行，为了避免系统运行资源抢占拥挤，申请到了筋斗云机器新环境，将原有系统迁移到新的运行环境，主要差异在于PHP版本的升级。

> PHP版本由PHP 5.6.37迁移至PHP 7.3.14-1

## 兼容性分析
- 从PHP官方给出的[不向后兼容的变更](https://www.php.net/manual/zh/migration70.incompatible.php)中，主要涉及到异常处理变更和相关语法废弃，对于业务逻辑以及数据库相关流程，与PHP5.6兼容，业务逻辑无需改动，即可平滑升级。
- 关于业务逻辑中使用频率非常高的foreach的方面，foreach遍历时不再修改数组指针。避免了引用遍历时，数组指针移动导致遍历结束后不做额外处理时使用位置异常的情况。
```
<?php
$array = [0, 1, 2];
foreach ($array as &$val) {
    var_dump(current($array));
}
?>

在PHP5.6中
int(1)
int(2)
bool(false)

在PHP7.3中
int(0)
int(0)
int(0)
```

## 性能差异分析
从著名云WordPresst托管服务商Kinsta这篇文章[The Definitive PHP 5.6, 7.0, 7.1, 7.2, 7.3, and 7.4 Benchmarks (2020)](https://kinsta.com/blog/php-benchmarks/)中可以看出，已经有超过68.6%的站点使用PHP7.3。

在其2019年报告中,以Laravel5.4.36为例（目前业务系统Laravel版本为5.4.30），对比PHP不同版本下，Laravel框架的基准测试情况。

![image](https://wiredgorilla.com/wp-content/uploads/2018/12/the-definitive-php-5-6-7-0-7-1-7-2-7-3-benchmarks-2019-20.png)


Laravel 5.4.36 PHP 5.6 benchmark results: 340.26 req/sec

Laravel 5.4.36 PHP 7.3 benchmark results: 717.06 req/sec

由此可见，PHP7.3版本的性能为PHP5.6版本下的2倍左右。

此外，以常见业务逻辑为例，简化为循环计数验证在不同PHP版本下常见业务逻辑处理速度情况的差异。
```
<?php
$start_time = microtime(true);
 
$index = 0;
for ($i = 0; $i < 10000000; ++ $i) {
    $index = $index + $i;
}
 
$end_time = microtime(true);
 
echo ($time_end - $time_start);

PHP5.6.37返回结果为：0.86726498603821
PHP7.3.14-1返回结果为：0.10278987884521
```

可以看出，PHP7.3版本处理速度提升8.437倍，考虑到新机器环境的CPU内核数和运行内存的增加，业务逻辑处理速度的提高能够有效降低接口响应时间。


- 数据库I/O差异

通过PDO的prepare和execute测试，数据库I/O差异不大。


## 迁移记录
1. Nginx配置中fastcgi_pass地址，与PHP版本相关，需要同步更新

```
    location / {
        #include snippets/fastcgi-php.conf;
        rewrite ^/(.*)$ /index.php?&query_string break;
        fastcgi_pass unix:/run/php/php7.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_read_timeout 300;
        include fastcgi_params;
    }
```

2. Laravel框架compose安装对应PHP扩展同步更新为PHP7版本

```
sudo apt-get install php-xml
sudo apt-get install php-mbstring
sudo apt-get install php-curl
sudo apt-get install php-gd
sudo apt-get install php-zip
sudo apt-get install php-mysql
```

3. MySQL驱动对比
  - Mysqlnd从PHP5.3开始已经默认为PHP的MySQL驱动。
  - 在其他Laravel框架系统中，运行环境的MySQL驱动为安装MySQL客户端对应版本（Libmysqlclient），而非Mysqlnd。
  - 两者区别在于Mysqlnd可以获取到数值类型，自动返回对应的类型参数。
```
Mysqlnd驱动：
Client API library version => mysqlnd 5.0.12-dev - 20150407 - $Id: 7cc7cc96e675f6d72e5cf0f267f48e167c2abb23 $

Libmysqlclient驱动：
Client API library version => 5.5.54
```

- 在构建本地开发环境时，因为MySQL驱动默认安装差异的不同，为了保持接口字段返回类型一致，可以通过在config/database.php文件中，添加options参数来强制PDO返回string类型。
```
'options'   => array(
    PDO::ATTR_STRINGIFY_FETCHES  => true,
),
```

## 后续优化
1. 迁移完成后，原有系统业务逻辑平稳运行。
2. 后续通过完善使用更多的PHP新版本特性来实现性能地提升。
