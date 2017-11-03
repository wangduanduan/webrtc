# 1 介绍
WebRTC 是一个开源项目，用于Web浏览器之间进行实时音频视频通讯，数据传递。
WebRTC有几个JavaScript APIS。 点击链接去查看demo。

- [getUserMedia(): 捕获音频视频]()
- [MediaRecorder: 记录音频视频]()
- [RTCPeerConnection: 在用户之间传递音频流和视频流]()
- [RTCDataChannel: 在用户之间传递文件流]()

## 1.1 在哪里使用WebRTC?
- Chrome
- FireFox
- Opera
- Android
- iOS

## 1.2 什么是信令
WebRTC使用`RTCPeerConnection`在浏览器之间传递流数据, 但是也需要一种机制去协调收发控制信息，这就是信令。信令的方法和协议并不是在WebRTC中明文规定的。 在codelad中用的是Node，也有许多其他的方法。

## 1.3 什么是STUN和TURN?
> STUN（Session Traversal Utilities for NAT，NAT会话穿越应用程序）是一种网络协议，它允许位于NAT（或多重NAT）后的客户端找出自己的公网地址，查出自己位于哪种类型的NAT之后以及NAT为某一个本地端口所绑定的Internet端端口。这些信息被用来在两个同时处于NAT路由器之后的主机之间创建UDP通信。该协议由RFC 5389定义。 [wikipedia STUN](https://zh.wikipedia.org/wiki/STUN)

> TURN（全名Traversal Using Relay NAT, NAT中继穿透），是一种资料传输协议（data-transfer protocol）。允许在TCP或UDP的连线上跨越NAT或防火墙。
TURN是一个client-server协议。TURN的NAT穿透方法与STUN类似，都是通过取得应用层中的公有地址达到NAT穿透。但实现TURN client的终端必须在通讯开始前与TURN server进行交互，并要求TURN server产生"relay port"，也就是relayed-transport-address。这时TURN server会建立peer，即远端端点（remote endpoints），开始进行中继（relay）的动作，TURN client利用relay port将资料传送至peer，再由peer转传到另一方的TURN client。[wikipedia TURN](https://zh.wikipedia.org/wiki/TURN)

> ICE （Interactive Connectivity Establishment，互动式连接建立 ），一种综合性的NAT穿越的技术。
互动式连接建立是由IETF的MMUSIC工作组开发出来的一种framework，可整合各种NAT穿透技术，如STUN、TURN（Traversal Using Relay NAT，中继NAT实现的穿透）、RSIP（Realm Specific IP，特定域IP）等。该framework可以让SIP的客户端利用各种NAT穿透方式打穿远程的防火墙。[wikipedia ICE](https://zh.wikipedia.org/wiki/%E4%BA%92%E5%8B%95%E5%BC%8F%E9%80%A3%E6%8E%A5%E5%BB%BA%E7%AB%8B)


WebRTC被设计用于点对点之间工作，因此用户可以通过最直接的途径连接。然而，WebRTC的构建是为了应付现实中的网络: `客户端应用程序需要穿越NAT网关和防火墙，并且对等网络需要在直接连接失败的情况下进行回调。` 作为这个过程的一部分，WebRTC api使用STUN服务器来获取计算机的IP地址，并将服务器作为中继服务器运行，以防止对等通信失败。(现实世界中的WebRTC更详细地解释了这一点。)

## 1.4 WebRTC是否安全?
WebRTC组件是强制要求加密的，并且它的JavaScript APIS只能在安全的域下使用(HTTPS 或者 localhost)。信令机制并没有被WebRTC标准定义，所以是否使用安全的协议就取决于你自己了。


# 2 概述
通过你的摄像头，构建一个app用来获取视频并截图，并且通过WebRTC端到端分享。过程中你会学习如何使用WebRTC核心的APIs，并且用Node建立一个消息服务器。


# 2.1 你会学到什么？
- 从你的摄像头获取视频
- 用RTCPeerConnection传递视频流
- 用RTCDataChannel传递数据流
- 发送信令去交换信息
- 组合端链接和信令
- 拍照并通过datachannel去分享

# 2.2 你要准备什么？
- Chrome 47以上
- 简易的WebServer
- 示例代码
- 文本编辑器
- 基本的HTML, CSS , JavaScript知识

# 3 从摄像头中获取视频流
## 3.1 完整代码
[查看在线demo](./demos/stream-video-from-your-webcam.html)

```
<!DOCTYPE html>
<html>

<head>
    <meta charset="utf-8">
</head>

<body>

    <!-- autoplay字段很重要，你可以不加这个字段试试，你会发现这个video图像是不会动的，因为它只是一帧 -->
    <video autoplay id="wdd"></video>

    <script type="text/javascript">
    var video = document.getElementById('wdd');

    // 检测浏览器是getUserMedia的方法名，不同浏览器可能有不同的前缀
    navigator.getUserMedia = navigator.getUserMedia ||
        navigator.webkitGetUserMedia ||
        navigator.mozGetUserMedia ||
        navigator.msGetUserMedia;

    // 判断浏览器是否支持WebRTC
    if (navigator.getUserMedia) {
        initVideo();
    } else {
        alert('你的浏览器不支持WebRTC');
    }

    function initVideo(){

        navigator.getUserMedia({
            audio: false,
            video: true
        }, function(stream){
            // 将流对象暴露在全局环境，方便打印
            window.stream = stream;
            video.srcObject = stream;
        }, function(error){
            console.error(error);
        });
    }
    </script>
</body>

</html>
```

## 3.2 工作原理

获取视频流的关键是`getUserMedia()`方法，一般在使用这个方法时，浏览器会请求用户给与权限。如果用户拒绝，那么视频流是无法获取的。

```
navigator.getUserMedia(constraints, successCallback, errorCallback);
```
getUserMedia有三个必须的参数

参数 | 类型 | 是否必须 |  默认值 | 描述
---|---|---|---|---
constraints | 对象 | 是 | 无 | 获取流的限制参数。常用的有`{audio: false,video: true}`, 表示要获取那种流
successCallback | 函数 | 是 | 无 | 成功的回调，有stream作为参数
errorCallback | 函数 | 是 | 无 | 失败的回调，有error作为参数

`示例`
```
navigator.getUserMedia({
    audio: false,
    video: true
}, function(stream){
    console.log(stream);
}, function(error){
    console.error(error);
});
```


## 3.3 注意点
- audio务必加上`autoplay`字段，否则只能看到一帧的画面

# 3.4 延伸
constraints的参数有很多，并不仅仅只有audio和video。
还可以传递下面这种对象，用来设置长宽和帧率。
```
{
    "audio": true,
    "video": {
        "width": {
            "min": "734",
            "max": "1189"
        },
        "height": {
            "min": "200",
            "max": "480"
        },
        "frameRate": {
            "min": "27",
            "max": "17"
        }
    }
}
```

# 使用RTCPeerConnection传递视频流
