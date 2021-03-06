# 「真®全栈之路 - DNS篇」故事从输入URL开始..... #

## 前言 ##

好久没写博客了，我原先的标题是 **“从输入url到页面加载完成的XXX”？**

但想着，这是别人嚼烂很多次的内容，缺乏挑战性，而且，页面操作过程中能优化的地方实在太多了。

那就干脆给自己挖个坑吧，好歹也在运维开发部待过一年的时间。

![](https://user-gold-cdn.xitu.io/2019/5/30/16b049445d763fb4?imageView2/0/w/1280/h/960/ignore-error/1)

本文将尝试从 **前后端或运维多个角度** ，来述说整个站点从解析到操作过程中的优化。

## 1. 流程回顾 ##

### 1. URL的输入到浏览器解析的一系列事件 ###

很多大公司面试喜欢问这样一道面试题，输入URL到看见页面发生了什么？,今天我们来总结一下。 简单来说，共有以下几个过程

* 浏览器中输入网址
* 域名解析（ ` DNS` ），找到IP服务器
* 发起 ` TCP` 连接， ` HTTP` 三次握手,发送请求（ ` Request` ）
* 服务器响应HTTP(Response)
* 浏览器下载资源 ` html css js images` 等
* 浏览器解析代码（如果服务器有 ` gzip` 压缩，浏览器先解压）
* 浏览器渲染呈现给用户

### 2. 结合操作页面到关闭标签页 ###

我们在页面渲染完成之后执行某些操作：

* 按钮重复点击
* 滚动操作
* 条件查询检索

姑且将以上都归为 **==> 8. 界面操作**

还在 **步骤3：发起TCP连接** 前插入：

* 浏览器允许的并发请求优化

下面就让我们从DNS解析开始...

## 2. DNS解析流程 ##

以 ` Chrome` 浏览器为例：

* 

Chrome浏览器 会首先搜索浏览器自身的DNS缓存。

(缓存时间比较短，默认只有1分钟，且只能容纳1000条缓存)

![](https://user-gold-cdn.xitu.io/2019/5/23/16ae54e9691b9eb3?imageView2/0/w/1280/h/960/ignore-error/1) 注： ` chrome://net-internals/#dns` 来进行查看 ` Chrome` 自身的缓存）

* 如果浏览器自身的缓存里面没有找到对应的条目，那么 ` Chrome` 会搜索操作系统自身的DNS缓存

* 

` Windows` - 在 ` Windows` 中查看 ` DNS` 缓存条目的过程非常简单。只需打开命令提示符并输入以下命令： ` ipconfig /displaydns` 。

* 

` Mac` - 在 ` Mac` 上查看 ` DNS` 缓存条目的过程略有不同。需要先打开控制台应用，从左侧边栏选择设备，然后输入： ` any:mdnsresponder` 进入搜索栏。接下来，打开命令行并输入以下命令：

` sudo log config --mode "private_data:on" sudo killall -INFO mDNSResponder 复制代码`

然后，返回控制台应用程序并查看缓存的DNS记录列表。例如，下面的屏幕截图显示了 ` wx.qlogo.cn` 的缓存 ` CNAME` 记录。

![](https://user-gold-cdn.xitu.io/2019/5/24/16ae57b80b0cfa2f?imageView2/0/w/1280/h/960/ignore-error/1)

* 

如果在系统的DNS缓存也没有找到，那么尝试读取 ` hosts` 文件。看看这里面有没有该域名对应的IP地址，如果有则解析成功。

* 注： ` Windows` 位于 ` C:\Windows\System32\drivers\etc` , ` Mac` 则是 ` /etc/hosts` 。
* **这种操作系统级别的域名解析通常会被不怀好意的人利用，通过修改你 ` hosts` 文件里的内容把域名解析到他指定的 ` ip` 地址上，造成所谓的域名劫持，所以将 ` hosts` 文件设置成了只读模式，防止被恶意篡改。**

* 

如果在 ` hosts` 文件中也没有找到对应的条目，浏览器就会发起一个 ` DNS` 的系统调用，请求本地域名服务器 ` localDNS` （ ` LDNS` ）来解析这个域名。

* 通过 ` UDP` 协议向 ` DNS` 的53端口发起请求，这个请求是递归的请求，也就是运营商的 ` DNS` 服务器必 须得提供给我们该域名的 ` IP` 地址)
* 第一次就会请求本地域名服务器（ ` LDNS` ）来解析这个域名，这台服务一般在你城市的某个角落，距离不会很远，并且他的性能很好，一般都会缓存域名解析结果，大概80%的域名解析到这里都结束了。
* 如果本地域名解析服务器也没有该域名的记录，则开始递归+迭代解析

> 
> 
> 
> 直到这里，浏览器能做的所有DNS解析已完成，接下来的步骤就是和服务器相关了。不想看的可以忽略。
> 
> 

![](https://user-gold-cdn.xitu.io/2019/5/30/16b0492545d7e0ca?imageView2/0/w/1280/h/960/ignore-error/1)

* 如果 ` localDNS` 仍然没有命中，就直接到 ` Root Server` 域名服务器请求解析。
* 根域名服务器返回给本地域名服务器一个所查询的主域名服务器（ ` gTLD Server` ）地址。 ` gTLD` 是国际顶级域名服务器，如 `.com` 、 `.cn` 、. ` org` 等，全球只有13台左右。
* 本地域名服务器 ` localDNS` 再向上一步返回的 ` gTLD` 服务器发送请求。
* 接受请求的 ` gTLD` 服务器查找并返回此域名对应的 ` Name Server` 域名服务器的地址，这个 ` Name Server` 通常就是用户注册的域名服务器，例如用户在某个域名服务提供商申请的域名，那么这个域名解析任务就由这个域名提供商的服务器来完成。
* ` Name Server` 域名服务器会查询存储的域名和IP的映射关系表，在正常情况下都根据域名得到目标IP地址，连同一个TTL值返回给 ` DNS Server` 域名服务器。
* 返回该域名对应的 ` IP` 和 ` TTL` 值， ` LDNS` 会缓存这个域名和 ` IP` 的对应关系，缓存时间由 ` TTL` 值控制。
* 把解析的结果返回给用户，用户根据TTL值缓存在本地系统缓存中，域名解析过程结束。

注：在实际的DNS解析过程中，可能还不止这11步(第1步其实可以忽略不计。)，如 ` Name Server` 可能有很多级，或者有一个 ` GTM` 来负载均衡控制，这都有可能会影响域名解析过程。

不想看文字可以看图：

![](https://user-gold-cdn.xitu.io/2019/5/26/16af09db4ce8438b?imageView2/0/w/1280/h/960/ignore-error/1)

## 3. DNS优化 ##

首先需要明确一点：DNS缓存存在多级缓存，从离浏览器的距离排序的话，有以下几种:

* 浏览器缓存
* 系统缓存
* 路由器缓存
* ` IPS` 服务器缓存
* 根域名服务器缓存
* 顶级域名服务器缓存
* 主域名服务器缓存。

如果每次都经过这么多步骤解析，是否太耗时间？如何减少该过程的步骤呢？ 那就需要 ` DNS` 优化了。而在前端优化中与 ` DNS` 有关的有两点：

* 减少 ` DNS` 的请求次数
* ` DNS` 预解析

` DNS` 作为互联网的基础协议，其解析的速度似乎很容易被网站优化人员忽视。现在大多数新浏览器已经针对DNS解析进行了优化，典型的一次 ` DNS` 解析需要耗费 ` 20-120` 毫秒，减少DNS解析时间和次数是个很好的优化方式。这里就不再述说，着重谈 ` DNS` 预解析吧。

### 3.1 前端： ` DNS prefetch` ###

![](https://user-gold-cdn.xitu.io/2019/5/30/16b0496c4dec0c43?imageView2/0/w/1280/h/960/ignore-error/1)

` DNS prefetch` 是让具有此属性的域名不需要用户点击链接就在后台解析，而域名解析和内容载入是串行的网络操作，所以这个方式能 减少用户的等待时间，提升用户体验 。

**默认情况下浏览器会对页面中和当前域名（正在浏览网页的域名）不在同一个域的域名进行预获取，并且缓存结果，这就是隐式的 ` DNS Prefetch`** 。

如果想对页面中没有出现的域进行预获取，那么就要使用显示的 ` DNS Prefetch` 。

其用法也很简单，只要在 ` link` 标签上加上对应的属性：

` /* 这是用来告知浏览器当前页面要做DNS预解析 */ <meta http-equiv= "x-dns-prefetch-control" content= "on" /> <link rel= "dns-prefetch" href= "//example.com" > 复制代码`

* 如果你的页面中需要大量访问不同域名的资源，可以利用这项技术加快资源的获取，从而获得更好的用户体验。
* 需要注意的是， ` DNS` 预解析虽好，但是也不能滥用。如果对多页面重复DNS预解析，会增加DNS的查询次数。

目前很多大型站点也应用了这一优化，例如：

淘宝：

![](https://user-gold-cdn.xitu.io/2019/5/24/16ae8a5b5f604bb3?imageView2/0/w/1280/h/960/ignore-error/1)

京东：

![](https://user-gold-cdn.xitu.io/2019/5/24/16ae8a680b3a45c3?imageView2/0/w/1280/h/960/ignore-error/1)

如果需要禁止隐式的 ` DNS Prefetch` ，可以使用以下的标签：

` <meta http-equiv= "x-dns-prefetch-control" content= "off" > 复制代码`

### 3.2 后端&运维： ` CDN` 与 ` HTTPDNS` ###

实际上后端&运维能做的优化有三种：

* ` CDN`
* ` HTTPDNS`
* ` DNS` 负载均衡

但稍微大型的Web站点，基本舍弃DNS负载均衡这一方案了，缺点太多。感兴趣的可以自行搜索了解。

#### 1. ` CDN` 与 ` DNS` 循环 ####

` CDN` , 全称是 ` Content Delivery Network` ，即内容分发网络。其目的是通过在现有的 ` Internet` 中增加一层新的 ` CACHE` (缓存)层，将网站的内容发布到最接近用户的网络”边缘“的节点，使用户可以就近取得所需的内容，提高用户访问网站的响应速度。

` DNS` 循环： 当权威 ` DNS` 发现一个域名映射多个 ` IP` 时，会使用 ` IP` 轮询的方式来将 ` IP` 平均分配给多个 ` DNS` 请求，从而达到负载均衡的效果。

**为什么需要 ` CDN` ?**

* 由于 ` DNS` 循环时平均分配，不能根据不同服务器的负载情况优化分配，甚至如果有一台服务器宕机了， ` DNS` 不能及时了解到该情况把该服务器的 ` IP` 分配出去，便会造成无法访问。
* 因此，在权威 ` DNS` 和 服务器之间加上一个 ` CDN` 层就显得很必要了。
* ` CDN` 在具备调度分配服务器能力的基础上，能够同步服务器运行情况，然后根据该情况及时适当调整调度策略，从而使得负载均衡能力大大提高。

**` CDN` 好处：**

* 解决服务器端的“第一公里”问题。
* 缓解甚至消除了不同运营商之间互联的瓶颈造成的影响。
* 减轻了各省的出口带宽压力。
* 缓解了骨干网的压力。
* 优化了网上热点内容的分布。

**` CDN` 的访问步骤：**

**（1）未部署CDN应用前：**

![](https://user-gold-cdn.xitu.io/2019/5/28/16afa4965377842e?imageView2/0/w/1280/h/960/ignore-error/1)

网络请求路径：

* 请求：本机网络（局域网）——》运营商网络——》应用服务器机房
* 响应：应用服务器机房——》运营商网络——》本机网络（局域网）

在不考虑复杂网络的情况下，从请求到响应需要经过 ` 3` 个节点， ` 6` 个步骤完成一次用户访问操作。

**（2）部署CDN应用后：**

![](https://user-gold-cdn.xitu.io/2019/5/28/16afa4b5169df7f5?imageView2/0/w/1280/h/960/ignore-error/1)

网络路径：

* 请求：本机网络（局域网）——》运营商网络
* 响应：运营商网络——》本机网络（局域网）

在不考虑复杂网络的情况下，从请求到响应需要经过 ` 2` 个节点， ` 2` 个步骤完成一次用户访问操作。

与不部署 ` CDN` 服务相比，减少了 ` 1` 个节点， ` 4` 个步骤的访问。极大的提高的系统的响应速度。

**以下是结合具体网络运维的步骤：**

` step1：用户向 local DNS发起请求查询输入域名对应的IP地址（若有缓存直接返回，否则去rootDNS查询）； step2： local DNS迭代向rootDNS查询，逐级迭代,rootDNS=>顶级DNS=>权限DNS； step3：获得权限DNS后， local DNS向权限DNS发起域名解析请求； step4：权限DNS通常会将域名CNAME 【如果有有CNAME则解析CNAME对应的CDN服务，否则的话默认为普通请求，直接返回解析到的IP】到另一个域名，这个域名最终会被指向CDN网络中的智能DNS负载均衡系统； step5：DNS负载均衡系统通过一些智能算法，将最合适的CDN节点IP地址返回给 local DNS； step6： local DNS将获得的IP地址返回给用户； step7：用户得到节点的IP地址后，向该节点发起访问请求； step8：CDN节点返回请求文件，如果该节点中请求的文件不存在，就会再回到源站获取这个文件，然后返回给用户。 复制代码`

![](https://user-gold-cdn.xitu.io/2019/5/30/16b0496067f91e09?imageView2/0/w/1280/h/960/ignore-error/1)

#### 2. ` HTTPDNS` ：解决 ` DNS` 挟持： ####

> 
> 
> 
> [来自：也谈 HTTPS - HTTPDNS + HTTPS](
> https://link.juejin.im?target=https%3A%2F%2Fblog.alswl.com%2F2016%2F11%2Fhttps-1%2F
> )
> 
> 

在惯有的印象中，很多时候觉得站点上完 ` HTTPS` 协议就 ` VANS` 。其实不然：

* 尽管使用了 ` HTTPS` 技术，部分邪恶的运营商，仍然使用 ` DNS` 污染技术，让域名指向的他们自己服务器。
* 而这些服务器并没有部署 ` SSL` 服务（就算部署了，也会触发 ` SSL` 证书 ` Common name` 不一致报警）， 导致 443 端口直接被拒绝。

运营商为了赚广告钱、省网间结算是不择手段的。 他们普遍使用的劫持手段是通过 ` ISP` 提供的 ` DNS` 伪造域名。 那有没有什么方法可以解决 ` DNS` 劫持呢？

业界有一套解决这类场景的方案，即 **` HTTPDNS`** ：

` HTTPDNS` 使用 ` HTTP` 协议进行域名解析，代替现有基于 ` UDP` 的DNS协议，域名解析请求直接发送到 ` HTTPDNS` 服务器，从而绕过运营商的 ` Local DNS` ，能够避 ` 免Local DNS` 造成的域名劫持问题和调度不精准问题。

` HTTPDNS` 的原理很简单，将 ` DNS` 这种容易被劫持的协议，转为使用 ` HTTP` 协议请求 ` Domain` <-> ` IP` 映射。 获得正确 ` IP` 之后， ` Client` 自己组装 ` HTTP` 协议，从而避免 ` ISP` 篡改数据。

![](https://user-gold-cdn.xitu.io/2019/5/28/16afa5475dd2f14c?imageView2/0/w/1280/h/960/ignore-error/1)

腾讯作为首家提供 ` HttpDNS` 服务的云服务商，有两篇相隔四年发布的文章，非常详细的揭示其中技术细节：

* [【鹅厂网事】千亿级HttpDNS服务是怎样炼成的]( https://link.juejin.im?target=https%3A%2F%2Fcloud.tencent.com%2Fdeveloper%2Farticle%2F1352018 )
* [【鹅厂网事】全局精确流量调度新思路-HttpDNS服务详解]( https://link.juejin.im?target=http%3A%2F%2Fmp.weixin.qq.com%2Fs%3F__biz%3DMzA3ODgyNzcwMw%3D%3D%26amp%3Bmid%3D201837080%26amp%3Bidx%3D1%26amp%3Bsn%3Db2a152b84df1c7dbd294ea66037cf262%26amp%3Bscene%3D2%26amp%3Bfrom%3Dtimeline%26amp%3Bisappinstalled%3D0%23rd )

## 未完待续... ##

下一篇的内容讲围绕 ` HTTP 优化` 的两个大方向：：

* 减少请求次数。
* 减少单次请求所花费的时间。

![](https://user-gold-cdn.xitu.io/2019/5/30/16b0494ed4f3b682?imageView2/0/w/1280/h/960/ignore-error/1)

与其对应的内容有：

* 浏览器允许的并发请求优化， ` nginx` 配置/ 域名发散收敛。
* 资源的压缩与合并， ` webpack/Gzip` 相关。
* 还有其它兴趣使然的内容...

## 作者掘金文章总集 ##

**需要转载到公众号的喊我加下白名单就行了。**

* [「真®全栈之路」Web前端开发的后端指南]( https://juejin.im/post/5cc02aacf265da039e1ff3fa )
* [「Vue实践」5分钟撸一个Vue CLI 插件]( https://juejin.im/post/5cb59c4bf265da03a743e979 )
* [「Vue实践」武装你的前端项目]( https://juejin.im/post/5cab64ce5188251b19486041 )
* [「中高级前端面试」JavaScript手写代码无敌秘籍]( https://juejin.im/post/5c9c3989e51d454e3a3902b6 )
* [「从源码中学习」面试官都不知道的Vue题目答案]( https://juejin.im/post/5c959f74f265da610c068fa8 )
* [「从源码中学习」Vue源码中的JS骚操作]( https://juejin.im/post/5c73554cf265da2de33f2a32 )
* [「从源码中学习」彻底理解Vue选项Props]( https://juejin.im/post/5c88e669f265da2d8f47792a )
* [「Vue实践」项目升级vue-cli3的正确姿势]( https://juejin.im/post/5c4a83e36fb9a049b13e91ba )
* [为何你始终理解不了JavaScript作用域链？]( https://juejin.im/editor/posts/5c8efeb1e51d45614372addd )

### 公众号 ###

![](https://user-gold-cdn.xitu.io/2019/3/29/169c519daed09b39?imageView2/0/w/1280/h/960/ignore-error/1) ![](https://user-gold-cdn.xitu.io/2019/3/29/169c818165484bb8?imageView2/0/w/1280/h/960/ignore-error/1)