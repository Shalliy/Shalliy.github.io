---
title: 基于Flutter的WebRTC学习
date: 2023-10-31 17:37
tags: Flutter,WebRTC
---

# 基于Flutter的WebRTC学习

学习WebRTC之前，先需要了解几个名词：

信令服务器：
是用来交换数据的，例如交换`SDP`等。
信令服务器的协议由多种方式，最常用的是`XHR`和`WebSocket`进行数据交换

SDP:
Session Discription Protocol(会话描述协议)
双方进行媒体协商(也就是**交换SDP**)的时候使用(例如：双方使用的音视频编码格式等等)
通过**信令服务器**进行传输

Candidate:
网络信息，通过**信令服务器**进行交换

ICE:
是一种框架，使各种NAT穿透技术可以实现统一，该技术可以让客户端成功地穿透远程用户与网络之间可能存在的各类防火墙。

Stun服务器：
STUN服务器能够知道Peer-A以及Peer-B的公网IP地址及端口

WebRTC主要流程分为三个部分：

1. 媒体协商
Peer-A和PeerB通过**信令服务器**进行媒体协商，双方交换的数据由**SDP**描述
2. 网络协商
Peer-A和PeerB通过**Stun服务器**获取到各自的网络信息(如：IP，端口等)，然后通过**信令服务器**转发
3. 建立连接
如果没有直连，那就通过**TURN中转服务器**进行转发音视频数据, 最终完成视频通话

#### WebRTC代码部分的流程

```dart
RTCVideoRenderer _localRenderer = RTCVideoRenderer(); //本地流渲染
RTCVideoRenderer _remoteRenderer = RTCVideoRenderer(); //远程流渲染
MediaStream? _localStream; //本地流
```

##### 1.获取媒体流

使用 `getUserMedia`获取媒体流，返回一个`stream`对象，可以用`RTCVideoRenderer`来进行渲染。

```dart
navigator
    .mediaDevices
    .getUserMedia(P2PConstraints.MEDIA_CONSTRAINTS)
    .then(stream) {
        _localStream = stream;
        //渲染本地流
        setState(() {
          _localRenderer.srcObject = stream;
        });
    }
```

##### 2.生成pc对象

`pc`对象其实就是是`RTCPeerConnection`对象，该类是最主要的`WebRTC`类，负责建立`WebRTC`连接，处理音视频数据流的传输

```dart
RTCPeerConnection? pc;
pc = await createPeerConnection(P2PConstraints.PC_CONSTRAINTS);
```

##### 3. 将`stream`添加到peer-A对象

将`第1步`获取到的`stream`添加到`peer-A`对象

```dart
_localStream!.getTracks().forEach((track) {
    pc!.addTrack(track, _localStream!);
});
```

##### 4.peer-A创建`offer`

创建`offer`会返回一个`RTCSessionDescription`对象，也就是`sdp`
```dart
RTCSessionDescription sdp =
          await pc!.createOffer(P2PConstraints.SDP_CONSTRAINTS);
```

##### 5.设置本地描述

设置本地描述
调用`setLocalDescription`方法后，会触发`pc`的`iceCandidate`方法

```dart
await pc!.setLocalDescription(sdp);
```

##### 6.将`peer-A`的`sdp`通过信令服务器发送给`peer-B`

```dart
//这个是自己实现的方法，
//通过websocket 或者 XHR 发送sdp到远端(peer-B)
await sendOfferSdp(sdp);
```

##### 7.收到远端`sdp`

根据信令服务器返回的信息，获取远端的`sdp`信息

```dart
RTCSessionDescription answerSdp =
          RTCSessionDescription(sdp, type);
```

##### 8.设置远端描述

至此，我们实现了交换`sdp`信息

```dart
pc!.setRemoteDescription(answerSdp);
```

##### 9.交换ice地址信息

在`第2步`中生成创建`pc`后，实现监听`onIceCandidate`，`第5步`以后会自动调用此方法，获取`candidate`信息

```dart
pc!.onIceCandidate = (candidate) {
    //发送candidate，自己实现的方法，通过信令服务器发送candidate
    sendCandidate(candidate);
};
```