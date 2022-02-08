# 开发中常用业务耗时分析与SQL优化工具实践

## 背景介绍

在负责部门平台系统系统的维护迭代与新需求开发过程中，页面访问以及页面操作缓慢是常见的反馈问题之一。

目前在不同平台系统中存在着不同的技术栈，例如PHP、Python以及Lua等，不同编程语言中的运行时间定位以及性能优化都属于共同的问题，在不同的业务框架中，定位分析方式不尽相同，但殊途同归。

## 业务耗时分析工具实践

### 常见问题反馈形式

从外部来看，主要有响应时间、吞吐量、错误率，从内部看，主要有CPU、内存、负载等运行指标。最常见为操作页面反馈的响应时间指标。

1. 页面响应时间

![1](http://pfp.ps.netease.com/kmspvt/file/5fe84fd768d86456817a7687VhybH29A01?sign=FPtEhioZGxVSHvWYsO0HUJz7WDU=&expire=1644342327)

- 在现场定位反馈问题时，往往先从Chrome浏览器中的Network的Timing获取复现请求情况。
- 对于服务器端，观察以下指标是否耗时异常
  - Request send：请求发送开始到发送完成消耗时间（上传时间）
  - Waiting(TTFB)：请求发送完成后到开始接收响应请求的时间，这个时间段主要包括服务器处理和返回数据网络延迟时间
  - Content Download：下载响应数据花费时间（下载时间）
- 在多数情况下，业务接口耗时大部分时间都在Waiting(TTFB)阶段，也是服务器端优化目标。

2. NGINX access log

```
10.2.12.26 - - [31/Aug/2020:23:00:06 +0800] "POST /select/query/filter_list_data HTTP/1.1" 200 37029 1798 "http://consumemgr.leihuo.netease.com/select/show" "Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/84.0.4147.135 Safari/537.36" - 1.859
```

同时在nginx的access_log中也可以判断接口响应时间，定位超时情况。
access_log中```$upstream_response_time```代表Nginx服务从上游业务接收响应返回数据花费时间，单位为秒，可以从这个字段判断定位问题。

### 耗时分析实践——PHP

XHProf是一个PHP轻量级的性能分析工具，具备提供函数层级的各种性能指标分析，并提供HTML的报告展示页面。

[XHProf](https://github.com/longxinH/xhprof)目前0.X版本是支持PHP5.X版本（当前最新版本为0.9.4），而在2.X版本后只支持7.0+以及8.0版本（最新版本为2.2.3）。

同时，也有相似的[tideways](https://github.com/tideways/php-xhprof-extension)扩展，兼容PHP7.X和Swoole，并可以结合[xhgui-branch](https://github.com/laynefyc/xhgui-branch)图形化展示工具进行展示。

以XHProf 0.9.4版本为例：

1. 安装(PECL安装以及源码编辑安装)

下载编译所需源码[xhprof-0.9.4](https://pecl.php.net/package/xhprof)

2. 编译安装扩展

```
which php-config
cd xhprof-0.9.4/extension/
phpize
./configure --with-php-config=<php-config-path>
make && make install
```
3. 修改php.ini启用xhprof扩展

```
[xhprof]
extension = xhprof.so
xhprof.output_dir = /tmp/xhprof
```
4. 修改xhprof页面展示对应nginx配置

修改root配置为xhprof.output_dir对应生成htmi文件路径

5. 添加入口

![2](http://pfp.ps.netease.com/kmspvt/file/5fe84fe68c5674bfaa48d0bcxzpLGQBF01?sign=rj3TPl2sdmwG4WpjRsbSrqpG_v8=&expire=1644342327)

图中为侵入式的性能检查方式。
- 通过```xhprof_enable()```开启xhprof
- 需要性能检查的业务逻辑代码
- 关闭xhprof
- 保存xhprof分析数据至对应本地目录

6. 运行结果 && 指标分析

![3](http://pfp.ps.netease.com/kmspvt/file/5fe84ff12dcade51295f518fSGbDrKOR01?sign=OTZ_8p0_3sx78Mqh_PtoFCWdUC4=&expire=1644342327)

在性能分析结果页面中，相关性能指标解释如下：

指标名称 | 指标说明
---|---
Calls | 方法被调用的次数
Calls% | 方法调用次数在同级方法总数调用次数中所占的百分比
Incl.Wall Time(microsec) | 方法执行花费的时间，包括子方法的执行时间。（单位：微秒）
Excl. Wall Time(microsec) | 方法本身执行花费的时间，不包括子方法的执行时间。（单位：微秒）
Incl. CPU(microsecs) | 方法执行花费的CPU时间，包括子方法的执行时间。（单位：微秒）
Excl. CPU(microsec) | 方法本身执行花费的CPU时间，不包括子方法的执行时间。（单位：微秒）
Incl.MemUse(bytes) | 方法执行占用的内存，包括子方法执行占用的内存。（单位：字节）
Excl.MemUse(bytes) | 方法本身执行占用的内存，不包括子方法执行占用的内存。（单位：字节）

### 耗时分析实践——Python

在业务开发过程中，排查定位接口中具体哪一个具体函数耗时是比较常见和突出的问题，line_profiler是一个很好的轻量小巧工具。

1. 安装（PIP 安装）
```
pip install line-profiler
```
2. 添加性能检测入口

第一种方式是在正常的接口访问过程中，在业务逻辑对应的函数中，通过装饰器```@func_line_time```标记需要调用分析的函数。

![4](http://pfp.ps.netease.com/kmspvt/file/5fe84ffe8c5674681f875127cRMohbIk01?sign=qkacbv39GXeU8MudBHjWb_YtWig=&expire=1644342327)

此外也可以通过@profile和kernprof实现python文件的逐行分析。

3. 运行结果

![5](http://pfp.ps.netease.com/kmspvt/file/5fe850088c5674bfaa48d0carq1QjN4Q01?sign=3VRlFrqDTkSA3MCb4KSHjqfGaTE=&expire=1644342327)

相关指标说明如下：

指标名称 | 指标说明
---|---
Line# | 行数
Hits | 调用次数
Time per Hit | 每次调用消耗的时间
%Time | 占总时间的百分比

### 耗时分析实践——Lua

1. 火焰图

火焰图是由 Brendan Gregg发明的一种可视化方法，用于展示某一种系统资源或性能指标。火焰图以一个全局的视野来看待时间分布，它从底部往顶部，列出所有可能导致性能瓶颈的调用栈。

常见的火焰图类型有On-CPU，Off-CPU等类型，如图所示：

![6](http://pfp.ps.netease.com/kmspvt/file/5fe8504b2dcade79b2c37af95sAEi8lj01?sign=s9ZFfppkmwRz7F51rr3jcjGzjTU=&expire=1644342327)

2. Openresty Xray

OpenResty XRay是新一代的应用性能管理(APM)产品，为OpenResty和其他开源软件使用，100%使用非侵入式的动态跟踪技术。

![7](http://pfp.ps.netease.com/kmspvt/file/5fe850292dcade51295f51ab3I7QtkRW01?sign=4TVBHXUehDbHiybWBoka5uvjgns=&expire=1644342327)

自己在本地开发环境中安装OpenResty XRay的页面情况如图所示，OpenResty XRay是一款商业产品，需要付费申请，但是免费版的试用期内还是可以让自己了解到最新的性能分析情况，虽然还有些使用上的问题与报错。

## SQL优化工具实践

### 慢查询日志格式
MySQL的慢查询日志是MySQL提供的一种日志记录，用来记录在MySQL中响应时间超过阈值的语句。

```
# Query_time: 0.480447  Lock_time: 0.000015 Rows_sent: 0  Rows_examined: 199216
use consumemgr;
SET timestamp=1596017859;
select * from `recruitmgr_consume` where `email` = 'qiaox@zju.edu.cn' and `project_id` = '1000000352';
```
以实际业务中慢查询日志为例，相关指标解释如下：

指标名称 | 指标说明
---|---
User@Host | 表示用户和慢查询查询的ip地址
Query_time | 表示SQL查询持续时间， 单位 (秒)
Lock_time | 表示获取锁的时间， 单位(秒)
Rows_sent | 表示发送给客户端的行数
Rows_examined | 表示服务器层检查的行数
set timestamp | 表示慢SQL记录时的时间戳


### SQL优化工具——SOAR

在实际业务中，函数调用最底层往往是与MySQL的数据交互，很多时候，我们的程序出现的“性能问题”，其实是我们自己写的那"坨"代码的问题，是自己Coding的问题，是MySQL的DML语句使用的问题，SQL查询优化问题，往往也是性能优化的常见方式。

SOAR（SQL Optimizer And Rewriter）是小米开源的一个对 SQL 智能优化与改写工具。

SOAR主要由语法解析器，集成环境，优化建议，重写逻辑，工具集五大模块组成。

SOAR具备以下特点：
  - 跨平台支持（支持Linux, Mac环境，Windows环境理论上也支持，不过未全面测试）
  - 支持基于启发式算法的语句优化，基于规则推荐
  - 支持复杂查询的多列索引优化（UPDATE, INSERT, DELETE, SELECT）
  - 支持EXPLAIN信息丰富解读

SOAR-WEB是基于SOAR的web图形化工具，SOAR针对SQL优化建议如图所示：

![8](http://pfp.ps.netease.com/kmspvt/file/5fe850338c5674bfaa48d0e4hEmXg9Z901?sign=3Uug_Vyy6xpqcdF8Id3GU7K9KOI=&expire=1644342327)

SOAR可以提供丰富多样的SQL建议，例如索引建议、SELECT建议、ORDER建议等，可以根据SOAR提供的优化建议，进行SQL改写，并通过explain或者慢查询日志进行实际测试验证。

## 后续实践

自己在不同变成语言项目的开发时间中，实践体会到了相同耗时定位，性能分析,SQL优化等工具实践对自己在业务逻辑开发中的帮助。

一方面往往是自己在代码层面存在多余循环判断以及重复逻，没有缓存以及异步处理导致的问题，通过工具精准定位到之后就可以更加高效地进行优化。

另一方面，目前自己实践的方式大部分是侵入式地在开发环境进行定位排查，对于线上环境下如何进行非侵入式地检测定位，不同项目可以可以统一分析方式也是后续实践的方向之一。

## 参考文献
- [https://developers.google.com/web/tools/chrome-devtools/network/understanding-resource-timing](https://developers.google.com/web/tools/chrome-devtools/network/understanding-resource-timing)
- [https://docs.nginx.com/nginx/admin-guide/monitoring/logging/](https://docs.nginx.com/nginx/admin-guide/monitoring/logging/)
- [https://github.com/phacility/xhprof](https://github.com/phacility/xhprof)
- [https://github.com/rkern/line_profiler](https://github.com/rkern/line_profiler)
- [https://openresty.org/en/build-systemtap.html](https://openresty.org/en/build-systemtap.html)
- [https://segmentfault.com/a/1190000030688899](https://segmentfault.com/a/1190000030688899)
- [https://openresty.com.cn/cn/xray/](https://openresty.com.cn/cn/xray/)
- [http://mysql.taobao.org/monthly/2017/02/05/](http://mysql.taobao.org/monthly/2017/02/05/)
- [https://tech.meituan.com/2017/03/09/sqladvisor-pr.html](https://tech.meituan.com/2017/03/09/sqladvisor-pr.html)
- [https://github.com/XiaoMi/soar](https://github.com/XiaoMi/soar)
- [https://github.com/xiyangxixian/soar-web](https://github.com/xiyangxixian/soar-web)
- [https://tech.meituan.com/2016/12/02/performance-tunning.html](https://tech.meituan.com/2016/12/02/performance-tunning.html)
- [https://blog.csdn.net/ROVAST/article/details/88431682](https://blog.csdn.net/ROVAST/article/details/88431682)