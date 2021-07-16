---
layout: post
title:  "implementing digest authentication using openssl"
---
recently, i got an opportunity to implement an rtsp client in janus because of certain limitations of libcurl. the rtsp source i was connecting to uses digest access authentication to authenticate a stream viewer. digest access authentication isn't new to me since i'd already implemented it in [my rtsp client library in elixir](https://github.com/abhijeetbhagat/hadean/blob/master/lib/auth/digest.ex). but that was last year and after a lot of intense squinting at the implementation, i could somewhat figure out how i was calculating the response string. fun times playing with a functional programming language.
&nbsp;
nevertheless, the [wiki](https://en.wikipedia.org/wiki/Digest_access_authentication) explains all the steps and it was time to implement it ... in c. the algorithm is pretty simple and the only thing we need is a library that has md5 support. luckily, janus uses openssl and openssl has md5 support.
&nbsp;
**how does digest authentication work with rtsp?**
&nbsp;
every rtsp connection starts with sending a plain DESCRIBE command. when digest auth is enabled by the rtsp server, however, we get a `401 unauthorized` error:
&nbsp;
```
RTSP/1.0 401 Unauthorized
CSeq: 1
Date: Fri, Jul 16 2021 07:44:57 GMT
WWW-Authenticate: Digest realm="LIVE555 Streaming Media", nonce="138d853ee8de434ee4c90f91b33fcbcb"

```
&nbsp;
we receive a `WWW-Authenticate` line with `realm` and `nonce` values from the server. this is when our digest auth algorithm kicks in.
&nbsp;
**calculating the response string**
&nbsp;
the rtsp source in this case doesn't adhere to the [latest rfc](https://datatracker.ietf.org/doc/html/rfc2617) but the [older one](https://datatracker.ietf.org/doc/html/rfc2069). so i am implementing the auth accordingly, although, the latest one should be preferred because of additional security features provided. first, we extract the realm and nonce values. then we create an md5 context and calculate the hash (m1) of `username:realm:password`:
&nbsp;
```
#include <openssl/md5.h>
...
unsigned char digest[16];
MD5_CTX md5_context;
MD5_Init(&md5_context);
MD5_Update(&md5_context, rtsp_username, strlen(rtsp_username));
MD5_Update(&md5_context, ":", 1);
MD5_Update(&md5_context, realm, strlen(realm));
MD5_Update(&md5_context, ":", 1);
MD5_Update(&md5_context, rtsp_password, strlen(rtsp_password));
MD5_Final(digest, &md5_context);

char m1_hash[33];
for(int i = 0; i < 16; ++i)
  sprintf(&m1_hash[i*2], "%02x", (unsigned int)digest[i]);
```
&nbsp;
the loop simply converts the digest into a hex string. next, we calculate the hash (m2) of `command:url` -
&nbsp;
```
...
MD5_Init(&md5_context);
MD5_Update(&md5_context, "DESCRIBE", strlen("DESCRIBE"));
MD5_Update(&md5_context, ":", 1);
MD5_Update(&md5_context, rtsp_url, strlen(rtsp_url));
MD5_Final(digest, &md5_context);

char m2_hash[33];
for(int i = 0; i < 16; ++i)
  sprintf(&m2_hash[i*2], "%02x", (unsigned int)digest[i]);
```
&nbsp;
and in the last part, we calculate the response string as `m1:nonce:m2` -
&nbsp;
```
MD5_Init(&md5_context);
MD5_Update(&md5_context, m1_hash, strlen(m1_hash));
MD5_Update(&md5_context, ":", 1);
MD5_Update(&md5_context, nonce, strlen(nonce));
MD5_Update(&md5_context, ":", 1);
MD5_Update(&md5_context, m2_hash, strlen(m2_hash));
MD5_Final(digest, &md5_context);

char response[33];
for(int i = 0; i < 16; ++i)
  sprintf(&response[i*2], "%02x", (unsigned int)digest[i]);
```
&nbsp;
after calculating the response string, we can then send this in a new DESCRIBE command:
&nbsp;
```
DESCRIBE rtsp://some-rtsp-server/stream RTSP/1.0
CSeq: 2
Accept: application/sdp
Authorization: Digest username="abhi", realm="LIVE555 Streaming Media", nonce="138d853ee8de434ee4c90f91b33fcbcb", uri="rtsp://some-rtsp-server/stream", response="d8ddca5075e187287533536968b27d3a"

```
&nbsp;
and the server responds with a 200 OK:
&nbsp;
```
RTSP/1.0 200 OK
CSeq: 2
Date: Fri, Jul 16 2021 07:44:58 GMT
...
<sdp redacted for brevity> 

```
&nbsp;
this is not the end though. everytime we send an rtsp command going forward, we need to repeat the steps where we -
a. calculate the hash (m2) - `command:url`
b. calculate the response string - `m1:nonce:m2`
c. send the response string part of the rtsp command leaving rest of the params in the `Authorization` line unchanged.
