---
layout:     post
title:      "fiddler跳过五秒盾的小窍门"
date:       2024-08-14 15:27:00
author:     "Hux"
catalog: true
tags:
    - Scrape
---



## 失败的尝试

当我用尝试用fidder抓取某网站时，触发了cloudflare的五秒盾

![image-20240729153938830](/img/in-post/image-20240729153938830.png)



于是我尝试使用chrome自带的network工具，这次没有五秒盾，成功抓到了我需要的请求包。

![image-20240729154841326](/img/in-post/image-20240729154841326.png)

是一个异步的请求，我在network中重新发送这个请求是成功的，也就是说同样的参数是可以重复发送的，但是在fidder中却无法实现。

我认为是指纹的原因，但是我模拟了网站的指纹的指纹之后请求还是失败：

![image-20240729160801575](/img/in-post/image-20240729160801575.png)



## 成功抓取

经过朋友指点，我发现虽然网页的请求会被五秒盾阻止，但是网页加载成功之后的ajax请求不一定会被阻止。

因此我在网页加载成功之后，开启fidder抓包，这时成功抓到了请求包。

请求包可以用python模拟，完全不需要cookie 或者 任何加密参数。





## 解决疑惑

这时我再回头去对比fidder抓到的请求包，发现network抓到的请求包多了很多sec开头的参数。

删掉这些参数之后重试，发现请求成功。

![image-20240729174746000](/img/in-post/image-20240729174746000.png)

![image-20240729174807293](/img/in-post/image-20240729174807293.png)