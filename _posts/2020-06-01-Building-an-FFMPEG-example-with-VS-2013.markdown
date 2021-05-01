---
layout: post
title:  "Building an FFMPEG example with VS 2013"
---
I've been trying to get into building a simple video player with FFMPEG on Windows using VS 2013. The first step is to get FFMPEG working. This is what I did - &nbsp;
1. Download and unzip FFMPEG shared and dev bundles from [here](https://ffmpeg.zeranoe.com/builds/). The shared bundle contains FFMPEG DLLs that we'll be loading at runtime when we invoke FFMPEG API. The dev bundle contains headers and static libs.&nbsp;
2. Open VS 2013 and create an empty VC++ empty project.&nbsp;
3. Add an example C source code file (say decode_video.c) from `/path/to/extracted/dev/bundle/examples` to the project. We'll be using this file in the steps going forward.&nbsp;
4. Since the example file contains an FFMPEG header, we'll need to set the include path in the project. Go to the project properties > Configuration Properties > VC++ Directories > Include Directories and add the `/path/to/extracted/dev/bundle/include` folder.&nbsp;
5. If you compile now, you will get an lot of errors related to the use of the `inline` keyword: &nbsp;
```
1>path\to\extracted\dev\bundle\include\libavutil\error.h(111): error C2054: expected '(' to follow 'inline'
1>path\to\extracted\dev\bundle\include\libavutil\error.h(112): error C2085: 'av_make_error_string' : not in formal parameter list
1>path\to\extracted\dev\bundle\include\libavutil\error.h(112): error C2143: syntax error : missing ';' before '{'
1>path\to\extracted\dev\bundle\include\libavutil\mem.h(669): error C2054: expected '(' to follow 'inline'
...
```
&nbsp;
This is because the VC++ compiler doesn't recognize `inline` but instead recognizes `_inline`. Instead of replacing all the `inline` keywords, we can use a macro for substitution. Open `decode_video.c` file and put this macro BEFORE the include directive to include the `libavcodec/avcodec.h` header: 
&nbsp;
```
#define inline _inline
```
&nbsp;
6. One other compilation error is related to the `sprintf` function. The VC++ compiler instead recognizes `_sprintf` and we can use a macro like above. Put this macro in the `decode_video.c` file: 
&nbsp;
```
#define sprintf _sprintf
```
&nbsp;
Our modifications should now look like this:
&nbsp;
...
#define inline _inline
#include <libavcodec/avcodec.h>
#define snprintf _snprintf
...
&nbsp;
7. After taking care of the compilation errors, we need to tell the linker where the static libs (.lib files) to be linked against are. We can do this by going to the project properties > Configuration Properties > Linker > Input > Additional Dependencies and adding the static libraries residing in the `/path/to/extracted/dev/bundle/lib` folder. For some reason though, this did not work as I got a lot of linker errors:
&nbsp;
&nbsp;
```
1>decode_video.obj : error LNK2019: unresolved external symbol av_frame_alloc referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol av_frame_free referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol av_packet_alloc referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol av_packet_free referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol avcodec_find_decoder referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol avcodec_alloc_context3 referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol avcodec_free_context referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol avcodec_open2 referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol avcodec_send_packet referenced in function decode
1>decode_video.obj : error LNK2019: unresolved external symbol avcodec_receive_frame referenced in function decode
1>decode_video.obj : error LNK2019: unresolved external symbol av_parser_init referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol av_parser_parse2 referenced in function main
1>decode_video.obj : error LNK2019: unresolved external symbol av_parser_close referenced in function main
```
8. We can instead use the `pragma comment` directive in the `decode_video.c` file to fix them:&nbsp;
```
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/avcodec.lib")
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/avfilter.lib")
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/avdevice.lib")
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/avformat.lib")
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/avutil.lib")
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/postproc.lib")
#pragma comment (lib,"/path/to/extracted/dev/bundle/lib/swresample.lib")
```
9. Now it builds fine. But there's still one final problem left. The static libs we linked against do not contain the object files (in other words, the implementation of the FFMPEG functions that we are calling) but instead are import libraries that contain only symbols allowing the linker to link to a DLL that contains actual object code. We must let the runtime find and load the FFMPEG DLLs when needed. One simple way is to copy the FFMPEG DLLs from `/path/to/extracted/shared/bundle/bin` folder to the folder that contains our generated executable.
10. We are now ready to run the exe with a test file. The `decode_video.c` file accepts an input file encoded using mpeg1 encoder. You can get it from [here](http://hubblesource.stsci.edu/sources/video/clips/) (don't worry about the http link; it is safe). Run the exe like:
&nbsp;
&nbsp;
```
$ decode_video.exe centaur_1.mpg
```
&nbsp;
and it should create a bunch of mpeg1 frame files in the same directory.
