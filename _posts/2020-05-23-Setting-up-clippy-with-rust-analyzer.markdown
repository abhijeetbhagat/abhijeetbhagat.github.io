---
layout: post
title:  "Setting up clippy with rust-analyzer in VS Code"
---
[TL;DR](#perform)
&nbsp;
I've recently started contributing to [boa](https://github.com/jasonwilliams/boa), an experimental JS interpreter being implemented in rust. I submitted a [PR](https://github.com/jasonwilliams/boa/pull/410) that would build 'fine' on my setup but the CI could not accept it. This was due to [clippy](https://github.com/rust-lang/rust-clippy), a linter for rust, rejecting my changes. And so, I got an excuse to start using clippy for all the current and future development in rust.
&nbsp;
I use VS Code for rust development and had the [RLS extension](https://github.com/rust-lang/rls-vscode) installed. RLS is a language server that provides a standard interface for IDEs, editors and tools to interact with rust eco-system. But it seems that [rust-analyzer](https://github.com/rust-analyzer/rust-analyzer) is a more preferred option nowadays in the community. So I installed the rust-analyzer extension for VS Code without disabling/uninstalling the RLS extension. This however caused a problem because whenever I saved a file, RLS would show an error related to my Cargo.toml showing squiggly lines. But `cargo build` never showed any problems. 
&nbsp;
I continued living with it, but after stumbling upon a suggestion on the internet to uninstall the RLS extension, I did just that and there were no more squiggly lines in the editor. 
&nbsp;
Pro-tip: Don't allow RLS and rust-analyzer to co-exist - at least, in the VS Code world.
&nbsp;
Now, when a file is saved, rust-analyzer kicks in and runs its analysis on the code. For this it invokes `cargo check` to check all of a package's dependencies for errors. But it doesn't do any linting. Oblivious to the fact that boa integrates clippy in its CI, I focused only on the warnings spewed by the rust compiler and fixed them. The realization to use clippy dawned upon me when the CI would not allow my build to pass.
&nbsp;
## How to make rust-analyzer run clippy in order to perform linting? <a name="perform"></a>
&nbsp;
Setup [rust-analyzer](https://marketplace.visualstudio.com/items?itemName=matklad.rust-analyzer) if you haven't already.
&nbsp;
First, we need to install clippy itself by running:
&nbsp;
`$ rustup component add clippy --toolchain nightly-x86_64-pc-windows-gnu`
&nbsp;
I am using the nightly gnu toochain on Windows. We can find what our active toolchain is by:
&nbsp;
```
$ rustup show
...
nightly-x86_64-pc-windows-gnu (default)
rustc 1.44.0-nightly (f509b26a7 2020-03-18)
```
&nbsp;
Once clippy is made part of our eco-system, we can now make rust-analyzer aware of it by changing its `checkOnSave` behavior to running clippy instead of `cargo check`. We can do this by opening the `package.json` file inside the rust-analyzer plugin folder which for me is `\path\to\my\profile\dir\.vscode\extensions\matklad.rust-analyzer-0.2.174`. Depending on your rust-analyzer version, the last folder name will be different.
&nbsp;
In the file, navigate to `"rust-analyzer.checkOnSave.command"` and replace the value for the `default` key from `"check"` to `"clippy"`. After this, reload the VS Code window by running `Reload Window` from the Command Palette. VS Code then tells you that the extension has changed and the window needs to be reloaded. Do it!
&nbsp;
Now, to see if clippy is invoked when you save a file, make a modification to your code and hit save. On the status bar at the bottom, you should see the text 'cargo clippy' with a snake like animation before it indicating that it is running and the linting output (if any) shown in the PROBLEMS panel.
&nbsp;
But wait, now that we've replaced `cargo check` with clippy, what happens to checking a package's dependencies for errors? Clippy internally invokes `cargo check` so we need not worry about it anymore.
