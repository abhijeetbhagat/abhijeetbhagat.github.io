---
layout: post
title:  "Setting up vim and coc for rust dev"
date:   2021-04-04 20:18:41 +0530
categories: vim coc rust
---
Install `coc` using `vim.plug` plugin manager by adding this to `vimrc`:
&nbsp;
```
call plug#begin('~/.vim/plugged')
...
Plug 'neoclide/coc.nvim', {'branch': 'release'}
...
call plug#end()
```
&nbsp;
Run `:PlugInstall` within vim to install `coc`.
&nbsp;
After `coc` is installed, run within vim:
&nbsp;
```
:CocInstall coc-rust-analyzer
```
&nbsp;
to install `coc-rust-analyzer`.
&nbsp;
Set `RUST_SRC_PATH` to `%USERPROFILE%\.rustup\toolchains\nightly-x86_64-pc-windows-msvc\lib\rustlib\src\rust\library`




