---
title: AWK分析nginx访问日志
date: 2017-07-15 23:06:17
tags: ["awk"]
category: Linux
---

如题，记录一次 awk 应用。通过分析 nginx 访问日志，得出 pv、uv等数据。

<!--more-->

### 背景

前段时间被要求通过分析 nginx 的访问日志得出项目的 pv、uv 和来源设备等占比。主要是该平台下面有多个 h5 子项目，每个子项目又是一个简单的 SPA 应用，所以就没有考虑到第三方统计平台。不过这个完成了好久，现在才写，主要还是个人懒癌发作。。

### AWK

首先介绍下 AWK 吧。

AWK 是一种 linux/unix 下的编程语言，强大的文本语言处理工具。它仅仅需要几行代码就能够完成复杂的文本处理工作。AWK 模式如下：

```bash
awk 'BEGIN{ commands } pattern{ commands } END{ commands }'
```

网上找到一张对应的流程图如下：

![awk 流程图](https://ws1.sinaimg.cn/large/cab372d4ly1fjmenvh1p1j20fb0f0t95.jpg)

BEGIN语句块在awk开始从输入流中读取行之前被执行，这是一个可选的语句块，比如变量初始化、打印输出表格的表头等语句通常可以写在BEGIN语句块中。

END语句块在awk从输入流中读取完所有的行之后即被执行，比如打印所有行的分析结果这类信息汇总都是在END语句块中完成，它也是一个可选语句块。

pattern语句块中的通用命令是最重要的部分，它也是可选的。如果没有提供pattern语句块，则默认执行 { print }，即打印每一个读取到的行，awk读取的每一行都会执行该语句块。

### 基本语法

AWK 的基本语法可参考 [linux命令大全](http://man.linuxde.net/awk)。

### 处理 nginx 日志

默认的 nginx 日志输出少了些我们需要的部分，所以我们需要先处理 nginx.conf 配置，打出我们需要的日志。这里先说下信息的统计思路吧。PV 很简单，每条 html 的访问记录算一条 PV，每个时间段的 PV 通过 nginx 日志里面的访问时间来判断，每天的 PV 通过将 nginx 日志切割，即每天的访问日志放到每天对应的文件里来判断。而 UV 则通过 cookie 来鉴别，即首先通过 javascript 在客户端种一个唯一的 cookie，每次访问记录中没有携带 cookie 的访问算一次 UV，客户端 cookie 的有效期到每天的0点。而来源及访问设备等可以通过浏览器携带的信息来判断。

### nginx 配置

我们采取的部分 nginx.conf 配置如下：

```js
http {
    include       mime.types;
    default_type  application/octet-stream;
    // 定义一种 project_log 格式的日志模式
    log_format project_log '$remote_addr - $time_local "$request" $status $sent_http_content_type UV: "$guid" $http_user_agent';
    sendfile        on;
    keepalive_timeout  65;
    client_max_body_size 2m;
    fastcgi_intercept_errors on;
    server {
	listen 443;
	server_name localhost;
	ssl on;
        ssl_certificate /test/server.crt;
	ssl_certificate_key /test/server.key;
    // 通过 cookie 正则匹配，如果有 _qingguoing 形式的 cookie 记录，就打到 log 中，默认是空
	if ($http_cookie ~* "_qingguoing=([A-Z0-9\.]*)" ) {
		set $guid $1;
	}
	if ( $request_method !~ GET|POST|HEAD|DELETE|PUT ) {
            return 403;
        }
	location /qingguoing/ {
            index index.html index.htm;
            error_page 404 ./404.html;
            // 日志输出到 logs/qingguoing.log 文件
	        access_log logs/qingguoing.log project_log;
            if (!-e $request_filename){
                return 404;
            }
        }
        error_page  404              /404.html;
    }
}
``` 
### nginx 日志切割脚本

```bash
#nginx 日志切割脚本
#!/bin/bash
# 设置日志文件存放目录
logs_path="/var/www/logs/";
#设置pid文件
pid_path="/var/www/logs/nginx.pid";
#重命名日志文件
mv ${logs_path}qingguoing.log ${logs_path}logs_daily/qingguoing_$(date -d "yesterday" +"%Y%m%d").log
#向ngin主进程发信号重新打开日志
kill -USR1 `cat ${pid_path}`
```

### AWK 处理脚本

```bash
BEGIN {
}
{
    # 过滤错误请求，只处理 html 200 请求，因为 html 没设缓存
    if ($8 != 200) next;
    if ($9 == "text/html") {
        # 来源匹配
        fromIndex = index($6, "from=");
        endIndex = index($6, "&");
        if (fromIndex != 0 && endIndex == 0) {
            # 增加 from= 的长度
            origin = substr($6, fromIndex + 5);
        } else {
            origin = substr($6, fromIndex + 5, endIndex);
        }
        # pathname 匹配
        pathLength = split($6, path, "/");
        # uv标记
        uv = 0;
        project = path[3];
        # 404.html
        if (pathLength > 4 || project == "404.html") next;
        if (origin != "platform") {
            # 日期 时间
            split($3, datetime, ":");
            if (timeArr[project] && timeArr[project] != datetime[2]) {
                # 新的一条记录，数据存到 mongodb 中并清空
                system("mongo 'yshow' --eval 'var project=\""project"\", date=\""dateArr[project]"\", time=\""timeArr[project]"\", pv=\""pvArr[project]"\", uv=\""uvArr[project]"\", pvFromOther=\""pvFromOtherArr[project]"\", pvFromTimeline=\""pvFromTimelineArr[project]"\", pvFromGroupmsg=\""pvFromGroupmsgArr[project]"\", pvFromSinglemsg=\""pvFromSinglemsgArr[project]"\",  uvFromOther=\""uvFromOtherArr[project]"\", uvFromTimeline=\""uvFromTimelineArr[project]"\", uvFromGroupmsg=\""uvFromGroupmsgArr[project]"\", uvFromSinglemsg=\""uvFromSinglemsgArr[project]"\", pvIos=\""pvIosArr[project]"\", uvIos=\""uvIosArr[project]"\", pvAdr=\""pvAdrArr[project]"\", uvAdr=\""uvAdrArr[project]"\", pvDeviceOther=\""pvDeviceOtherArr[project]"\", uvDeviceOther=\""uvDeviceOtherArr[project]"\"' ./logs_mongo_insert.js");
                delete dateArr[project];
                delete timeArr[project];
                delete pvArr[project];
                delete uvArr[project];
                delete pvFromOtherArr[project];
                delete pvFromTimelineArr[project];
                delete pvFromGroupmsgArr[project];
                delete pvFromSinglemsgArr[project];
                delete uvFromOtherArr[project];
                delete uvFromTimelineArr[project];
                delete uvFromGroupmsgArr[project];
                delete uvFromSinglemsgArr[project];
                delete pvIosArr[project];
                delete uvIosArr[project];
                delete pvAdrArr[project];
                delete uvAdrArr[project];
                delete pvDeviceOtherArr[project];
                delete uvDeviceOtherArr[project];
            }
            # save
            timeArr[project] = datetime[2];
            dateArr[project] = datetime[1];
            pvArr[project]++;
            if ($11 == "\"\"") {
                uvArr[project]++;
                uv = 1;
            }
            ios = index($0, "iPhone");
            adr = index($0, "Android");
            # 设备
            if (ios > 0) {
                if (uv) uvIosArr[project]++;
                pvIosArr[project]++;
            } else if (adr > 0) {
                if (uv) uvAdrArr[project]++;
                pvAdrArr[project]++;
            } else {
                if (uv) uvDeviceOtherArr[project]++;
                pvDeviceOtherArr[project]++;
            }
            # 其他
            if (fromIndex == 0 ) {
                if (uv) uvFromOtherArr[project]++;
                pvFromOtherArr[project]++;
            }
            # 微信朋友圈
            if (origin == "timeline") {
                if (uv) uvFromTimelineArr[project]++;
                pvFromTimelineArr[project]++;
            }
            # 微信群
            if (origin == "groupmessage") {
                if (uv) uvFromGroupmsgArr[project]++;
                pvFromGroupmsgArr[project]++;
            }
            # 好友分享
            if (origin == "singlemessage") {
                if (uv) uvFromSinglemsgArr[project]++;
                pvFromSinglemsgArr[project]++;
            }
        }
    };
}
END {
    for (project in pvArr) {
        system("mongo 'yshow' --eval 'var project=\""project"\", date=\""dateArr[project]"\", time=\""timeArr[project]"\", pv=\""pvArr[project]"\", uv=\""uvArr[project]"\", pvFromOther=\""pvFromOtherArr[project]"\", pvFromTimeline=\""pvFromTimelineArr[project]"\", pvFromGroupmsg=\""pvFromGroupmsgArr[project]"\", pvFromSinglemsg=\""pvFromSinglemsgArr[project]"\",  uvFromOther=\""uvFromOtherArr[project]"\", uvFromTimeline=\""uvFromTimelineArr[project]"\", uvFromGroupmsg=\""uvFromGroupmsgArr[project]"\", uvFromSinglemsg=\""uvFromSinglemsgArr[project]"\", pvIos=\""pvIosArr[project]"\", uvIos=\""uvIosArr[project]"\", pvAdr=\""pvAdrArr[project]"\", uvAdr=\""uvAdrArr[project]"\", pvDeviceOther=\""pvDeviceOtherArr[project]"\", uvDeviceOther=\""uvDeviceOtherArr[project]"\"' ./logs_mongo_insert.js");
    }
}
```

至此基本逻辑已写完。只需要在服务器里设置定时任务就行，例如每天的0点切割nginx日志，0点30开始处理前一天的日志存储统计信息。所以当前该思路唯一也是最大的缺陷就是没办法获取实时访问记录。后续如果有新思路会及时补充。

至此基本逻辑已写完。只需要在服务器里设置定时任务就行，例如每天的0点切割nginx日志，0点30开始处理前一天的日志存储统计信息。所以当前该思路唯一也是最大的缺陷就是没办法获取实时访问记录。

### 最后

前面也说了，最近才写这篇文章的主要原因是懒癌，导火索是前两天想通过 AWK 处理一个 hosts 文件到 charles 的 DNS 记录里，居然忘了 AWK 该怎么写了，果然好记性不如烂笔头。

本文代码地址：https://github.com/qingguoing/test/tree/master/awk/nginx

hosts 文件转 charles DNS 记录文件：https://github.com/qingguoing/test/tree/master/awk/DNS

本文参考内容：

1. nginx日志切割: http://www.nginx.cn/255.html
2. linux命令大全: http://man.linuxde.net/awk
