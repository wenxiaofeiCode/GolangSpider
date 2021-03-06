# Https在各种Web服务器下配置 #

### 前言 ###

前端很多情况需要用启动web服务器，而为了保证数据的安全性，都需要用 ` Https` 对传输的数据进行加密传输，而且有些 ` web-view` 只允许 ` https` 通过访问，所以学习怎么配置 ` https` 也成为大前端不可以少的功课之一。下面本妹子将先简单介绍下 ` Https` ,再依次介绍怎么在 ` Node` 、 ` webpack-dev-server` 和 ` nginx` 这三个最常见的前端web服务器下配置 ` Https` ，以及关于证书的扩展干货。

### Https简介 ###

` Https` 协议的主要作用可以分为两点： 1.建立一个信息安全通道，来保证数据传输的安全； 2.确认网站的真实性； 而 ` https` 的加密方式主要是非对称加密，所以会有一对密钥，分别是一个公钥(证书)和私钥。

![image.png](https://user-gold-cdn.xitu.io/2019/6/5/16b23380480f170d?imageView2/0/w/1280/h/960/ignore-error/1)

浏览器allen访问的时候，客户端会接受这个证书，并利用这个证书里的公钥对数据加密，加密后，服务端tony用自己的私钥解密就行了。

Web服务一般使用 ` OpenSSL` 工具提供的密码库，生成PEM、KEY、CRT等格式的证书文件。 其中 `.crt` 后缀一般存放证书， `.key` 后缀一般存放私钥， `.pem` 后缀的可以同时存放把CA证书和密钥。

### 生成证书和密钥 ###

可以直接用命令生成

` #使用以下命令生成一个证书密钥对 key.pem 和 cert.pem，它将有效期约10年（准确地说是3650天） openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout key.key -out cert.crt #或者直接生成到一个pem文件中 openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout all.pem -out all.pem 复制代码`

可以用 ` selfsigned` 插件生成

` const selfsigned = require ( 'selfsigned' ); var selfsigned = require ( 'selfsigned' ); var attrs = [{ name : 'commonName' , value : 'contoso.com' }]; //有效期约10年 var pems = selfsigned.generate(attrs, { days : 3650 }); fs.writeFileSync( 'all.pem' , pems.private + pems.cert, { encoding : 'utf-8' }); 复制代码`

### node配置https ###

用openssl生成all.pem

` openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout all.pem -out all.pem 复制代码`

在同目录编写 ` app.js` ,并运行 ` node app.js` ,启动 ` node` 服务监听 ` 88` 端口

` var https = require ( 'https' ); const fs = require ( 'fs' ); const path = require ( 'path' ); const options = { key : fs.readFileSync(path.resolve(__dirname, './all.pem' )), //密钥 cert: fs.readFileSync(path.resolve(__dirname, './all.pem' )) //证书 }; https.createServer(options, (request, response)=> { // 发送 HTTP 头部 // HTTP 状态值: 200 : OK // 内容类型: text/plain response.writeHead( 200 , { 'Content-Type' : 'text/plain' }); // 发送响应数据 "Hello World" response.end( 'Hello World\n' ); }).listen( 88 ); // 终端打印如下信息 console.log( 'Server running at https://127.0.0.1:88/' ); 复制代码`

![browser.png](https://user-gold-cdn.xitu.io/2019/6/5/16b2338049e64cf1?imageView2/0/w/1280/h/960/ignore-error/1)

具体代码可查看 [github.com/raoenhui/no…]( https://link.juejin.im?target=https%3A%2F%2Fgithub.com%2Fraoenhui%2Fnode-https.git )

### web-dev-server 配置https ###

web-dev-server配置很简单，只要将 ` https` 选项设成 ` true` 即可。

` module.exports = { devServer : { https : true } }; 复制代码`

web-dev-server原理 server.js里面判断如果没有证书和密钥，则用 ` selfsigned` 生成，并放在ssl目录中的 ` server.pem` 文件下,默认生成有效期为 ` 30` 天。

` const certPath = path.join(__dirname, '../ssl/server.pem' ); let certExists = fs.existsSync(certPath); ... if (!certExists) { log( 'Generating SSL Certificate' ); const attrs = [{ name : 'commonName' , value : 'localhost' }]; const pems = selfsigned.generate(attrs, { algorithm : 'sha256' , days : 30 , keySize : 2048 , extensions : [...] }); fs.writeFileSync(certPath, pems.private + pems.cert, { encoding : 'utf-8' }); } fakeCert = fs.readFileSync(certPath); 复制代码`

启动服务的时候引用证书和密钥

` options.https.key = options.https.key || fakeCert; options.https.cert = options.https.cert || fakeCert; ... this.listeningApp = spdy.createServer(options.https, app); 复制代码`

### Nginx 配置https ###

` # 进入nginx目录 cd nginx # 在key文件夹中用openssl生成 mkdir key && cd key && openssl req -newkey rsa:2048 -new -nodes -x509 -days 3650 -keyout all.pem -out all.pem 复制代码` ` # HTTPS server # server { listen 443 ssl; server_name localhost; #证书路径 ssl_certificate key/all.pem; ssl_certificate_key key/all.pem; ssl_session_cache shared:SSL:1m; ssl_session_timeout 5m; ssl_ciphers HIGH:!aNULL:!MD5; ssl_prefer_server_ciphers on; location / { root /HD/mycode/ test ; index index.html index.htm; } } 复制代码`

### 知识扩展 ###

#### 其他证书格式 ####

上面本妹子所讲的是对于前端比较常见的web服务器，而其实还有很多种服务器它们的证书格式是不一样的，而且可以互相转换，具体如下图所示

![2.png](https://user-gold-cdn.xitu.io/2019/6/5/16b23381f71d0e4c?imageView2/0/w/1280/h/960/ignore-error/1)

### CA ###

CA全称为Certificate Authority 证书授权中心。上面本妹子是自己生成证书，这个证书在互联网上是认为不安全的。只有通过CA机构下发的证书，浏览器和电脑才会认为是可靠安全的。 其中证书也有很多分类，可以向CA申请不同类型的证书。下面举例几种常见证书类型的区别

![3.png](https://user-gold-cdn.xitu.io/2019/6/5/16b233804d15aab5?imageView2/0/w/1280/h/960/ignore-error/1)

还可以用SSL证书在线检测工具，检测网站证书详细信息。 [www.chinassl.net/ssltools/ss…]( https://link.juejin.im?target=https%3A%2F%2Fwww.chinassl.net%2Fssltools%2Fssl-checker.html )

如www.jd.com

![image.png](https://user-gold-cdn.xitu.io/2019/6/5/16b233804ce9eb17?imageView2/0/w/1280/h/960/ignore-error/1) ![image.png](https://user-gold-cdn.xitu.io/2019/6/5/16b233808af118c2?imageView2/0/w/1280/h/960/ignore-error/1)

### 相关链接 ###

> 
> 
> 
> * 原文地址: [raoenhui.github.io/nodejs/2019…](
> https://link.juejin.im?target=https%3A%2F%2Fraoenhui.github.io%2Fnodejs%2F2019%2F06%2F04%2Fhttps%2F
> )
> * [Https网络安全架构设计与实践视频教程](
> https://link.juejin.im?target=https%3A%2F%2Fstudy.163.com%2Fcourse%2FcourseLearn.htm%3FcourseId%3D1209189811%23%2Flearn%2Flive%3FlessonId%3D1278788317%26amp%3BcourseId%3D1209189811
> )
> * [selfsigned说明文档](
> https://link.juejin.im?target=https%3A%2F%2Fwww.npmjs.com%2Fpackage%2Fselfsigned
> )
> * [数字证书](
> https://link.juejin.im?target=https%3A%2F%2Fwww.jianshu.com%2Fp%2F42bf7c4d6ab8
> )
> 
> 
> 

Happy coding .. :)