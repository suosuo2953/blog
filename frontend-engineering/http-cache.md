# 网站的缓存控制策略最佳实践及注意事项

对于一个网站来讲，性能关乎用户体验，你在更短的时间内打开网站，你将会留住更多的用户。

缓存控制是网站性能优化中至为常见及重要的一环，好的缓存控制，除了使网站在性能方面有所提升，在财务方面也有重要提升: 更好的缓存策略意味着更少的请求，更少的流量，更少的峰值带宽，从而节省一大笔服务器或者 CDN 的费用。

缓存控制策略就是 http caching 的策略，通过标记相关 http header 来通知浏览器做出反应。化繁为简，最有效的策略往往是很简单的。在最简单的粗略下，你对 http cache 只需要了解一个 `Cache-Control` 的头部。

一个较好的缓存策略只需要两部分，而他们只需要通过 `Cache-Control` 控制

+ 带指纹资源: 永久缓存
+ 非带指纹资源: 每次进行新鲜度校验

![缓存控制策略](./assets/http-cache-control.jpg)

## 带指纹资源: 永久缓存

```
Cache-Control: max-age=31536000
```

天下武功，无坚不摧，唯快不破。资源请求最快的方式就是不向服务器发起请求，通过以上响应头可以对资源设置永久缓存。

1. 静态资源带有 hash 值，即指纹
1. 对资源设置一年过期时间，即 31536000，一般认为是永久缓存
1. 在永久缓存期间浏览器不需要向服务器发送请求

那为什么带有 hash 值的资源可以永久缓存呢？

**因为该文件的内容发生变化时，会生成一个带有新的 hash 值的 URL。** 前端将会发起一个新的 URL 的请求。

## 非带指纹资源: 每次进行新鲜度校验

```
Cache-Control: no-cache
```

1. 由于不带有指纹，每次都需要校验资源的新鲜度。(从缓存中取到资源，可能是过期资源)
1. 如果校验为最新资源，则从浏览器的缓存中加载资源

`index.html` 为不带有指纹资源，如果把它置于缓存中，则如何保证服务器刷新数据时，被浏览器可以获取到新鲜的资源？因此，使用 `no-cache` 每次对服务器进行新鲜度校验。

但即使每次校验新鲜度，也不需要每次都从服务器下载最新资源: **如果浏览器/CDN上缓存经校验是最新资源时**。这就属于协商缓存了，此时 http 状态码返回 304，代表 `Not Modified`，那隐藏在 `Not Modified` 背后的原理呢？

**幸运的是，关于协商缓存你无需管理，无需配置，** `nginx` 或者一些 `OSS` 都会自动配置协商缓存，而对于协商缓存，也有它们自己的算法。协商缓存背后是基于 `Last-Modified/ETag`。浏览器每次请求资源时，会携带上次服务器响应的 `ETag/Last-Modified` 作为标志，与服务端此时的 `ETag/Last-Modified` 作比较，来判断内容更改。

而在最底层，`Last-Modified` 往往通过文件系统(file system)中的 `mtime` 属性生成，而 `ETag` 提供比 `Last-Modified` 更精细的粒度，由 `hash` 或者 `mtime/size` 生成。可以参看我的每日一题：[http 响应头中的 ETag 值是如何生成的](https://github.com/shfshanyue/Daily-Question/issues/112)

## 一定要为你的资源添加 Cache-Control 响应头

我会经常接触到一些网站，他们的资源文件并没有 `Cache-Control` 这个响应头。究其原因，在于缓存策略配置这个工作的职责不清，有时候它需要协调前端和运维。

**那如果不添加 `Cache-Control` 这个响应头会怎么样？**

是不是每次都会自动去服务器校验新鲜度，很可惜，不是。 **此时会对资源进行强制缓存，而对不带有指纹信息的资源很有可能获取到过期资源。** 如果过期资源存在于浏览器上，还可以通过强制刷新浏览器来获取最新资源。但是如果过期资源存在于 CDN 的边缘节点上，CDN 的刷新就会复杂很多，而且有可能需要多人协作解决。

**那默认的强制缓存时间是多少**

首先要明确两个响应头代表的含义：

1. `Date`: 指源服务器响应报文生成的时间，差不多与发请求的时间等价
1. `Last-Modified`: 指静态资源上次修改的时间，取决于 `mtime`

`LM factor` 算法认为当请求服务器时，如果没有设置 `Cache-Control`，如果距离上次的 `Last-Modified` 越远，则生成的强制缓存时间越长。

用公式表示如下，其实 `factor` 介于 0 与 1 之间

``` js
MaxAge = (Date - LastModified) * factor
```

![LM factor](./assets/http-lm-factor.jpg)

## 分包：尽量减少资源变更


1. `webpack-runtime`: 应用中的 `webpack` 的版本比较稳定，分离出来，保证长久的永久缓存
1. `react-runtime`: `react` 的版本更新频次也较低
1. `vundor`: 常用的第三方模块打包在一起，如 `lodash`，`classnames` 基本上每个页面都会引用到，但是它们的更新频率会更高一些
1. `pageA`: A 页面，当 A 页面的组件发生变更后，它的缓存将会失效
1. `pageB`: B 页面
1. `echarts`: 不常用且过大的第三方模块单独打包
1. `mathjax`: 不常用且过大的第三方模块单独打包