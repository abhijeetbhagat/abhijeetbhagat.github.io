---
layout: post
title:  "Bypassing CORS problem when making cross domain requests"
---

!CAUTION! Exercise caution when playing with security features of browsers.
===========================================================================
&nbsp;
I am working on a chat application and wanted to check if the CouchDB instance running on my VM could be accessed using a JS wrapper library. Now, I am not a full-stack or even a JS developer by any means and have barely worked on the front-end tech a few occasions eons ago. Combining my very limited browser apps related experience with the power of the internet, it feels cool to be able to work on JS, albeit, with occasional stumbling blocks.
&nbsp;
Now I am using Chrome for the development, so I'll keep everything I write here related to Chrome. This may apply to other browsers. Who knows! 
&nbsp;
Anyway, one of the problems I recently faced is CORS or Cross Origin Resource Sharing. CORS seems to be security mechanism that enables browsers to disallow making requests (as in HTTP requests) to a domain which the script is not part of. We'll see what that means in minute but how on earth did I even come to know about it? Well, to play with CouchDB using the JS library I talked about earlier, I created a simple HTML page and added a couple of script tags representing the library and its dependency. The library loaded fine and I was now ready to make some calls into it.
&nbsp;
First, I set a URL-prefix of the CouchDB server. In my case it is http://172.16.255.140:5984 (this is where the trouble starts). Next, to check if the library is of any use, I tried to fetch a list of databases. All this in the devtools console: 
&nbsp;
```
$.couch.urlPrefix = "http://172.16.255.140:5984"
$.couch.allDbs({success: function(dbs){console.log(dbs);}})
```
&nbsp;
However, I was greeted with this instead:
&nbsp;
*Access to XMLHttpRequest at 'http://172.16.255.140:5984/_all_dbs' from origin 'null' has been blocked by CORS policy: Response to preflight request doesn't pass access control check: No 'Access-Control-Allow-Origin' header is present on the requested resource.*
&nbsp;
Intimidated by the words in the above error, I went to the internet to see what was up. Turns out, when a browser finds out that an HTTP request is being made to a different domain, it first checks if that domain accepts requests from a different domain. In our case, we loaded a simple HTML file in the browser so that it loaded the CouchDB library and were making requests to the 172.16.255.140... er... domain...? of the CouchDB server. It does this by sending an HTTP request with the [OPTIONS](https://tools.ietf.org/html/rfc7231#section-4.3.7) verb and the server responds with an 'approval'. But looks like the CouchDB server can't approve it and so I had to find a way to disable CORS in Chrome.
&nbsp;
Importantly, this 'preflight request' wasn't visible in the Networks tab of devtools. I had to make it appear by disabling the '[Out of blink CORS](chrome://flags/#out-of-blink-cors)' flag and restarting Chrome. I could now see this OPTIONS preflight request getting sent by Chrome and the server responding saying 'method not allowed'. But our problem is still not solved.
&nbsp;
I then disabled CORS by launching Chrome with extra switches:
&nbsp;
```
$ path\to\chrome\chrome.exe --disable-web-security --disable-gpu --user-data-dir=path\to\some\dir
```
&nbsp;
After it launched, at the very top Chrome warned about security and stability of the browser. Ignoring it for development purposes only, I could now get a list of databases from CouchDB though.

References:
===========
https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS
https://alfilatov.com/posts/run-chrome-without-cors/
