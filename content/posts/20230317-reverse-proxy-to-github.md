---
title: "利用CloudFlare的Worker反代Github站点"
date: 2023-03-17T12:55:54+08:00
draft: false
---
由于最近Github.com站点经常有Timeout或Reset的情况发生，所以想零成本的做一个Github的代理。

这里使用CloudFlare的免费Worker功能，对Github做全球代理。同时Cloudflare会对静态页面做CDN缓存，也会加快我们访问github的速度。

很少的几行代码，解决了很大的事情。

<!--more-->

# 前提

首先，需要注册一个[Cloudflare](https://www.cloudflare-cn.com/)的免费账号

然后，由于Cloudflare提供的 [*.workers.dev](https://github-proxy.xxfe.workers.dev/) 域名在国内经常有访问不到的问题，所以最好拥有一个自定义域名，并托管到Clareflare。


# 在Cloudflare创建Worker并设置自定义域名

在Cloudflare的控制台创建Worker，并将如下代码复制到Worker中，保存并发布。

```javascript
export default {
  async fetch(request, env) {
    const _url = new URL(request.url);
    const hostname = _url.hostname
    _url.hostname = "github.com"
    const req = new Request(_url, request);
    req.headers.set('origin', 'https://github.com');
    
    const res = await fetch(req);
    let newres = new Response(res.body, res);

    let location = newres.headers.get('location');
    if (location !== null && location !== "") {
      location = location.replace('://github.com', '://'+hostname);
      newres.headers.set('location', location);
    }
    return newres 
  },
};
```

# 设置自定义域名

在worker的配置中，添加CustomDomain，我这里使用的是 [https://gh.xxfe.com](https://gh.xxfe.com)，可直接代理到github.com。


# 注意事项

## 请求-修改Origin头

Github对于Post请求，会检查Origin头，如果不是github自身的域名，会直接返回`422`错误。

这里使用以下代码，将转发给Github的header覆盖Origin头

```javascript
   req.headers.set('origin', 'https://github.com');
```

## 响应-修改Location头

当用户没有登录的时候，响应的Location字段会被设置为https://github.com/login，这里为了避免浏览器跳转到github官网，修改了location字段到请求域。

```javascript
    let location = newres.headers.get('location');
    if (location !== null && location !== "") {
      location = location.replace('://github.com', '://'+hostname);
      newres.headers.set('location', location);
    }
```

# 效果

打开反代的网站 https://gh.xxfe.com ，即可看到如下效果（**此站点会一直保留，给需要的朋友使用**）:
{{< image src="/images/20230317-github/github-homepage.png" width="80%" max-width="600px" title="reverse github preview" >}}


# 使用

以`git clone`为例，以前仓库路径为：[https://github.com/anhk/ztserver.git](https://github.com/anhk/ztserver.git)，那么将路径中的`github.com`修改为 `gh.xxfe.com` 即可

```bash
$ git clone https://gh.xxfe.com/anhk/ztserver.git
正克隆到 'ztserver'...
remote: Enumerating objects: 32, done.
remote: Counting objects: 100% (32/32), done.
remote: Compressing objects: 100% (24/24), done.
remote: Total 32 (delta 11), reused 26 (delta 5), pack-reused 0
Unpacking objects: 100% (32/32), done.
$
```
