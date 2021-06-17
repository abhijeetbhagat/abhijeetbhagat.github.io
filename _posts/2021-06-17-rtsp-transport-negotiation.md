---
layout: post
title:  "RTSP transport negotiation between a client and a server"
---

when a client connects to an rtsp server to request a media stream, it goes through the usual execution of rtsp commands  - options, describe, setup, play, etc. streaming over rtsp/rtp can be done using both udp or tcp. but how does a client pick either udp or tcp as a transport protocol? here's how i found by observing traffic in wireshark.
&nbsp;
for this, we setup an rtsp server with tcp as a transport. when a client (vlc in our case) sends a `setup` to an rtsp server, it chooses udp by default (from wireshark): 
&nbsp;
```
    Request: SETUP rtsp://orion:8554/live_stream/trackID=0 RTSP/1.0\r\n
    CSeq: 4\r\n
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)\r\n
    Transport: RTP/AVP;unicast;client_port=65406-65407
    \r\n
```
&nbsp;
**`RTP/AVP`** really means **`RTP/AVP/UDP`**.
&nbsp;
the server responds with:
&nbsp;
```
    Response: RTSP/1.0 461 Unsupported Transport\r\n
    CSeq: 4\r\n
    Server: gortsplib\r\n
    \r\n
```
&nbsp;
so the client again sends a `setup` request with tcp as the transport protocol:
&nbsp;
```
    Request: SETUP rtsp://orion:8554/live_stream/trackID=0 RTSP/1.0\r\n
    CSeq: 5\r\n
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)\r\n
    Transport: RTP/AVP/TCP;unicast;interleaved=0-1
    \r\n
```
&nbsp;
the difference now is **`RTP/AVP/TCP`** and **`interleaved`** (all data will be interleaved on a single port). but there's no **`client_port`** sent like in the first `setup` request. the entire data, thus, is sent over the same tcp port used to send rtsp commands. this is the packet used to send the second `setup` request (observe the src port):
&nbsp;
```
Transmission Control Protocol, Src Port: 3220, Dst Port: 8554, Seq: 456, Ack: 619, Len: 179
    Source Port: 3220
    Destination Port: 8554
    ...
Real Time Streaming Protocol
    Request: SETUP rtsp://orion:8554/live_stream/trackID=0 RTSP/1.0\r\n
    CSeq: 5\r\n
    User-Agent: LibVLC/3.0.8 (LIVE555 Streaming Media v2016.11.28)\r\n
    Transport: RTP/AVP/TCP;unicast;interleaved=0-1
    \r\n
```
&nbsp;
and this is an rtp packet sent by the server (observe the dst port):
&nbsp;
```
Transmission Control Protocol, Src Port: 8554, Dst Port: 3220, Seq: 898, Ack: 795, Len: 460
    Source Port: 8554
    Destination Port: 3220
    ...
RTSP Interleaved Frame, Channel: 0x00, 456 bytes
    Magic: 0x24
    Channel: 0x00
    Length: 456
Real-Time Transport Protocol
    10.. .... = Version: RFC 1889 Version (2)
    ..0. .... = Padding: False
    ...0 .... = Extension: False
    .... 0000 = Contributing source identifiers count: 0
    1... .... = Marker: True
    Payload type: DynamicRTP-Type-96 (96)
    Sequence number: 8221
    Timestamp: 2313639317
    Synchronization Source identifier: 0xdf64ccfa (3747925242)
    Payload: 000001b65c6053ffffffffffffffef00008f0afffffffffffffff700009e0affffffffff
```
