---
title: HTTP 系列之 Content-Type
date: 2018-11-12 15:18:20
tags: HTTP
category: HTTP
---

HTTP 请求头 Content-Type 字段学习理解

<!--more-->

### 前言
刚开始学习了解前端开发时，表单信息的提交用的还是 form 关键字，后来随着 Ajax 的盛行，就直接使用对应的封装好的库了，但是对于具体的原理和区分则早就已经忘了。本文重在区分常用的 Content-Type 值及其相关适用场景。

### 原理
HTTP 请求方法中，GET 请求的参数直接挂在 url 参数后面，POST 的请求参数需要放在请求报文中。而 Content-Type 的值就是定义请求报文的格式，常用的值有如下几种：

#### application/x-www-form-urlencoded

首先想到的第一个就是上面所说的 form 表单，采用的就是默认的 `application/x-www-form-urlencoded`。把对应的 key-value 拼接城 `key1=value1&key2=value2` 的形式，中间通过 & 进行连接。
在 chrome 浏览器下的请求中，默认会被转成 `key: value` 的对应形式。
![](./1.jpeg)

#### application/json

这个在现在 ajax 方式盛行的现况下，是最常见的请求方式了。表示发送到服务端的数据是一个 json 字符串，服务端可以转成 json 对象后方便使用。

#### multipart/form-data

适用于 form 表单的文件上传情形。如下形式：

```js
--aBoundaryString
Content-Disposition: form-data; name="myFile"; filename="img.jpg"
Content-Type: image/jpeg

(data)
--aBoundaryString
Content-Disposition: form-data; name="myName"

(data)
--aBoundaryString
```

其中 name 对应 form field 的名字，data 对应该 field 字段的值。而文件类型的后面会有对应的文件名，浏览器也会自动添加表明当前文件的 MIME 类型值。字段之间彼此被 `--aBoundaryString` 分开。

而此时 Content-Type 的值后面除了 `multipart/form-data` 之外，也会跟有 boundary=--aBoundaryString。例如：

```js
Content-Type: multipart/form-data; boundary=----WebKitFormBoundaryePkpFF7tjBAqx29L
```

该值主要用来告诉服务端，在此种情形下，获取每个字段的值得分界线是 `----WebKitFormBoundaryePkpFF7tjBAqx29L`。

### 总结

本文主要区分了几种常用的 Content-Type 值的使用场景和原理。