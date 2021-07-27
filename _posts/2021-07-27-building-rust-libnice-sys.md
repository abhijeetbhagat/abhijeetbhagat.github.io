---
layout: post
title:  "building rust-libnice-sys on ubuntu"
---
`rust-libnice-sys` is a crate that provides rust bindings to [libnice](https://libnice.freedesktop.org/) - a c library implementing the ice protocol. the 
process is straight-forward:
&nbsp;
1. clone the repo
2. cd into the repo
3. `cargo build`

&nbsp;
it didn't quite work as expected on ubuntu:
&nbsp;
  ```
  --- stderr
  /usr/include/glib-2.0/glib/gmacros.h:38:10: fatal error: 'stddef.h' file not found
  /usr/include/glib-2.0/glib/gmacros.h:38:10: fatal error: 'stddef.h' file not found, err: true
  thread 'main' panicked at 'Unable to generate bindings: ()', build.rs:53:10
  note: run with `RUST_BACKTRACE=1` environment variable to display a backtrace
  ```
&nbsp;
no idea why that header isn't found. after installing `clang`, it worked!
