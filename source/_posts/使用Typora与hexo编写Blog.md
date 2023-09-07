---
title: 使用Typora与hexo编写Blog
commends: false
date: 2023-09-06 17:24:17
categories:
tags:
typora-root-url: ..
---

## nodejs版本问题

> hexo4.2.0中不支持更高14+更高级的nodejs，因此需要将nodejs版本降级，使用工具nvm来进行操作
>
> 参考链接：https://juejin.cn/post/7094576504243224612  [nvm下载](https://github.com/coreybutler/nvm-windows/releases)

安装命令：

```shell
nvm install 12.17.0
```

使用命令：

```shell
nvm use 12.17.0
```

查看nodejs版本：

```shell
nvm -v
```

查看hexo版本：

```shell
hexo --version
hexo: 4.2.0
hexo-cli: 4.3.1
os: win32 10.0.23486
node: 12.17.0
v8: 7.8.279.23-node.37
uv: 1.37.0
zlib: 1.2.11
brotli: 1.0.7
ares: 1.16.0
modules: 72
nghttp2: 1.40.0
napi: 6
llhttp: 2.0.4
http_parser: 2.9.3
openssl: 1.1.1g
cldr: 37.0
icu: 67.1
tz: 2019c
unicode: 13.0
```

## Typora与hexo图像路径问题

解决的原理很简单，上传时将图片资源放到/source/imgs下，在Typora中使用相对路径即可。

在截图粘贴时，保证复制一份图片到imgs下的同名文件中即可。设置Typora的偏好，如下所示。

![image-20230907092903014](/imgs/%E4%BD%BF%E7%94%A8Typora%E4%B8%8Ehexo%E7%BC%96%E5%86%99Blog/image-20230907092903014.png)

同时为保证其使用相对路径，需设置图片的根目录在source下。其具体设置方式为**格式**->**图像**->**设置图片根目录**。

最好在创建md文件时即可执行该项设置，因此在模板中增加以下一项。

```json
title: {{ title }}
date: {{ date }}
commends: false
categories:
tags:
typora-root-url: ..
```



