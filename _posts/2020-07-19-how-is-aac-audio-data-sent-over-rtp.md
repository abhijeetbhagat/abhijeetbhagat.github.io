---
layout: post
title:  "how aac audio data is sent over rtp"
---
usually, surveillance cameras support video codecs like h264, h265 and audio codecs like pcmu, pcma and aac. this post briefly describes how the aac-hbr (high bit rate) audio data coming from a camera is transported over rtp.
&nbsp;
[rfc 3640](https://datatracker.ietf.org/doc/html/rfc3640) explains how mpeg4 elementary streams are transported over rtp. of particular importance to us is [section 3.3.6](https://datatracker.ietf.org/doc/html/rfc3640#section-3.3.6) which tells how to extract the aac frames from an rtp packet.

&nbsp;
this is how an rtp packet looks like:
&nbsp;
```
         +---------+-----------+-----------+---------------+
         | RTP     | AU Header | Auxiliary | Access Unit   |
         | Header  | Section   | Section   | Data Section  |
         +---------+-----------+-----------+---------------+

                   <----------RTP Packet Payload----------->

                   figure: the rtp packet
```
&nbsp;
this is how the access header section looks like:
&nbsp;
```
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- .. -+-+-+-+-+-+-+-+-+-+
      |AU-headers-length|AU-header|AU-header|      |AU-header|padding|
      |                 |   (1)   |   (2)   |      |   (n)   | bits  |
      +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+- .. -+-+-+-+-+-+-+-+-+-+

                   figure: the access unit headers section
```
&nbsp;
the padding bits make sure that the access-unit header section is octet (byte) aligned. this is so because the `AU-headers-length` is in bits. if this length is, say, 12 bits, then 4 bits padding bits will nicely align the headers section on an octet level.

&nbsp;
when an rtp packet (assuming this is an rtsp over tcp interleaved stream where every rtp packet starts with 4 marker bytes) looks like this:
&nbsp;
[0x24, 0, 0, 1, ... rtp header ..., 00, 10, 04, 00, ff, f9, ... rest of the rtp payload ...]
&nbsp;
the first two bytes - 0x00 and 0x10 - right after the rtp header, represent the access-unit headers section length in bits. so combining those two bytes gives us 0x0010 (decimal 16) which is how long the access unit headers section is. in this packet, there's just one access unit header 16 bits wide - 0x04 and 0x00. 

&nbsp;
**what does the access unit header tell us?**
&nbsp;
```
      +---------------------------------------+
      |     AU-size                           |
      +---------------------------------------+
      |     AU-Index / AU-Index-delta         |
      +---------------------------------------+
      |     CTS-flag                          |
      +---------------------------------------+
      |     CTS-delta                         |
      +---------------------------------------+
      |     DTS-flag                          |
      +---------------------------------------+
      |     DTS-delta                         |
      +---------------------------------------+
      |     RAP-flag                          |
      +---------------------------------------+
      |     Stream-state                      |
      +---------------------------------------+

      figure: an access unit header
```
&nbsp;
it tells us what the size of the aac frame is in terms of number of bytes. here's how.
&nbsp;
combining the two bytes of the access unit header gives us 0x0400. if we look at the audio fmtp line in the sdp:
&nbsp;
```
...
a=fmtp:97 streamtype=5;profile-level-id=1;mode=aac-hbr;sizelength=13;indexlength=3;indexdeltalength=3;config=1188
...
```
&nbsp;
we can see there are different parameters:
&nbsp;
```
streamtype = 5
profile-level-id = 1
mode = aac-hbr
sizelength = 13
indexlength = 3
indexdeltalength = 3
config = 1188
```
&nbsp;

the `sizelength` parameter tells us the size of the access unit (`AU-size` in the above figure), or in other words, the length of an aac frame. since `sizelength` is 13 in the fmtp line, if we take 13 bits from the access unit header - 0x0400 - we get to know that the aac frame is 128 bytes long. also, since our access-unit header is only 16 bits wide, we dont have to worry about parsing the `CTS-flag`, `CTS-delta`, `DTS-flag`, `DTS-delta`, `RAP-flag` and `Stream-state` fields since they are absent. however, if the header was wider than 16 bits, we would have to check for the presence of the following parameters in the ftmp line:
&nbsp;
`CTSDeltaLength`: number of bits indicating how wide the CTS-delta info is
`DTSDeltaLength`: number of bits indicating how wide the DTS-delta info is
`randomAccessIndication`: whether the RAP-flag is there or not
`streamStateIndication`: number of bits indicating how wide the Stream-state info is
&nbsp;
and parse those fields accordingly.

&nbsp;
**where does the aac frame start from?**
&nbsp;

since aac has an mpeg4 profile, its data is encapsulated in an access unit. the access unit starts right after the auxiliary section, which is optional. so how do we know if there's an auxiliary section in our rtp packet? the rfc mandates that there should be no auxiliary section when using hbr mode. also, looking back at the audio fmtp line, we do not see any `auxiliaryDataSizeLength` parameter. the absence of this parameter tells us that there's no auxiliary section.  therefore, our access unit, or the aac frame, starts right after the access unit header (from bytes 0xff, 0xf9, ... in our rtp packet).
&nbsp;
note that there could be either a single aac frame, multiple aac frames or even a fragment of an aac frame in an rtp packet. the number of access-unit headers indicate the number of frames - one header for each frame - and looking at the access-unit length information in each header, we can parse the aac frames accordingly.
