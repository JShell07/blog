---
title: webRTC_Directory
date: 2017-11-28 20:03:36
tags:
	- WebRTC
	- Streaming
categories: WebRTC
---

# 1.Backgroud
WebRTC is a free, open project that enables web browsers with Real-Time Communications (RTC) capabilities via simple JavaScript APIs. The WebRTC components have been optimized to best serve this purpose.  

The WebRTC initiative is a project supported by Google, Mozilla and Opera, amongst others. This page is maintained by the Google Chrome team.

And it's source code structure like this:  
![](https://raw.githubusercontent.com/JShell07/images/master/webRTC/webRTC_Structure.png)  
More detail see <https://webrtc.org>  

<!-- more -->
# 2.Source code Directory
Dir. | Remarks
-    | :-
api | WebRTC 接口层。包括 DataChannel, MediaStream SDP相关的接口。各浏览器都是通过该接口层调用的 WebRTC。
call | 存放的是 WebRTC “呼叫（Call）” 相关逻辑层的代码。
audio | 存放音频网络逻辑层相关的代码。音频数据逻辑上的发送，接收等代码。
video | 存放视频逻辑层及视频引擎层的相关的代码。视频数据逻辑上的发送，接收等代码。视频引擎层就是指如何控制视频采集，处理和编解码操作的逻辑。
voice_engine | 存放音频引擎代码。主要是控制音频的采集，处理，编解码的操作。*这个目录后面可能也会被拿掉。*
sdk | 存放了 Android 和 IOS 层代码。如视频的采集，渲染代码都在这里。
pc | 存放一些业务逻辑层的代码。如 channel, session等。
common_audio | 放一些音频的基本算法。包括环形队列，博利叶算法，滤波器等。
common_video |存放了视频算法相关的常用工具，如libyuv, sps/pps分析器，I420缓冲器等。
modules | ** 后面单独列举 **
media | 存放媒体相关的代码。
p2p |  
rtc_base | 存放了一些基础代码。如线程，事件，socket等相关的代码。
rtc_tools | 存放了一些工具代码。如视频帧比较，I420转RGB，视频帧分析。
stats | 存放各种数据统计相关的类。
libjingle | 网络库。
system_wrapper | 与操作系统相关的代码，如 CPU特性，原子操作，读写锁，时钟等。

*** Modules: ***  

Dir. | Remarks
-    | :-
audio_coding | 音频编解码相关代码。
audio_conference_mixer | 会议混音相关代码。
audio_device | 音频采集与音频播放相关代码。
audio_mixer | 混音相关代码，这部分是后加的。
audio_processing | 音频前后处理的相关代码。
bitrate_controller | 码率控制相关代码。
congestion_controller | 流控相关的代码。
desktop_capture | 桌面采集相关的代码。
media_file | 播放媒体文件相关的代码。
pacing | 码率探测相关的代码。
remote_bitrate_estimator | 远端码率估算相关的代码。
rtp_rtcp | rtp/rtcp协议相关代码。
video_capture | 视频采集相关的代码。
video_coding | 视频编解码相关的代码。
video_processing | 视频前后处理相关的代码。

