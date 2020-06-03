---
layout: post
title:  "Calling an FFMPEG function from Rust on Windows"
date:   2020-06-02 10:25:41 +0530
categories: ffmpeg
---
After [building and running an FFMPEG example using Visual Studio 2013](https://abhijeetbhagat.github.io/ffmpeg/2020/06/01/Building-an-FFMPEG-example-with-VS-2013.html), let's see how to call the simplest (one that does not have a signature involving composite types) function from FFMPEG from a rust program. FFMPEG is a C library and you can easily call into a C library via rust by using FFI (Foreign Function Interface).
&nbsp;
We'll use the same Windows bundles of FFMPEG from [here](https://ffmpeg.zeranoe.com/builds/). Download and unzip those FFMPEG shared and dev bundles. The shared bundle contains FFMPEG DLLs that are loaded at runtime when we invoke FFMPEG API. The dev bundle contains headers and static libs that aid in building.
&nbsp;
Before we begin though, one important note: the standard way to implement FFI is by using `bindgen`, a crate that parses header files and generates the FFI automatically. But we will give it a rest to understand the process itself in a better way.
&nbsp;
Let's create a new rust project:
&nbsp;
```
$ cargo new --bin ffmpeg-ffi
```
&nbsp;
This creates a `Cargo.toml` file and an src directory containing a `main.rs` file. The first thing we do is add the `libc` crate to our `Cargo.toml` file:
&nbsp;
```toml
[dependencies]
libc = "*"
```
&nbsp;
Why we need this crate will be explained shortly.  We'll invoke the `avcodec_version` function from the `libavcodec\avcodec.h` header so let's create a new file called `ffi.rs` and add the prototype of the `avcodec_version` function to it:
&nbsp;
```rust
extern crate libc;

use self::libc::*;

#[link(name="avcodec")]
extern {
    pub fn avcodec_version() -> c_uint;
}
```
&nbsp;
Its C prototype looks like this:
&nbsp;
```C
unsigned avcodec_version(void);
```
&nbsp;
Here, we've 'translated' the prototype of the `avcodec_version` C function to Rust. Since it returns an unsigned integer, we set the return type to `c_uint` from the `libc` crate (this is why we need libc). We also wrap our function in an `extern` block annotated with the name of the static library (avcodec.lib). On Windows, you can't link directly with a DLL. So you link against a static lib that contains references to symbols from the corresponding DLL.
&nbsp;
When we invoke `cargo build`, it invokes the system linker available on the platform - `ld` on Linux and `link.exe` on Windows (which requires C/C++ build tools or Visual Studio to be installed first). In our case, we are invoking `link.exe` and it needs the location of the FFMPEG static libs; otherwise, it fails. Since we are only experimenting, we won't fiddle with modifying `LIBPATH`/`LIB` (I am not sure) environment variables or even using the developer command prompt that comes with Visual Studio. Instead, we'll copy our FFMPEG static libraries to the path which `link.exe` looks into when invoked by `cargo build`.
&nbsp;
On my machine, this is the `%USERPROFILE%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\x86_64-pc-windows-msvc\lib` folder. Let's copy the `avcodec.lib` there.
&nbsp;
Once we have our 'interface' file ready, and our project should build fine at this point in time but not run, we can now call our `avcodec_version` function in the `main.rs` file:
&nbsp;
```rust
pub mod ffi;
use ffi::*;

fn main() {
    unsafe{
        println!("Hello, ffmpeg {}!", avcodec_version());
    }
}
```
&nbsp;
Since we are calling a function that is in a C library that cannot convey safety guarantees to the rust compiler, we need to wrap our call to `avcodec_version` in an unsafe block.
&nbsp;
We now want to make sure that when we run our program, the right FFMPEG DLL containing implementation of the `avcodec_version` function is found and loaded. Let's copy the FFMPEG DLLs to the `target\debug` folder.
&nbsp;
Finally, when we run our program to display the FFMPEG version number:
&nbsp;
```
$ cargo run
Hello, ffmpeg 3824228!
```
