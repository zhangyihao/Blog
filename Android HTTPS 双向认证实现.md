# Android HTTPS 双向认证实现 #

在做项目的过程中，碰到了App需要使用双向认证的问题，记录下解决方法。

## 什么是双向认证？ ##
简单来说，在一次请求中，客户端需要校验服务端证书合法性，服务端同时校验客户端合法性。
详细过程为：
1. 客户端向服务端发送SSL协议版本号、加密算法种类、随机数等信息。
2. 服务端给客户端返回SSL协议版本号、加密算法种类、随机数等信息，同时也返回服务器端的证书，即公钥证书
3. 客户端使用服务端返回的信息验证服务器的合法性，包括：
	1. 证书是否过期
	2. 发型服务器证书的CA是否可靠
	3. 返回的公钥是否能正确解开返回证书中的数字签名
	4. 服务器证书上的域名是否和服务器的实际域名相匹配
	5. 验证通过后，将继续进行通信，否则，终止通信
6. 服务端要求客户端发送客户端的证书，客户端会将自己的证书发送至服务端
7. 验证客户端的证书，通过验证后，会获得客户端的公钥
8. 客户端向服务端发送自己所能支持的对称加密方案，供服务器端进行选择
9. 服务器端在客户端提供的加密方案中选择加密程度最高的加密方式
10. 将加密方案通过使用之前获取到的公钥进行加密，返回给客户端
11. 客户端收到服务端返回的加密方案密文后，使用自己的私钥进行解密，获取具体加密方式，而后，产生该加密方式的随机码，用作加密过程中的密钥，使用之前从服务端证书中获取到的公钥进行加密后，发送给服务端
12. 服务端收到客户端发送的消息后，使用自己的私钥进行解密，获取对称加密的密钥，在接下来的会话中，服务器和客户端将会使用该密码进行对称加密，保证通信过程中信息的安全。

除了双向认证，还有单向认证，具体可参考文末连接。

## 准备工作 ##
1. 项目部署结构：
	- 客户端：Android App
	- 服务端：Ngnix（反向代理） + 后台服务
2. 准备相关证书
	- CA机构证书（ca.crt）
	- 服务器证书（server.crt）
	- 服务器私钥文件（server.key）
	- 客户端证书（p12格式）（client.p12）

## Nginx配置SSL证书 ##

```
...
http {
    ...
    server {
        listen       8000 ssl;
        server_name  10.2.12.72;
		#服务器证书文件
		ssl_certificate     cert/server.crt;
		#私钥文件
		ssl_certificate_key cert/server.key;
		#CA机构证书文件
		ssl_client_certificate   cert/ca.crt;
		#开启客户端证书校验
		ssl_verify_client on;
        ...
    }
}

```

## Android端SSL认证 ##
一般客户端验证SSL有两种方式：
1. 通过SSLSocketFactory方式创建，需要设置域名及端口号(适应于HttpClient请求方式)。
2. 通过SSLContext方式创建(适用于HttpsURLConnection请求方式).

本文介绍的是使用第二种SSLContext方式。

最初，使用网络上客户端证书（client.bks）及客户端证书库(truststore.bks)方式创建SSLContext，一直报“Trust anchor for certification path not found.”的错，后来根据Android官网代码及客户端使用p12格式证书解决了问题。 具体可参考文章后边“遇到的坑”部分。

详细代码为：
```java
package com.zhangyida;

import java.io.InputStream;
import java.net.CookieManager;
import java.net.CookiePolicy;
import java.security.KeyStore;
import java.security.cert.Certificate;
import java.security.cert.CertificateFactory;

import javax.net.ssl.HostnameVerifier;
import javax.net.ssl.KeyManagerFactory;
import javax.net.ssl.SSLContext;
import javax.net.ssl.SSLSession;
import javax.net.ssl.TrustManagerFactory;

import com.squareup.okhttp.OkHttpClient;

import android.content.Context;
import android.util.Log;

public class HttpUtil {
	public static final String SERVER_PROTOCAL = "https";
	public static final String SERVER_HOST = "10.2.8.11";
	public static final String SERVER_PORT = "8000";
	
	private static final String KEY_STORE_TYPE_P12 = "PKCS12";//证书类型 固定值
    private static final String KEY_STORE_CLIENT_PATH = "client.p12";//客户端要给服务器端认证的证书
    private static final String KEY_STORE_SERVER_PATH = "server.crt";//客户端验证服务器端的证书库
    private static final String KEY_STORE_PASSWORD = "123456";// 客户端证书密码
    
    private static OkHttpClient okHttpClient;
    
	/**
     * 获取SSLContext
     *
     * @param context 上下文
     * @return SSLContext
     */
    public static SSLContext getSSLContext(Context context) {
        try {
        	//参考 https://developer.android.com/training/articles/security-ssl.html
        	CertificateFactory  certificateFactory = CertificateFactory.getInstance("X.509");
        	//这里导入服务端SSL证书文件
        	InputStream inputStream = context.getAssets().open(KEY_STORE_SERVER_PATH);  
              
        	Certificate  cer = certificateFactory.generateCertificate(inputStream);  
  
            //创建一个证书库，并将证书导入证书库  
            KeyStore trustStore = KeyStore.getInstance(KeyStore.getDefaultType());  
            trustStore.load(null,null);
            trustStore.setCertificateEntry("trust", cer);  
        	
        	
            // 服务器端需要验证的客户端证书
            KeyStore keyStore = KeyStore.getInstance(KEY_STORE_TYPE_P12);
            InputStream ksIn = context.getResources().getAssets().open(KEY_STORE_CLIENT_PATH);
            try {
                keyStore.load(ksIn, KEY_STORE_PASSWORD.toCharArray());
            } catch (Exception e) {
                Log.e("Exception", e.getMessage(), e);
            } finally {
                try {
                    ksIn.close();
                } catch (Exception ignore) {
                }
                try {
                	inputStream.close();
                } catch (Exception ignore) {
                }
            }
            SSLContext sslContext = SSLContext.getInstance("TLS");
            TrustManagerFactory trustManagerFactory = TrustManagerFactory.getInstance(TrustManagerFactory.getDefaultAlgorithm());
            trustManagerFactory.init(trustStore);
            
            KeyManagerFactory keyManagerFactory = KeyManagerFactory.getInstance("X509");
            keyManagerFactory.init(keyStore, KEY_STORE_PASSWORD.toCharArray());
            sslContext.init(keyManagerFactory.getKeyManagers(), trustManagerFactory.getTrustManagers(), null);
            return sslContext;
        } catch (Exception e) {
            Log.e("tag", e.getMessage(), e);
        }
        return null;
    }
    
    /**
     * 获取SSL认证需要的HttpClient
     *
     * @param context 上下文
     * @return OkHttpClient
     */
    public static OkHttpClient getSSLContextHttp(Context context) {
		if (okHttpClient == null) {
			synchronized (HttpUtil.class) {
				if (okHttpClient == null) {
					okHttpClient = new OkHttpClient();
					SSLContext sslContext = getSSLContext(context);
					if (sslContext != null) {
						okHttpClient.setSslSocketFactory(sslContext.getSocketFactory());
					}
					//设置cookie处理器
					okHttpClient.setCookieHandler(new CookieManager(null, CookiePolicy.ACCEPT_ORIGINAL_SERVER));
					//设置服务器HostName校验
					okHttpClient.setHostnameVerifier(new HostnameVerifier() {

						@Override
						public boolean verify(String host, SSLSession paramSSLSession) {
							if (SERVER_HOST.equals(host)) {
								return true;
							}
							return false;
						}
					});
				}
			}
		}
		return okHttpClient;
    }
}
```
代码地址：https://github.com/zhangyihao/AndroidSSL/blob/master/AndroidSSL/src/com/zhangyida/HttpUtil.java

## 遇到的坑 ##
### Android下证书问题 ###
Java平台默认识别jks格式的证书文件，但是android平台只识别bks格式的证书文件。
可以使用[Portecle](https://sourceforge.net/projects/portecle/files/)将客户端证书转换为bks格式（下载Portecle，解压后，使用命令jave -jar bcprov.jar即可打开GUI界面）。
我这边是使用客户端crt文件先转换为jks格式，在转换为bks格式，但是根据网络上代码，一直报错，报错信息为“Trust anchor for certification path not found”，解决方法见下面。

### Trust anchor for certification path not found错误 ###
具体报错信息为：
```
javax.net.ssl.SSLHandshakeException: java.security.cert.CertPathValidatorException: Trust anchor for certification path not found.
        at org.apache.harmony.xnet.provider.jsse.OpenSSLSocketImpl.startHandshake(OpenSSLSocketImpl.java:374)
        at libcore.net.http.HttpConnection.setupSecureSocket(HttpConnection.java:209)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.makeSslConnection(HttpsURLConnectionImpl.java:478)
        at libcore.net.http.HttpsURLConnectionImpl$HttpsEngine.connect(HttpsURLConnectionImpl.java:433)
        at libcore.net.http.HttpEngine.sendSocketRequest(HttpEngine.java:290)
        at libcore.net.http.HttpEngine.sendRequest(HttpEngine.java:240)
        at libcore.net.http.HttpURLConnectionImpl.getResponse(HttpURLConnectionImpl.java:282)
        at libcore.net.http.HttpURLConnectionImpl.getInputStream(HttpURLConnectionImpl.java:177)
        at libcore.net.http.HttpsURLConnectionImpl.getInputStream(HttpsURLConnectionImpl.java:271)
```
根据Android官网解释，出现此情况的原因主要有：
1. 办法服务器证书的CA未知。
2. 服务器证书不是由CA签署的，而是自签署。
3. 服务器缺少中间CA。
由于我使用证书是自己做的CA证书，属于CA未知问题，使用官网解决方案即可。
附官网解释地址：https://developer.android.com/training/articles/security-ssl.html

### 使用官网示例代码依然报错问题 ###
根据官网解释，使用官网示例后，依然报“Trust anchor for certification path not found”的错。 后来在网络上发现，有人在创建信任证书库时，和官网代码有一处不同，代码为：
```
//官网示例代码：
keyStore.setCertificateEntry("ca", ca);

//修改后代码：
trustStore.setCertificateEntry("trust", cer);
```


## 参考文章 ##
1. [Https单向认证和双向认证](http://blog.csdn.net/duanbokan/article/details/50847612 "Https单向认证和双向认证")
2. [Aandroid中https请求的单向认证和双向认证](http://blog.csdn.net/u011394071/article/details/52880062)
3. [Android HTTPS SSL双向验证(CA根证书)](http://frank-zhu.github.io/android/2017/03/30/android-https-ssl-part-02/)