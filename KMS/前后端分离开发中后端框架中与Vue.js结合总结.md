# 前后端分离开发中后端框架中与Vue.js整合总结

> 在项目开发过程中，常见采用的是前后端分离的代码编写方式以及项目发布方式，在不同项目中采用的编程语言以及框架均有不同，但对应前端的前端框架都是Vue.js。在项目实际开发过程中，学习了解并总结不同框架下结合Vue.js的结合方式，有助于更加深入了解框架细节，提高后端开发维护的效率。


## Laravel+Vue

> 在校招系统与confluence数据统计系统中，后端采用Laravel框架，前端采用Vue框架

### 扩展安装
自Laravel 5.3 开始，Vue.js成为框架默认的标配，详细配置在package.json文件中。同时在项目开发中使用到的axios、vue-router等扩展也可以在package.json文件中进行自定义完善。

### 创建前端结构文件

vue主要文件结构在/recources/assets/js中，主要功能是挂载到页面的节点以及处理路由和渲染组件。

- 在resource/views/目录下新建index.blade.php文件
- 修改/recources/assets/js/app.js，导入Vue
- 校招系统Vue入口文件为t/resources/views/collect/list.blade.php
```
<!DOCTYPE html><html><head><meta charset=utf-8><meta name=renderer content=webkit><meta name=renderer content=webkit|ie-comp|ie-stand><meta http-equiv=X-UA-Compatible content="IE=edge,chrome=1"><meta name=robots content=all><title>leihuo</title><link rel=stylesheet href=/static/css/common20190311.css><link rel=stylesheet href=/static/css/iconfont20181101.css><script src=https://nie.res.netease.com/comm/js/jquery(mixNIE).1.11.js></script><script src=https://nie.res.netease.com/comm/js/nie/ref/swiper.4.1.6.js></script><link href=/static/css/app.6e63abeaac3ebdf4b7f9d001526777f7.css rel=stylesheet></head><body><div id=app></div><script type=text/javascript src=/static/js/manifest.c655f67c5eb353d1e91e.js></script><script type=text/javascript src=/static/js/vendor.29c5b5851e843ed3e8dd.js></script><script type=text/javascript src=/static/js/app.6a96c3d082b147f82544.js></script></body></html>
```
页面每次打包更新同时更新list.blade.php文件中内容

### 编译运行
通过发布系统npm run build自动完成构建后，同步至/public/static/css以及/public/static/js目录中


## Django+Vue
> 在美术外包系统中，后端采用Django框架，前端采用Vue框架

### 创建Vue.js项目
通过npm创建frontend项目并安装相关依赖

### 修改项目模板搜索路径
在settings.py文件中修改：
```
TEMPLATES = [
    {
        'BACKEND': 'django.template.backends.django.DjangoTemplates',
        'DIRS': [os.path.join(PROJECT_ROOT, 'frontend/dist/')],
        'APP_DIRS': True,
        'OPTIONS': {
            'context_processors': [
                'django.template.context_processors.debug',
                'django.template.context_processors.request',
                'django.contrib.auth.context_processors.auth',
                'django.contrib.messages.context_processors.messages',
            ],
        },
    },
]
```
### 修改静态文件搜索路径
在settings.py文件中修改：
```
STATIC_URL = '/frontend/dist/static/'
SITE_URL = '/'
STATICFILES_DIRS = (os.path.join(PROJECT_ROOT, 'frontend/dist/static/'),)
SITE_ENTRANCE_DIR = (os.path.join(PROJECT_ROOT, 'frontend/static/'),)
```
### 编译运行
通过npm run build完成构建


## 写在最后
- 本文通过记录总结项目时间过程中后端框架与Vue.js结合的方式细节，可以在后期API接口开发中，更加高效地符合前端数据处理方式，提高协同开发过程中的沟通协作效率。
- 参考文献：
  - [Laravel 和 Vue 的项目搭建：基础篇](https://segmentfault.com/a/1190000013212484)
  - [使用 Django，Vue.js 搭建 Web 项目](https://www.jianshu.com/p/a463e97def9c)