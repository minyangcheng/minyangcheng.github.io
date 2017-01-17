---
title: 深入研究http和https
date: 2016-11-05 15:52:37
tags:
---
任何数据在互联网上传输，都是经过了物理层、数据链路层、网络层、传输层、会话层、表示层和应用层7层，那http和https就属于应用层协议。https是在http基础上发展而来，其主要在http协议下面加上了ssl层，解决http网络传输数据不安全的问题。
<!-- more -->

## 数据是如何在网络上传输的

数据在网络上传输时是需要经过7层的，即：物理层、数据链路层、网络层、传输层、会话层、表示层、应用层。我这里讨论的主要是传输层和应用层，其它层的作用请自行搜索。
1. 传输层主要解决数据如何在网络中传输，tcp/ip即为传输层协议。
2. 应用层主要解决如何包装数据和解包数据，http、https即为应用层协议。

### tcp/ip

1. 三次握手建立连接
 * client对server说，我想向你传输数据行不行？
 * server对client说，我同意你向我传输数据，并且我也想向你传输数据行不行？
 * client对server说，我也同意你向我传输数据。

2. 四次握手关闭连接（由于tcp/ip是双工的，因此每个方向需要单独关闭）：
 * client对server说，我的数据传输完毕了，我请求关闭由我向你发送数据这个方向的连接（client->server）。
 * server对client说，我同意关闭连接（client->server）。
 * server对client说，我的数据传输完毕，我请求关闭由我向你发送数据这个方向的连接（server->client）。
 * client对server说，我同意关闭连接（server->client）。

下面用两张图说明建立连接和关闭连接过程：

![](/images/con_handshake.jpg)
![](/images/discon_handshake.jpg)

### http

> HTTP：是互联网上应用最为广泛的一种网络协议，是一个客户端和服务器端请求和应答的标准（TCP），用于从WWW服务器传输超文本到本地浏览器的传输协议，它可以使浏览器更加高效，使网络传输减少。

#### 与tcp/ip传输协议关系：

![](/images/http_tcpip.jpg)

#### http协议特点：

HTTP协议的主要特点可概括如下：
1. 支持客户/服务器模式。
2. 简单快速：客户向服务器请求服务时，只需传送请求方法和路径。请求方法常用的有GET、HEAD、POST。每种方法规定了客户与服务器联系的类型不同。由于HTTP协议简单，使得HTTP服务器的程序规模小，因而通信速度很快。
3. 灵活：HTTP允许传输任意类型的数据对象。正在传输的类型由Content-Type加以标记。
4. 无连接：无连接的含义是限制每次连接只处理一个请求。服务器处理完客户的请求，并收到客户的应答后，即断开连接。采用这种方式可以节省传输时间。
5. 无状态：HTTP协议是无状态协议。无状态是指协议对于事务处理没有记忆能力。缺少状态意味着如果后续处理需要前面的信息，则它必须重传，这样可能导致每次连接传送的数据量增大。另一方面，在服务器不需要先前信息时它的应答就较快。

#### http协议：

1. http url是一种特殊类型的URI，包含了用于查找某个资源的足够的信息。
2. http请求包含请求行、消息报头、请求正文。
3. http响应包含状态行、消息报头、响应正文。

*具体的协议内容和格式你可以通过Fiddler进行抓包查看，或者在chrome上按照postman插件查看。*

### https

> HTTPS：是以安全为目标的HTTP通道，简单讲是HTTP的安全版，即HTTP下加入SSL层，HTTPS的安全基础是SSL，因此加密的详细内容就需要SSL。

#### 与ssl、tcp/ip传输协议关系：

![](/images/https_ssl_tcpip.jpg)

#### https协议特点

由于其实在http上建立的，因此拥有http的特点。此外，https传输数据也更为安全。

#### https协议

我们其实可以将https理解为http+ssl，其主要是在ssl层构建了一层加密通道，之后的数据在这个加密通道上传输。

#### 让服务器支持https

如果要让你的服务器支持https，你需要向正规的CA机构申请证书，申请成功后会，会有公钥和私钥证书，然后在你的服务器进行证书配置证书。当client请求server数据时，server就会把公钥证书下发给client，client就去校验证书的合法性。

##### 如何校验证书的合法性

在server发送公钥证书给client的时候，发送的不是一个证书，而是一个证书链。客户端拿到证书链后，会从子代证书到根证书，向上开始查找是否有证书在客户端的证书集合中，如果有则信任。并且，一个证书的父代证书被信任了，其子代证书也会被自动信任。
android系统中一般会自带100多种证书，也就是说如果我们的证书是这些证书机构的子证书，那么我们的证书就会被自动信任。

##### 构建加密通道的过程

先上一张图：
![](/images/https_process.png)

1. client将自己支持的加密算法、hash算法发送给server
2. server从中选出一种加密算法和hash算法，并将公钥证书返回给client
3. client接受到公钥证书，会首先校验证书的合法性，如果信任该证书，则会产生一个随机数字（作为之后通信过程中对称加密的秘钥），然后取出证书中的公钥，用公钥对这个随机数字进行加密，然后和hash结果一并返回给server。
4. server接受到数据后，首先验证hash值的一致性，如果一致，则会取出私钥证书中的私钥，对这个随机数解密。然后再用解密后的随机数作为秘钥，加密一段数据，返回给client。
5. client接受到数据，用这个随机数解密接受到的数据，如果成功则握手完成，接下来的数据传输都用这个随机数进行加密。

## android如何https请求

我目前用的是okhttp框架,如果使用HttpUrlConnection也是大致一样的处理。
1. 信任指定的证书
```java
public OkHttpClient.Builder setCertificates(OkHttpClient.Builder builder,InputStream[] certificates) {
        try{
            SSLContext sslContext = SSLContext.getInstance("TLS");
            //certificate
            CertificateFactory certificateFactory = CertificateFactory.getInstance("X.509");
            KeyStore keyStore = KeyStore.getInstance(KeyStore.getDefaultType());
            keyStore.load(null);
            int index = 0;
            for (InputStream certificate : certificates){
                String certificateAlias = Integer.toString(index++);
                keyStore.setCertificateEntry(certificateAlias, certificateFactory.generateCertificate(certificate));
                try{
                    if (certificate != null) certificate.close();
                } catch (IOException e){
                }
            }
            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            trustManagerFactory.init(keyStore);
            sslContext.init(null, trustManagerFactory.getTrustManagers(), new SecureRandom());

            builder.sslSocketFactory(sslContext.getSocketFactory());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return builder;
    }
```

2. 信任所有证书（不推荐使用该方法）
```java
private static class UnSafeTrustManager implements X509TrustManager
    {
        @Override
        public void checkClientTrusted(X509Certificate[] chain, String authType)
                throws CertificateException
        {
        }

        @Override
        public void checkServerTrusted(X509Certificate[] chain, String authType)
                throws CertificateException
        {
        }

        @Override
        public X509Certificate[] getAcceptedIssuers()
        {
            return new X509Certificate[]{};
        }
    }
```
```java
public OkHttpClient.Builder setCertificates(OkHttpClient.Builder builder,InputStream[] certificates,InputStream bksFileStream, String password) {
        try{
            SSLContext sslContext = SSLContext.getInstance("TLS");

            sslContext.init(null, new TrustManager[]{new UnSafeTrustManager()}, new SecureRandom());
            builder.sslSocketFactory(sslContext.getSocketFactory());
        } catch (Exception e) {
            e.printStackTrace();
        }
        return builder;
    }
```

#### 参考
1. <http://blog.csdn.net/lmj623565791/article/details/48129405>
2. <http://www.cnblogs.com/li0803/archive/2008/11/03/1324746.html>
3. <http://blog.csdn.net/lintcgirl/article/details/52066875>
4. <http://www.mahaixiang.cn/internet/1233.html>
5. <https://www.douban.com/note/482351370/>