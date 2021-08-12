---
layout: post
title:  "Janus.js 使用方法"
description: 介绍使用Janus.js 的方法，包括建立连接，绑定插件，发送流等
date:   2021-07-10 23:10:05 +0800
categories: javascript
tags: janus webrtc
---
# Janus.js 使用方法
> Janus是webrtc服务器应用，可以通过添加插件的方式，实现在线会议等使用webrtc技术可实现的功能
>
> Janus官网：https://janus.conf.meetecho.com/
>
> Janus.js是web端使用Janus功能的库，下面按照使用步骤介绍该库



## Janus.init-初始化Janus



在使用Janus前，需要对Janus进行初始化，初始化方式是Janus.init(options)

options参数需要是一个对象，该对象可包含有用的属性如下：

```javascript
options = {
    callback,//function 初始化成功后会回调它
    debug,//boolean 设定是否打印janus内部的log
    dependencies = {//object 初始化一些janus需要用到的方法，可以在浏览器不支持webrtc，promise等时通过添加补丁库解决.不是必要的，janus内部会在使用者没有传任何内容时使用默认的配置
    	newWebSocket,// function 建立wbesocket时使用的方法
		isArray,//function 判断变量是否是数组
		extension,//object 在浏览器不支持屏幕共享时，可以安装janus插件实现，现在不需要了
		webRTCAdapter,//object 使用webrtc需要用到的工具库，解决webrtc在不同浏览器上的差异，webrtc-adapter这个库就可以使用
		httpAPICall，//function 如果使用http服务器，janus会采用这个方法进行长轮询
	},
}
```



## Janus-实例化Janus



使用new Janus(gatewayCallbacks)就可以实例化一个janus，当实例化时，janus会和服务器进行连接

这时的连接只代表连接了janus信令服务，用于后续建立webrtc连接时交换信息的通道

janus可以使用websocket和http两种方式进行通信，以下介绍以websocket方式为主

```javascript
gatewayCallbacks = {
    success, //function 连接成功后的回调
    error,//function 连接失败后的回调
    destroyed,//function 连接关闭后的回调
    servers,//array | string janus服务器地址
    iceServers,//array 获取candidate信息的ice服务器地址，ice服务器就是获取设备内网穿透信息的地址
    ...
}
```



实例化过程中，Janus构造函数中有一些属性和方法不会添加到实例化的对象上，而是通过闭包的方式由实例化对象上的方法使用，介绍其中的一部分

```
transactions,//object 用于记录每次请求成功后需要执行的回调
```



建立连接是使用了createSession方法，createSession(gatewayCallbacks)这里的gatewayCallbacks和实例化janus时使用的是同一个

### createSession

1. 生成建立请求后第一次发送的消息，消息内容是

```json
{"janus": "create", "transaction"： transaction}//transaction是随机生成的12长度的字符串，用于标记每次的请求，每次请求都不一样，服务端在响应请求后会携带同样的信息，这样就能知道每次请求的结果，并根据结果做相应的处理
```

2. 使用newWebSocket方法建立websocket连接，newWebSocket(server, 'janus-protocol')
3. 给websocket对象注册事件监听，包括error，open，message，close

和服务器通信时，消息的格式如下，收发都符合这个格式

```json
{"janus": 消息名,"transaction"： 标记请求的id, ...消息内容 }
```



### handleEvent



服务器端通过websocket发来的消息，都通过这个方法处理，下面对个别消息做个介绍



```
trickle,//candidate 信息
webrtcup,//webrtc连接建立成功
hangup,//janus连接断开
detached,//插件移除
media,//发送音视频流的客户端会收到的消息，audio和video各一条消息，用于说明流是否发送成功
event,//sdp信息
```



### janus-实例化后的结果



实例化后的janus对象上有一些方法，下面介绍其中的一些

```javascript
janus = {
	getServer,//function 当前连接的janus服务器
	isConnected,//function 当前janus是否连接中
	reconnect,//function 重连服务器
	getSessionId,//function 获取本次sessionId
	destroy,//function 断开本次连接
	attach,//function 添加插件，实际调用的是createHandle这个方法，需要重点了解，在建立webrtc连接前需要使用该方法
	...
}
```



## janus.attach-添加插件-得到管理webrtc连接的handler对象



实际使用的是Janus.js中的createHandle方法

使用方法janus.attach(callbacks)



### createHandle



```javascript
callbacks = {
    plugin,//string plugin的名字
	opaqueId,//string 识别插件的id
	//以下都是function类型
	success,//添加插件成功时调用，调用时参数是attch方法创建的handler对象
	error,//webrtc连接出错调用
	consentDialog,//一些操作完成后，无论是否成功都会调用
	iceState,//icestate发生变化时调用
	mediaState,//流发送方在webrtc连接建立后会收到media消息，收到该消息时调用
	webrtcState,//webrtc state发生变化后调用
	slowLink,//webrtc通信速度慢时调用
	onmessage,//收到janus服务器发来的event消息时调用，服务器会发来sdp信息，需要在这个回调里将sdp保存到本地
	onlocalstream,//本地的流可以使用时调用
	onremotestream,//远端的流可以使用时调用
	ondata,//webrtc datachannel里的消息会调用
	ondataopen,//webrtc datachannel打开时调用
	oncleanup,//webrtc 连接关闭时调用
	ondetached,//plugin 移除时调用
	...
}
```



向服务器端发送attach消息，在收到success的消息后会创建一个handler对象，并将该对象通过callbacks.success发送给业务



```javascript
pluginHandler = {
	session,//object 实例化的janus对象
    plugin,//string 插件名
    id,//string 插件id
    detached,//boolean 插件是否移除
    webrtcStuff,//object 保存webrtc相关信息的对象，包括RTCPeerConnection实例，sdp，stream等
    //以下都是function类型
    send,//通过janus发送消息
    data,//通过webrtc datachannel发送消息
    createOffer,//创建webrtc连接的方法，需要详细了解，下面会进行介绍
    handleRemoteJsep,//在收到远端的sdp信息后需要调用
    hangup,//关闭webrtc连接
    detach//移除插件
    ...
}
```



## handler.createOffer-建立webrtc连接-发送sdp和candidate



实际使用的是Janus中的prepareWebrtc方法

createOffer(callbacks),这里的callbacks是一个新的对象，和之前的没有联系



### prepareWebrtc



```javascript
callbacks = {
	success,//function 在peerconnection生成本地offer成功后调用，需要将生成的offer的信息发送到服务器端（业务中如果需要一些信息标识用户所在的房间，可以在这个方法里要将本次连接的jsep和房间信息发送到janus服务器）
	error,//function
	media,//objct 设置webrtc建立后流接收发送的类型
	stream,//object MediaStream实例，发送端需要制定发送的流
	jsep,//object 使用指定的sdp信息建立连接
	customizeSdp,// function 在生成offer后会调用该方法，用于修改默认的sdp信息
}
```



media 对象举例(发送音视频，并接收音视频, 并开启webrtc的datachannel)



```javascript
media = {
	audioRecv： true,
	videoRecv: true,
	audioSend: true,
	videoSend: true,
    data: true,
}
```



在该方法中会调用stramsDone建立webrtc连接



### streamsDone



1. 创建RTCPeerConnection实例pc，并保存在handler的webrtcStuff对象上
2. 在pc上注册事件监听，监听的事件，及其回调如下

```
onicecandidate,//将pc收到的candidate通过janus发送到服务器端
ontrack,//收到webrtc中的流调用handler中onremotestream
ondatachannel,//使用createDataChannel创建data channel，不做介绍
onicesconnectionstatechange,//当ice state发生变化时调用handler中的iceState
```



如果没有指定sdp信息，也就是没有jsep对象，则会使用createOffer方法生成sdp信息



### createOffer



1. 使用peerConnection上的createOffer创建offer
2. 成功创建后将sdp存在handler的webrtcStuff对象上
3. 使用peerConnection上的setLocalDescription方法设置本地的sdp信息



## handler.handleRemoteJsep-建立webrtc连接-设置远端的sdp



在收到服务端的event消息时，如果消息里有内容，也就是onmessage回调执行时有第二个参数时，就是收到了服务器端的jsep信息，需要使用handleRemoteJsep将这个信息保存到客户端

handleRemoteJsep实际执行的是prepareWebrtcPeer



### prepareWebrtcPeer



1. 使用peerConnection上的setRemoteDescription将服务器端的jsep信息保存
2. 使用peerConnection上的addIceCandidate将服务器端的candiata信息保存
3. 完成建立连接



## Webrtc连接建立过程再说明



webrtc建立需要两端交换sdp和candidata信息，客户端发送这2个信息和接收这2个信息都是通过janus的，其中有些操作必须在回调里执行，否则webrtc连接无法建立成功，下面对建立连接的过程再做介绍



1. streamsDone 创建peerConnection，注册oncandidate回调，将客户端的candidate发送到服务器端
2. janus在收到trickle消息时，消息内容里有candidate信息，完成1和2就完成了candidate交换
3. handler.createOffer 使用peerConnection创建offer，使用setLocalDescription设置本地的sdp信息
4. handler.createOffer 传入的success回调里将本地offer发送到服务器端
5. janus.attach 传入的onmessage里将服务器端的jsep使用handler.handleRemoteJsep方法，将服务器端的sdp保存到本地
6. 在执行第5步中的handleRemoteJsep方法时，会将服务器端的candiate添加到peerConnection



完成以上步骤webrtc连接建立成功



