---
title: webrtc_directory_analyze
date: 2017-12-11 17:06:12
tags:
	- WebRTC
categories: WebRTC
---

# 1. Structure
![](https://raw.githubusercontent.com/JShell07/images/master/webRTC/webrtc_call_flow.png)

<!-- more -->
# 2. Directory Analyze
Directory | Remarks
-         | :-
api       | native api, support for app or web browser. Note **medianinterface.h** and **peerconnectioninterface.h**
**pc**    | 1.hannel manager; <br>2.**peerconnecction**, **mediastream** manage; <br>3.RTP receive/send
**audio** | audio receive/send
**video** | 1.video receive/send; <br>2.video encoder;  <br>3.handle synchronize, quality
common audio | 1.audio converter; <br>2.channel buffer; <br>3.smoothing filter; <br>4.handle wav file
common video | 1.**h264**, **libyuv**; <br>2.bitrate control; <br>3.video frame, video render frame
call      | provide base class for audio and video
**media** | Audio and Video will be AddStreams as track. <br>1.webrtc general video handle; <br>2.sctp.(Session Description Transport Protocol)
stats     | reference counter
**modules** | 1.audio (coding, device, mixer, processing); <br>2.bitrate controller; <br>3.video(capture, coding)
p2p       | 1.port, session, stun(server, port); <br>2.stunprober
rtc_base  | base class. <br>1.bind, network, socket; <br>2.crc32, md5, openssl; <br>3.(bit, byte)buffer, memory; <br>4.task, thread
systerm_wrapper | 1.clock; <br>2.event; <br>3.rw_lock