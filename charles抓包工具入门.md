## charles抓包工具入门

​	手机不能直接启用开发者工具去获取小程序与app发出的请求，对于移动端开发来说，需要抓包去查看问题。日常来说当要盗图、拿视频，有时甚至需要刷单、抢单等情况，还能注入代码~~。

​	此外还能够去替换真实请求文件信息，将线上代码替换成自己的开发代码，下面会具体演示

### 1、原理

#### https加密解密过程

- 单向认证

<img src="https://raw.githubusercontent.com/caifeng123/pictures/master/169c44ac0af42de7%7Etplv-t2oaga2asx-watermark.awebp" width="70%" />

- 双向认证

<img src="https://raw.githubusercontent.com/caifeng123/pictures/master/169c44ac0ad58847%7Etplv-t2oaga2asx-watermark.awebp" width="70%" />

1. 客户端向服务端发送SSL协议版本号、加密算法种类、随机数等信息。
2. 服务端给客户端返回SSL协议版本号、加密算法种类、随机数等信息，同时也返回服务端的证书，即公钥证书。
3. 客户端使用服务端返回的信息验证服务器的合法性，包括：
   - 证书是否过期。
   - 发行服务器证书的CA是否可靠。
   - 返回的公钥是否能正确解开返回证书中的数字签名。
   - 服务器证书上的域名是否和服务器的实际域名相匹配。

验证通过后，将继续进行通信，否则，终止通信。

4. **服务端要求客户端发送客户端的证书，客户端会将自己的证书发送至服务端**。
5. **验证客户端的证书，通过验证后，会获得客户端的公钥**。 
6. 客户端向服务端发送自己所能支持的对称加密方案，供服务端进行选择。
7. 服务端在客户端提供的加密方案中选择加密程度最高的加密方式。
8. 服务端将加密方案通过**使用之前获取到客户端公钥进行加密**，返回给客户端。 
9. 客户端收到服务端返回的加密方案密文后，使用自己的私钥进行解密，获取具体加密方式，而后，产生该加密方式的随机码，用作加密过程中的密钥，使用之前从服务端证书中获取到的公钥进行加密后，发送给服务端。 
10.  服务端收到客户端发送的消息后，使用自己的私钥进行解密，获取对称加密的密钥。

#### charles代理

- charles作为中间层 客户端/服务端都会经过它
- charles - 服务端 ca证书
- charles - 客户端 charles证书

<img src="https://raw.githubusercontent.com/caifeng123/pictures/master/169c44ac0ae69a06%7Etplv-t2oaga2asx-watermark.awebp" />

#### 避免方法

经过观察后发现，可以发现中间人攻击的要点的伪造了一个假的服务端证书给了客户端，客户端误以为真。

- 客户端留存一份ca证书（不必是全部证书）

### 2、Charles

> Charles 是一个 HTTP 代理/HTTP 监视器/反向代理，它使开发人员能够查看他们的机器和 Internet 之间的所有 HTTP 和 SSL/HTTPS 流量。这包括请求、响应和 HTTP 标头（其中包含 cookie 和缓存信息）

- 官网Charles 现在不要钱了，随便下载来用
- 查看请求、响应和 HTTP 标头（其中包含 cookie 和缓存信息）

- 替换文件

  (对于线上请求的文件，可以利用插件将请求地址指向本地。帮你本地的javascript 文件替换线上的javascript 文件的插件，在前端开发中，可以在线上环境中，把本地更改过的js替换线上js 来做测试，方便快捷。)
  
- 对于chrome可以安装ReRes插件 - 替换PC端文件调试工具

  - 演示baidu->google

### 3、配置介绍

- 配置ssl proxying setting

> 设置你想抓到的ip和端口、当无所谓时 选择动态端口即可

![image-20210628202040820](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210628202040820.png)

- 配置proxy settings

![a](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210628201743816.png)

### 4、一些简单操作入门

1、获取请求信息

- 请求基本信息（链接、状态、协议、方法、请求头）
- TLS安全传输层协议
- 时间
- 大小
- cookies

![image-20210910163007602](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210910163007602.png)

2、替换远程设置

> Tools =》 Map remote
>
> 设置远程映射表，将请求A资源映射于B资源 类似上面提到的chrome插件

![image-20210628214649793](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210628214649793.png)

3、替换返回值

> Tools=>Rewrite
>
> 能对请求信息进行替换

![image-20210910170102616](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210910170102616.png)

### 5、手机抓包

- 配置ssl proxying setting
- 安装ca证书

> HELP => SSL Proxying =>xxx mobile device xxx

![image-2021091017494162](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210910174941626-20210910181945663-20210910182014316.png)

- 手机配置代理 获取ca

  ![4df5157a6a21853cf2ac90d944f8d6f2](https://raw.githubusercontent.com/caifeng123/pictures/master/4df5157a6a21853cf2ac90d944f8d6f2.png)

- 在 `Proxy =》 Access control setting` 配置上 手机的ip

  - 必须配置此处 才可以让外侧访问
  - pc端一样需要配置对应ip才可以 之后

![image-20210915160744506](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210915160744506.png)

- 访问chls.pro/ssl 获取认证证书
- 信任ca证书

![3d5f3136e39aaa50bf99f5129968bf56](https://raw.githubusercontent.com/caifeng123/pictures/master/3d5f3136e39aaa50bf99f5129968bf56.png)

- 此时就可以运行抓到手机端的请求了

![image-20210910182545388](https://raw.githubusercontent.com/caifeng123/pictures/master/image-20210910182545388.png)





翻墙url http://pac.internal.baidu.com/bdnew.pac

