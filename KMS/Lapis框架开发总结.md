# Lapis框架开发总结
> Lapis框架是一个基于Openresty的Web开发框架，通过Lua/MoonScript编写，而OpenResty是一个基于Nginx与Lua的高性能Web平台。在部门中有许多业务场景正在运用Openresty，通过美术招投标系统的项目开发，自己更加深入了解Openresty以及Lapis框架的应用，并自己的开发总结当做抛砖引玉，不断学习。

## Lapis框架介绍[^1]
通过`lapis new --lua`创建一个新的Lapis项目后，代码结构主要代码结构如下：
├── app.lua
├── mime.types
├── models.lua
├── nginx.conf
借鉴之前开发所用Laravel框架代码目录，按照MVC模式，将代码分为以下部分：
* config——接口、用户权限配置文件
* lib——第三方组件库
* controller——控制层文件
* service——服务层文件
* model——模型层文件
* console——脚本执行文件
* util——基础工具类文件

## Lapis框架开发总结

### Lua的简洁语法[^2][^3]
1. table

在LuaJIT中，table是唯一的数据结构，在其他编程语言中常见的数组、集合、哈希等数据结构概念，被统一为table。
在实际使用场景中，我们会使用到例如判断元素是否在数组中，两个数组是否粗在交集等需求，这个时候就需要利用table自带的函数库进行封装实现了：

* 判断数组中是否包含指定元素

```
function _TABLE_UTIL.table_contains(table, element)
    if table and next(table) then
        for _, value in pairs(table) do
            if value == element then
                return true
            end
        end
    end

    return false
end
```
* 判断两个数组的交集&&差集

```
function _TABLE_UTIL.table_intersect(m,n)
    local result = {}
    for index_m , value_m in pairs(m) do
        for index_n , value_n in pairs(n) do
            if (value_m == value_n) then
                table.insert( result, value_m)
            end
        end
    end
    return result
end
```

```
function _TABLE_UTIL.table_difference(a, b)
    local aa = {}
    for k, v in pairs(a) do
        aa[v] = true
    end
    for k, v in pairs(b) do
        aa[v] = nil
    end
    local ret = {}
    local n = 0
    for k, v in pairs(a) do
        if aa[v] then
            n = n + 1
            ret[n] = v
        end
    end
    return ret
end
```

2. metatable

metatable（元表）引申于table，类似于操作符重载，常见于Openresty组件库中，例如lua-resty-mysql、lua-resty-redis等。
处理metatable的函数主要有：
* `setmetatable(table, metatable)` 为一个table设置元表
* `getmetatable(table)` 获取table的元表
常用的重载元表的元方法（metamethod）
* `__tostring` 重载table对应的元表的`tostring`方法，可以输出table的数据信息
* `__index` 在table中查找一个元素，当在table中没有找到时，会继续从元表中的`__index`查询
* `__call` 实现table可以被调用

3. 冒号和点号

类似于其他语言的语法糖，冒号也是Lua中的一个语法糖，冒号与点号的区别就在于可以省略函数的第一个参数self。

4. 关于空的判断

在Lua中，关于空的概念，在不同的应用场景中有不同的具体定义：
* nil 没有赋值的变量，对nil进行索引会导致异常
* '' 空字符串
* {} 空table
* [] 空数组，需要借助cjson库实现，`cjson.empty_array`
* null 空元素，需要借助ngx实现，`ngx.null`

在实际开发过程，需要对不同的类型的情况进行判断，首先需要对nil进行判断，然后再对不同的类型进行判断。

5. 魔法字符

* Lua中的魔法字符：. % + * - ? [ ] ^ ( ) $
* 转义：通过添加%进行转义
* 应用场景，对于版本类型编号(X.X.X)的排序时，需要通过点进行分隔排序，需要对魔法字符进行转义


```
function _FILTER_CONTROLLER.sort_by_link_number(a, b)

    local a_link_number = a.number
    local b_link_number = b.number

    local a_link_number_list = {}
    local b_link_number_list = {}

    -- 魔法字符转义
    local a_link_id_result = string_find(a_link_number, '%.')
    if a_link_id_result then
        a_link_number_list = string_util.split(a_link_number, '%.')
    else
        table.insert(a_link_number_list, a_link_number)
    end

    local b_link_id_result = string_find(b_link_number, '%.')
    if b_link_id_result then
        b_link_number_list = string_util.split(b_link_number, '%.')
    else
        table.insert(b_link_number_list, b_link_number)
    end

    local compare_length = math.min(#a_link_number_list, #b_link_number_list)

    if compare_length > 0 then
        for i = 1, compare_length do
            local a_number = tonumber(a_link_number_list[i])
            local b_number = tonumber(b_link_number_list[i])
            if a_number ~= b_number then
                return a_number < b_number
            end
        end
    end

    return #a_link_number_list < #b_link_number_list
end
```

### Openresty组件库使用

> 在Openresty中，尽可能优先使用Openresty API，其次为LuaJIT API，最后考虑使用Lua库提供的API。这样可以有效避免结果差异和效率问题

> 在开发CRUD API过程中，需要结合多种组件进行开发，简要介绍常用的Openresty组件库。


1. cjson
Lua cjson是一个用于处理json的lua模块。由于json编码时无法区别array和dict这两个不同的类型，因为Lua中只有table一种数据类型。在前后端分离的项目中，往往需要接口提供空数组在表示为空的字段。

实现空数组主要有以下方式：
* 使用`encode_empty_table_as_object`方法

这种方法是修改全局默认值，会影响所有的table。
```
cjson.encode_empty_table_as_object(false)
```

* 使用`cjson.empty_array`赋值给table

这种方法方式会修改table的类型为userdata，在进行索引操作操作时，需要首先对类型进行判断。
```
local cjson = require "cjson"
local json  = cjson.empty_array
```


* 使用`cjson.empty_array_mt`标记table

这种方法方式通过元表方式对table进行设置。
```
local cjson = require "cjson"
local json  = {}
setmetatable(json, cjson.empty_array_mt)
```

2. openid[^4]

在项目开发中，需要与openid登录系统打通实现用户通过openid登录。
lua-resty-openidc常用参数如下：
```
    local opts = {
        token_endpoint_auth_method = "client_secret_basic",
        redirect_uri_scheme = redirect_uri_scheme,
        discovery = open_id_discovery_uri,
        redirect_uri = redirect_uri,
        client_id = client_id,
        client_secret = client_secret,
        ssl_verify = "no",
        scope = scope,
        force_reauthorize = false,
        session_contents = { id_token = true },
        revoke_tokens_on_logout = true,

        logout_path = logout_path,
        redirect_after_logout_uri = background_supplier_uri,
        post_logout_redirect_uri = background_supplier_uri
    }

    local target_url
    local unauth_action = nil
    local session_opts = { cookie = { domain = domain_uri } }

    local res, err = require("resty.openidc").authenticate(opts, target_url, unauth_action, session_opts)
```

* revoke_tokens_on_logout：退出登录后通知授权服务器之前获取的token信息无效
* logout_path：退出登录接口路径
* redirect_after_logout_uri：退出登录后跳转URL
* post_logout_redirect_uri：退出登录后跳转URL
* session_opts：设置session对应的domain信息

在openid登录过程中，登录接口（login）、返回接口（callback）以及退出登录接口（logout）都可以指向同一个controller方法，会根据接口请求路径来自动判断对应的登录逻辑。

3. redis[^5]

项目中在测试环节中采用单点redis，正式环境中采用基于哨兵模式的主从集群。
采用lua-resty-redis-connector库实现哨兵模式的redis连接。

redis连接与执行代码如下：
```
--
-- 所有的查询以及Set等操作都是通过这个 func 进行处理
-- @param self
-- @param func
--
function _S_REDIS_SENTINEL.exec(self, func)

    local rc = redis_connector.new()

    rc:set_connect_timeout(self.connect_timeout)
    rc:set_read_timeout(self.read_timeout)
    rc:set_connection_options(self.keepalive_timeout)

    -- Connections via Redis Sentinel
    local master, err = rc:connect({
        url = self.url,
        sentinels = self.sentinels
    })

    if not master then
        ngx.log(ngx.ERR, "Cannot connect, host: " .. self.url .. ", sentinels: " .. cjson.encode(self.sentinels))
        return nil, err
	end

    local res, err = func(master)
    if err then
        ngx.log(ngx.ERR, "redis error : ", err)
    end

    if res then
        local ok, err = rc:set_keepalive(master)
        if err then
            ngx.log(ngx.ERR, "redis set_keepalive : ", err)
            return nil, err
        end
        if not ok then
            master:close()
        end
    end

    return res, err
end

---
-- 对 Redis 进行 new 操作
-- @param opts
--
function _S_REDIS_SENTINEL.new(opts)
    local config = opts or {}

    local self = {
        -- lua-resty-redis-connector参数配置
        connect_timeout = config.connect_timeout or 50,
        read_timeout = config.read_timeout or 5000,
        keepalive_timeout = config.keepalive_timeout or 30000,

        url = config.url or "",
        sentinels = config.sentinels or {}
    }
    return setmetatable(self, mt)
end
```

4. mysql

在Lapis框架中，没有如Laravel框架中的ORM，直接采用的是原生SQL语句拼接并执行。

针对于SQL注入问题，可以采用`ngx.quote_sql_str`对字符进行转义，返回值会带上引号。

主要注意的是，如果是需要更新为NULL的话，`ngx.quote_sql_str`会将NULL也转义加上引号导致执行失效，这种特殊情况需要单独判读处理。

### Lapis框架开发Tips

* 切换当前执行环境配置
通过添加lapis_enviroment.lua文件来指定当前执行环境配置
```
-- lapis_environment.lua
return "production"
```
* nginx配置
通过nginx作为反向代理服务器，将访问URL映射到跨域访问的Web服务器中。
但需要注意的是，nginx的反向代理的缓存时间设置，避免前端页面更新后由于缓存设置导致无法显示最新的资源。
通常nginx的缓存功能有两种方式：
    * expires方式，通过服务器在HTTP相应头部插入cache-control的字段实现缓存解决带宽资源。
    * cache方式，在本地磁盘创建一个文件目录，根据设置，将请求的资源以K-V形式缓存。

## Lapis使用感受

1. lua的简洁与高效

在熟悉和使用Lua进行项目业务开发的过程中，Lua语言的显著特征就是简洁和高效。Openresty中采用的LuaJIT使得Lua的性能更进一步。
2. Openresty生态完整性[^6]

在基于Openresty平台中，具备了多种语言的结合能力：
* Lua：这个就是openresty正在做的事情
* PHP：ngx_php功能是为nginx模块嵌入php脚本语言
* Go ：gopher-lua，一个纯Golang实现的Lua虚拟机
 
## 参考文献

[^1]: Lapis-快速开始 https://www.shengguocun.com/blog/2018/10/23/lapis-getting-started/
[^2]: OpenResty 最佳实践 https://moonbingbing.gitbooks.io/openresty-best-practices/
[^3]: Lapis文档 http://leafo.net/lapis/reference/command_line.html
[^4]: lua-resty-openidc https://github.com/zmartzone/lua-resty-openidc
[^5]: lua-resty-redis-connector  https://github.com/ledgetech/lua-resty-redis-connector
[^6]: 当 OpenResty 遇上教育行业 https://www.upyun.com/opentalk/422.html