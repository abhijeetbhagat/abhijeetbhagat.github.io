---
layout: post
title:  "Setting up a test rtsp stream"
---

steps for setting up a test rtsp source:
&nbsp;
1. download `rtsp-simple-server` from [here](https://github.com/aler9/rtsp-simple-server and extract the archive)
2. download `rtsp-simple-server.yml` file from [here](https://raw.githubusercontent.com/aler9/rtsp-simple-server/main/rtsp-simple-server.yml)
3. either set `protocols` value to `udp` or `tcp` in the yml file
4. turn off rtmp, hls in the yml file
5. create an mp4 file to stream from:
&nbsp;
```
$ ffmpeg -f lavfi -i testsrc -t 30 -pix_fmt yuv420p test.mp4
```
6. run:
&nbsp;
```
$ ./rtsp-simple-server rtsp-simple-server.yml 
$ ffmpeg -re -stream_loop -1 -i test.mp4 -f \
  rtsp -rtsp_transport tcp rtsp://localhost:8554/live_stream
```
7. play `rtsp://localhost:8554/live_stream` in vlc
