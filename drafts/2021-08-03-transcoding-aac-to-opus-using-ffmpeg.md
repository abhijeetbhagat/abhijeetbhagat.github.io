---
layout: post
title:  "how i made a camera audio stream workable for webrtc or how to transcode aac to opus"
---
i've continued hacking on janus (my own fork) to implement a webrtc broadcasting solution for cameras that stream over rtsp. but here's the thing: webrtc mandates support for a small subset of audio and video codecs. the minimum supported audio codecs are opus, g7.11. g7.11 (both pcma and pcmu) is a narrowband (300-3400 Hz) codec while opus supports all spectrums - narrowband to fullband. this makes opus ideal for optimized network transport of audio streams. opus is actually two codecs combined together - SILK and CELT. SILK is good for music while CELT is good for speech making opus the right choice for webrtc.

however, a surveillance cam often has aac (advanced audio codec) for audio enabled and it can't be played in a browser. therefore, the only option is to transcode it to a webrtc compliant audio codec. so i implemented aac to opus transcoding in janus so that webrtc viewers can listen to the camera audio. i had absolutely no idea about audio prior to this. i had no idea how audio was packetized into rtp packets. i didn't even know how audio was represented.

there are several frameworks/libraries like gstreamer, ffmpeg's libav, etc. that can do this transcoding for us. i had built a small video player /*insert spe link*/ using ffmpeg and rust by porting an old ffmpeg c tutorial some time ago. so riding on that confidence, i picked ffmpeg. and also, should the need arise, i can reimplement the transcoding in rust using high level libav api. 

there are two things needed to get the aac audio played in a browser in the webrtc context:
1. needless to say, transcoding
2. sdp negotiation between the webrtc peers (browser and janus)
transcoding takes the bulk of the efforts while fixing sdp negotiation is trivial.

transcoding
 *installing libav dev libs*
 lets install libavcodec et. al. i am using an ubuntu 20 lts system.
    `sudo add-apt-repository ppa:savoury1/ffmpeg4`
   and then:
    ```
    sudo apt install ffmpeg \
        libavcodec-dev \
        libavdevice-dev \
        libavfilter-dev \
        libavformat-dev \
        libavresample-dev \
        libavutil-dev \
        libpostproc-dev \
        libswresample-dev \
        libswscale-dev \
    ```
    this will install the headers and shared libs needed in our implementation.

 *rtp packet parsing*
 as explained earlier /*insert aac packet parsing link*/, an aac rtp packet may have a single frame, multiple frames, a partial frame or a sensible combination of the three. thankfully, ffmpeg already has an aac packet parser.
 *aac decoding*
 once we have an aac packet, it needs to be decoded into frames.
 *resampling*
 an aac sample is represented by a float (32 bits). when using the float representation, planar format should be used. /*planar vs interleaved*/
 whereas, an opus sample is represented either using signed 16 bits or float. with interleaved format. therefore, simply trying to encode the decoded aac frames does not work. the aac samples need to be converted to opus-encodeable samples and this is what is called as resampling.
 *opus encoding*
 now that we have decoded and resampled aac frames, we can encode them using an opus encoder.
 *rtp packetizing*
 after we have an opus packet representing compressed audio data, we need to packetize it for rtp transport. fortunately, packetizing opus data isn't complicated like aac packetization.

sdp negotiation
 *remove the aac fmtp line*
 *modify the rtpmap line to reflect opus params*
