# H5 与 Native 交换图片技术方案

## 背景

目前在APP内的H5分享图片流程如下：

> H5 -> canvas 绘制界面获取要分享的图片 -> 上传图片到 CDN -> 调用 JSSDK 的分享 API，入参是图片的 CDN 链接 -> App 收到分享请求后，会根据图片的 CDN 链接下载图片 -> 然后调用第三方 SDK 分享图片。

从这个过程中，实际上 H5 和 App 一共会进行两次的 CDN 操作，这样的操作是非常耗时，网络差的时候，在 H5 内的分享图片操作会非常卡，体验很差。


## 方案选型
#### Base64 方案：
此方案是把图片二进制的一个字节转成一个字符，最后会生成一个纯字符串，通过 JSBridge 把改字符串传给 Native，但是如果图片过大，产生的字符串很长，会导致 H5 和 Natvie 传输很慢，会让页面有卡死的现象。

#### Local Web Server 方案：
逆向了微信，看了下微信里 log 日志，会发现 
> [INFO] WCProxyServer started on port 48765 and reachable at http: //localhost:48765/
    2019 - 05 - 30 22 : 17 : 52.821043 + 0800 WeChat[625 : 18089] evaluateJavaScript: if (window.WeixinJSBridge) {
        WeixinJSBridge._handleMessageFromWeixin({
            "__sha_key": "e917c46a4fec7f116c7a3a9b325ab946e94bb15b",
            "__json_message": "{\"__params\":{\"err_msg\":\"imageProxyInit:ok\",\"serverUrl\":\"http://localhost:48765/\",\"port\":48765},\"__callback_id\":\"1018\",\"__msg_type\":\"callback\"}"
        });
    }
    
    
这段日志，说明微信在本地也是开启了一个本地web服务，来进行 H5 和 Native 的图片交换。

所以我们也决定使用 Local Web Server。

## Local Web Server

### 介绍
相当于在 App 内部启动了一个 Web 服务器，这样我们就可以绕过 JSBridge，让 H5 和本地 Web 服务器进行通信。


所以对于此方案可以简单的想象是把图片上传到 CDN 和从 CDN 下载的过程，替换成把图片上传到本地 Web 服务器和从本地 Web 服务器下载图片。

### 技术选型
#### iOS Local Web Server
[CocoaHTTPServer](https://github.com/robbiehanson/CocoaHTTPServer)

#### Android Local Web Server（仅供 Android 参考，要选能支持 Https 的服务器）

[AndroidAsync](https://github.com/koush/AndroidAsync)

[android-http-server](https://github.com/piotrpolak/android-http-server)

[nanohttpd](https://github.com/NanoHttpd/nanohttpd)

### 流程图
![image](https://github.com/karosLi/H5NativeImageExchange/blob/master/H5%E4%B8%8ENative%E5%9B%BE%E7%89%87%E4%BA%A4%E6%8D%A2%E6%97%B6%E5%BA%8F%E5%9B%BE.jpg?raw=true)

### 遇到的问题
#### 跨域
由于大部分的链接都是支持 Https，如果本地服务器是 Http 会产生 跨域问题。那怎么让本地服务器支持 Https 呢，此时就需要使用自签名证书和代码上需要集成支持自签名证书。

#### 自签名证书会过期
自签名会有过期的问题，怎么好动态更新证书。我们可以在创建自签名证书时设置有效期为 10 年，那这样就可以不用考虑动态更新了。

#### 自签名证书信任问题
通过 NSURLProtocol/WebViewClient 拦截请求，在校验证书合法性时，始终信任自签名证书可以解决这个问题。

#### 计算上传文件 MD5
可以使用 spark-md5 在 H5 侧计算上传图片的 MD5 值。

#### 本地图片删除策略
什么时候去清掉不用的图片呢，不然本地目录会越来越大的。通过设定最大缓存大小是 100M，最大保留时间是 7天来解决此问题，触发的时机可以在 App 启动 60s 后在子线程来做一次清理。

#### 动态端口
由于本地服务器的端口只有 Native 知道，而 H5 是不知道的，那怎么把端口号告诉 H5 呢，如果通过 JSBridge 告诉 H5 就比较繁琐了。有没有可能直接把 https://resource/uploadFile 映射到 https://localhost:60000/uploadFile 呢？

是有可能的，可以通过 NSURLProtocol/WebViewClient 针对 AJAX POST 请求进行拦截，并把 https://resource/uploadFile 重定向到 https://localhost:60000/uploadFile 上，这样就可以完美的 屏蔽动态端口对 H5 的影响。

#### App 怎么更好的访问本地图片
App 是否支持这样的链接呢？
https://resource/76a9ca44b0af8b4d5fa238e5f19f5a4f

也是可以的，同样是通过 NSURLProtocol/WebViewClient 针对 Get 请求进行拦截，把 https://resource/76a9ca44b0af8b4d5fa238e5f19f5a4f 重定向到 https://localhost:60000/76a9ca44b0af8b4d5fa238e5f19f5a4f 上即可。

#### 本地服务会占用资源
这种方式都是比较成熟的，可以使用轻量级的 Web Server 框架。

另外可以监听前台和后台通知，在后台时停止服务，前台时恢复服务，这样可以避免占用资源导致在后台时被系统杀掉。


## 未来考虑
这是一个基础的能力，未来还可以有其他想象，比如选择照片时可以返回刚给 H5 本地图片 URL；比如预览图片时，H5 也可以把本地图片 URL 交给 Native 去完成直接预览；比如有场景的话，还可以给 H5 提供上传本地图片到远程 CDN 的能力。
