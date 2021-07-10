---
layout: post
title:  "Adding support for tcp based rtsp streaming in janus"
---

janus is a cute little webrtc gateway implemented in c. one of the features it supports is streaming from an rtsp source. however, it only supports rtsp streaming over udp. for cases where tcp based streaming is required such as from surveillance cams or re-streamers that support only tcp based streaming, janus will not be useful. so i decided to add this support (still in progress).
&nbsp;
**how is tcp based streaming different from udp based streaming?**
&nbsp;
in a udp based streaming mode, the rtsp commands - OPTIONS, DESCRIBE, SETUP, etc. - are sent over a tcp socket, since rtsp itself is a tcp based protocol. the media and rtcp data, however, flows on different udp sockets. in a tcp based streaming mode, everything flows over a single tcp socket in an interleaved fashion.
&nbsp;
**how to proceed?**
&nbsp;
rtsp streaming in janus is taken care of by the streaming plugin (`plugins/janus_streaming.c`). this plugin uses the [curl library](https://curl.se/libcurl/) to send rtsp commands:
&nbsp;
```
...
CURL *curl = curl_easy_init();
curl_easy_setopt(curl, CURLOPT_NOPROGRESS, 1L);
curl_easy_setopt(curl, CURLOPT_URL, source->rtsp_url);
curl_easy_setopt(curl, CURLOPT_RTSP_REQUEST, (long)CURL_RTSPREQ_DESCRIBE);
curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, janus_streaming_rtsp_curl_callback);
// set a couple of options here ...
int res = curl_easy_perform(curl);
...
```
&nbsp;
libcurl itself takes care of creating a tcp socket and send/recv data over it. we do not have any control we want over this socket that we want to receive the rtp packets from once we send the PLAY command. the `janus_streaming_rtsp_curl_callback` callback is invoked when a response is recvd. but it can't be used to recv rtp packets. 
&nbsp;
**libcurl and external sockets**
&nbsp;
luckily, libcurl provides a way to work with 'external sockets' such as one created using the `socket` syscall.
&nbsp;
```
static curl_socket_t opensocket(void *clientp,
                                curlsocktype purpose,
                                struct curl_sockaddr *address)
{
  curl_socket_t sockfd;
  (void)purpose;
  (void)address;
  sockfd = *(curl_socket_t *)clientp;
  /* the actual externally set socket is passed in via the OPENSOCKETDATA
     option */
  return sockfd;
}

static int sockopt_callback(void *clientp, curl_socket_t curlfd,
                            curlsocktype purpose)
{
  (void)clientp;
  (void)curlfd;
  (void)purpose;
  /* This return code was added in libcurl 7.21.5 */
  return CURL_SOCKOPT_ALREADY_CONNECTED;
}

...
curl_socket_t sockfd = socket(AF_INET, SOCK_STREAM, 0);
CURL *curl = curl_easy_init();
curl_easy_setopt(curl, CURLOPT_NOPROGRESS, 1L);

sockaddr_in remote_addr;
memset(&remote_addr, 0, sizeof(remote_addr));
remote_addr.sin_family = AF_INET;
remote_addr.sin_port = htons(554);
remote_addr.sin_addr.s_addr = inet_addr("1.2.3.4");

if (connect(sockfd, (struct sockaddr *) &remote_addr, sizeof(remote_addr)) == -1) {
  close(sockfd);
  JANUS_LOG(LOG_ERR, "couldn't connect to remote addr. errno: %s\n", strerror(errno));
  return -1;
}

curl_easy_setopt(curl, CURLOPT_URL, source->rtsp_url);
curl_easy_setopt(curl, CURLOPT_OPENSOCKETFUNCTION, opensocket);
curl_easy_setopt(curl, CURLOPT_OPENSOCKETDATA, &sockfd);
curl_easy_setopt(curl, CURLOPT_SOCKOPTFUNCTION, sockopt_callback);
curl_easy_setopt(curl, CURLOPT_RTSP_STREAM_URI, source->rtsp_url);
curl_easy_setopt(curl, CURLOPT_RTSP_REQUEST, (long)CURL_RTSPREQ_DESCRIBE);
curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, janus_streaming_rtsp_curl_callback);
// set a couple of options here ...
int res = curl_easy_perform(curl);
...
```
&nbsp;
above, we created our socket and passed it to `curl_easy_setopt` with the `CURLOPT_OPENSOCKETDATA` flag. this will now cause libcurl to use our socket to connect to the rtsp source. the `janus_streaming_rtsp_curl_callback` callback will still be invoked to recv the responses to the commands sent. however, now we can simply use the `recv` syscall to read the rtp packets streamed by the source:
&nbsp;
```
gint8_t *buf = (gint8_t*)g_malloc0(1500);
ssize_t num_bytes = recv(sockfd, buf, 1500, 0);
// do something with buf containing rtp data
g_free(buf)
```
&nbsp;
cool! so we got the rtp stream (as well as rtcp if any) flowing over tcp in janus. but ... there is a small problem.
&nbsp;
**extracting rtp packets**
&nbsp;
in the udp based streaming mode, an entire datagram recvd on a media port contains a single rtp packet. in the tcp mode, however, situation is a little tricky. a single `recv` operation can either -
a. read a single rtp packet
b. read multiple rtp packets
c. read a partial rtp packet
d. read only the marker bytes
&nbsp;
for e.g.,
&nbsp;
[0x24, 0, 0, 5, ff, ff, ff, ff, ff] -> a single rtp packet
[0x24, 0, 0, 5, ff, ff, ff, ff, ff, 24, 0, 0, 3, ff, ff, ff] -> 2 rtp packets in a single read
[0x24, 0, 0, 3, ff, ff, ff, 24, 0, 0, 5, ff, ff] -> last rtp packet is partial; remaining packet data available in the next read
[0x24, 0, 0, 6] -> only marker bytes read; entire packet available in the next read
&nbsp;
the rtp packet extraction algorithm is therefore non-trivial and has to consider all the above scenarios:
&nbsp;
```
uint16_t rtp_packet_len = 0;
uint16_t offset_to_next_rtp_packet = 0;
uint16_t num_pending_rtp_bytes = 0;
uint8_t *rtp_data;

while (1) {
  ssize_t num_bytes = recv(sockfd, buf, 1500, 0);
  signed short count = 0; 

  while (count < num_bytes) {
    if (num_pending_rtp_bytes > 0) {
      // if yes, copy the pending bytes from the start of the buf to the proper offset
      // in the rtp_data array
      memcpy(rtp_data + (rtp_packet_len - num_pending_rtp_bytes), buf, num_pending_rtp_bytes);

      // reduce `count` by 4 because after the rtp packet is processed,
      // count will be incremented by (count + offset_to_next_rtp_packet + 4) to point to '$'
      // of the next rtp packet.
      count -= 4; // num_pending_rtp_bytes;
      // rtp_packet_len needs to be set to the pending bytes read
      // since the count will be incremented by the rtp_packet_len
      // after the rtp packet is processed.
      offset_to_next_rtp_packet = num_pending_rtp_bytes;

      if (num_bytes < num_pending_rtp_bytes) {
        // we still have more bytes to read
        num_pending_rtp_bytes -= num_bytes;
        break;
      } else {
        num_pending_rtp_bytes = 0;
      }
      // we now have a complete previous rtp packet in rtp_data
      // so process it ...
      goto process;
    } else {
      // we may have marker bytes
      if (buf[count] == 0x24) {
        // combile the last 2 bytes to get the length of this rtp packet
        rtp_packet_len = ((uint16_t)buf[count + 2] << 8) | (uint16_t)buf[count + 3];
      } else {
        rtp_packet_len = num_bytes;
      }

      // allocate some space for our rtp packet
      rtp_data = (uint8_t *)g_malloc0(rtp_packet_len);
    }
    
    // check if we have a partial rtp packet or a full rtp packet
    if ((num_bytes - (count + 4)) < rtp_packet_len) { // partial packet
      // do we have any bytes of the current rtp packet to read?
      int room = num_bytes - (count + 4);
      if (room > 0) {
        memcpy(rtp_data, buf + (count + 4), room);
        num_pending_rtp_bytes = rtp_packet_len - room;
        count += 4 + room;
      } else { // maybe the buffer contains [$, 0, 0, 3] or [..., $, 0, 0, 3]
        num_pending_rtp_bytes = rtp_packet_len;
      }
      // we now continue with next read operation
      break;
    } else { // full packet
      memcpy(rtp_data, buf + (count + 4), rtp_packet_len);
    }

    offset_to_next_rtp_packet = rtp_packet_len;

process:
    // do something with the rtp packet

    g_free(rtp_data);
    count += (offset_to_next_rtp_packet + 4);
  }
}
```
&nbsp;
the extracted rtp packets can now be relayed to all the webrtc viewers.
