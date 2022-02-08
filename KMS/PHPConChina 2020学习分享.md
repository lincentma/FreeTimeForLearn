# PHPConChina 2020学习分享

> 作为自己目前业务系统开发的主要业务语言之一，PHP语言是自己入门和持续实践的语言之一。参加10月17至10月18号举办的第八届PHP开发者峰会，扩展了自己的眼界，转眼年底将至，自己也把自己的所看所想整理分享出来。

## 《What's new in PHP 8.0?》
来自PHP核心开发者Nikita Popov的分享就为我们来了PHP8.0的新特性解析。

在PHPCon China 2020大会结束后的感恩节，PHP 8.0的正式版本正式发布。与目前业务系统仍然存在PHP5.6、PHP7.3等众多时代相比，作为已经25岁的编程语言，仍然在不断地拥抱众多新特性。

以下节选与自己业务实践相关的特性：

1. JIT

PHP8.0在内核中添加了即时编译JIT编译器，极大地提高了性能。
在PHP5.5以后的版本中，PHP已经绑定了Opcache扩展，优化了每一次加载和解析PHP脚本的开销数据。而JIT是Opcache优化的基础上结合Runtime信息将字节码编译为机器码缓存起来，JIT不是Opcache替代，而是增强，在启用JIT的情况下，如果Zend底层发现特定字节码已经编译为机器码，则可以绕过Zend VM直接让CPU执行机器码，从而提高代码性能。
![1](http://pfp.ps.netease.com/kmspvt/file/5fe701a968d864027c13a498kRvAfO1101?sign=vZzo6hfBXyg4r6kggxxlCNEgjH8=&expire=1644342289)


对于同样在业务开发中使用的Openresty，Lua作为一种轻量和快速的脚本语言，LuaJIT即时编译器会将频繁执行的Lua代码编译成本地机器码交给CPU直接执行，执行效率更高，OpenResty会默认启用LuaJIT。

目前JIT广泛应用于主流的变成语言中，在实际开发应用过程中，开启Opcache往往是最有效地提升性能的方法之一。

2. 注解

对于PHP8.0新添加的注解（Attributes），第一反应是Java中的注解特性。注解是一种很好地非侵入式扩展功能的方式，常见的应用场景有编译运行时动态处理、数据检查、文档生成等，而在PHP8.0之前，PHP是不支持注解功能，只能通过第三方组件进行分析文档注释的方式进行实现。

![2](http://pfp.ps.netease.com/kmspvt/file/5fe701ca6158bcb879e0a839waU8Miik01?sign=_hZ0qu78OLlBSL4zcex4IW8wD60=&expire=1644342289)

![3](http://pfp.ps.netease.com/kmspvt/file/5fe701da6158bc1633510747cLy6F5qZ01?sign=6Up7WX5qIBCOjPJH0qlF1EYuQUs=&expire=1644342289)

而新的注解允许你添加元数据到PHP函数、参数、类等，通过反射以及实例化的方式获取注解内容。从这一点看PHP与Java相近了一些。

注解允许你添加元数据到 PHP 函数、参数、类等，这些元数据随后可以通过可编程方式获取，

3. 字符串与数字的比较更符合逻辑

PHP 8 比较数字字符串（numeric string）时，会按数字进行比较。 不是数字字符串时，将数字转化为字符串，按字符串比较。

而在之前的业务开发中，会遇到当```$sign_process_status```为字符串类型时，与数字0比较的时候永远为true。在以下的业务代码中，我们可以看到，如果```$sign_process_status```为字符串类型，那么会一直进入case 0的判断逻辑中。


```
    switch ($sign_process_status) {
        case 0:
            ...
            break;
        case 1:
            ...
            break;
        case 2:
            ...
            break;
        case 3:
            ...
            break;
        default:
            break;
    }
}
```


## 《次时代Swoole, 青年PHP的无尽探索》

这是一位年轻的Swoole核心开发人Twosee员来带的Swoole4.5最新特性以及Swoole相关应用介绍。

对于PHP来说，Swoole是一个PHP的异步、并行、高性能网络通信引擎。PHP + Swoole的异步协程的特性，对比与传统的FPM的方式，有了非常显著的提升。对于自己来说，通过引入Swoole来加速优化当前业务系统是一个非常实际有效的方式。

- swoole特性

![4](http://pfp.ps.netease.com/kmspvt/file/5fe7026668d8646f4ff936d8XGD8lrq801?sign=k79S__oESoi_jyS1t2dDbWgwjW4=&expire=1644342289)

- 项目中实践应用

目前在业务系统中常用的Laravel框架，也可以很好地与Swoole相结合。以[LaravelS](https://github.com/hhxsv5/laravel-s)为例，目前在美术外包系统中通过LravelS进行重构开发。通过Nginx的不同端口配置，可以灵活地在Swoole和FPM模式下切换。

- 协程不是银弹

![5](http://pfp.ps.netease.com/kmspvt/file/5fe7027868d86447995f4edewfSbja5q01?sign=glgnGt-ObG4WN7nRkv-_Jyye7XA=&expire=1644342289)

异步或是协程不是万能的银弹，需要针对业务系统中的具体需求进行判断是I/O的密集瓶颈，还是其他方面的瓶颈。同时Swoole对于开发调试和问题排查方面增加了难度和要求。


## 《PHP下AOP的实现与原理》
这是Hyperf框架作者黄朝辉带来的AOP相关分享。

- AOP概念

![6](http://pfp.ps.netease.com/kmspvt/file/5fe703e12dcade79b2c33a19bKjwbi1U01?sign=x4HtCfYQceREh3aZeDDd7otLRAU=&expire=1644342289)

AOP（面向切面编程）的概念我们并不陌生，更早地从Java中的Spring AOP等语言框架中了解。目前在自己实际开发过程中还没有真正应用到项目中。与OOP（面向对象编程）不同的是，从重用方向来看，OOP 实现的是继承树的纵向方向重用，而AOP沿横向方向重用。

- AOP应用场景

![7](http://pfp.ps.netease.com/kmspvt/file/5fe703ef2dcade79b2c33a1eSvkP6W6F01?sign=1MI1PxiDul-ktWEWPR_dnf-4QSQ=&expire=1644342289)

AOP能够以非侵入的方式来实现横向的数据校验、数据埋点统计、性能统计、异常统计等方方面面。

- AOP实现方式

PHP-Parse是分析PHP代码生成AST的库，通过PHP-Parse可以在原有代码中插入自定义的切面代码片段，并形成新的代理类，最终通反射实现注入。

![8](http://pfp.ps.netease.com/kmspvt/file/5fe703fe8c5674b572c76be2j7mRsytw01?sign=Mdwr8RLqnsQA7YKPY1OQFDWWFaU=&expire=1644342289)

- 项目中实践应用方向

实际上Laravel、Slim等PHP框架中的Middleware中间件方式，设计思想上与AOP类似，虽然在应用范围上比AOP小一些。通过AOP的方式，我们可以更加优雅地处理分页数据中的数据字段、类型的展示方式以及权限校验处理，而不用在具体的Service层业务逻辑代码中耦合具体的代码逻辑，避免重复代码，提高业务主流程的清晰度以及维护更高的可扩展性。

## 《从开源项目汲取养分助力业务发展》
这是由学而思高级技术专家景罗带来的开源项目学习与实际业务时间的分享。

- 阅读源码的方式

团队通过每天早晨的早读会的形式，通过书籍、文档、博客等渠道，借助工具分析、Trace日志，GDB源码等方式，学习整理熟悉的知识结构，并关联至业务开发过程中解决实际问题，并有书籍和公众号的产出成果。

![9](http://pfp.ps.netease.com/kmspvt/file/5fe7043068d864479d98fc041l4JL92t01?sign=Y-OopywMZ0grKUzW740EZTAkcHw=&expire=1644342289)

这是让自己感受比较深刻的点，对于自己来说，自己每天用到的编程语言，开发框架爱，中间件，数据库等，目前都是处于能用，会用的阶段，而为什么这样设计，内部具体的细节却有些一知半解。

- 学习开源项目解决业务需求痛点

![10](http://pfp.ps.netease.com/kmspvt/file/5fe7043a2dcade114b79de8buooOWO4f01?sign=Bjo0kPhffqa1gX67oRoSXzu67n0=&expire=1644342289)

在源码项目中的体现的优秀设计模式，代码习惯，以及系统设计方案，都会给我们的实际业务系统中带来益处。

## 自己的思考

1. 虽然这次鸟哥没有到现场，但包括韩天峰在内的各路大佬的精彩分享，让自己了解到了目前最近的技术发展趋势，以及业界最新的实践成果。
2. 通过学习不同种类的实践分享，对于自己目前的业务系统都有很多启发，无论是系统开发框架整合优化，代码性能优化等诸多方面。
3. 每一个大牛的分享，都代表着一段时间的时间总结，分享是最好的学习方式，用户输出倒逼输入，用热爱全力以赴，这也是自己不断努力的方向。


关于PHPCon China 2020的其他分享内容，都可以在这里学习[https://github.com/ThinkDevelopers/PHPConChina](https://github.com/ThinkDevelopers/PHPConChina)

## 参考文献
- [https://www.laruence.com/2020/06/27/5963.html](https://www.laruence.com/2020/06/27/5963.html)
- [https://blog.p2hp.com/archives/7577](https://blog.p2hp.com/archives/7577)
- [https://www.laruence.com/2020/06/12/5902.html](https://www.laruence.com/2020/06/12/5902.html)