---
layout: post
title: Overriding dependencies in rust
category: rust 
tags: [rust, cargo, crate]
---

In this post we will use the **cargo** [overriding dependencies](https://doc.rust-lang.org/cargo/reference/specifying-dependencies.html#overriding-dependencies) feature
 to build a binary with a modified crate. It comes from my other post [Creating an Android virtual input device using rust and uinput]({% post_url 2018-11-03-android-virtual-input-with-rust %}) and the objective is to fix the compilation issue with `ioctl-sys`, to be able to build the project for Android.

In this case the depedencies that insterest us are:
```
rust-touch > uinput > uinput-sys > ioctl-sys
```
As I said the issue we are facing is with `ioctl-sys`, but our `rust-touch` project doesn't depend on it directly but through other dependencies. We need to make that when **Cargo** tries to resolve the dependencies use our replacemente(s) instead.

# Finding a proper fix

Before doing anything, let's do a quick analysis on where and how to fix the issue. The error in the build is:
 ```bash
   Compiling ioctl-sys v0.5.2
   Compiling bytes v0.4.10
error[E0432]: unresolved import `platform_not_supported`
  --> /home/bdemartino/.cargo/registry/src/github.com-1ecc6299db9ec823/ioctl-sys-0.5.2/src/lib.rs:16:5
   |
16 | use platform_not_supported;
   |     ^^^^^^^^^^^^^^^^^^^^^^ no `platform_not_supported` in the root

error: aborting due to previous error

For more information about this error, try `rustc --explain E0432`.
error: Could not compile `ioctl-sys`.
warning: build failed, waiting for other jobs to finish...
error: build failed
```

If we go to the conflicting file [lib.rs](https://github.com/jmesmon/ioctl/blob/master/ioctl-sys/src/lib.rs) in the `ioctl-sys` crate's repository we can see that it only supports **linux** and **macos**, while our **android** target will try to use `platform_not_supported` and error out. Since `ioctl` in Android is common to Linux, a simple fix would be to add **android** as `target_os` in the crate wherever **linux** is used. For example `lib.rs` would look like:
```rust
use std::os::raw::{c_int, c_ulong};

#[cfg(any(target_os = "linux", target_os = "macos", target_os = "android"))]
#[macro_use]
mod platform;

#[cfg(any(target_os = "linux", target_os = "macos", target_os = "android"))]
pub use platform::*;

extern "C" {
    #[doc(hidden)]
    pub fn ioctl(fd: c_int, req: c_ulong, ...) -> c_int;
}

#[cfg(not(any(target_os = "linux", target_os = "macos", target_os = "android")))]
use platform_not_supported;
```

Another alternative is to replace the dependency in `rust-uinput-sys` from `ioctl-sys` to `nix`, a crate that also provides the `ioctl` functionality and already compiles for Android. Although this might be a better alternative in the long run, in this tutorial I'll just update the `ioctl-sys` crate since the code changes are simpler, allowing us to focus on dependency overriding.

# Getting the code

To start working, we need to get a local copy of the code. If you only need the changes for yourself, you can just clone the crate repository. For example in this case you could do:
```bash
$ git clone https://github.com/jmesmon/ioctl.git
```

But if you plan to share the changes or even do a pull request to the original crate, you should first fork the project and then clone the fork. In my case I'll clone my own fork:
```bash
$ git clone https://github.com/brunodmt/ioctl.git
```

# Using the local copy

We now need to update our `rust-touch` project to use our local copy of `ioctl-sys` instead of the repository version. Assuming you cloned the crate sources next to your `rust-touch` project folder, open the `rust-touch/Cargo.toml` file and add:
```toml
[patch.crates-io]
ioctl-sys = { path = "../ioctl/ioctl-sys" }
```

If you want to be sure the local copy is used, run
```bash
$ cargo update -p ioctl-sys --precise 0.5.2
```
where 0.5.2 matches the exact version you cloned.

# Fixing the error

Now we need to update the code. In this case it's simple, we only need to add `target_os = "android"` wherever `target_os = "linux"` is used in the current code. As a reference here is the diff:

```diff
diff --git a/ioctl-sys/src/lib.rs b/ioctl-sys/src/lib.rs
index 73fe28b..8fbfd86 100644
--- a/ioctl-sys/src/lib.rs
+++ b/ioctl-sys/src/lib.rs
@@ -1,10 +1,10 @@
 use std::os::raw::{c_int, c_ulong};
 
-#[cfg(any(target_os = "linux", target_os = "macos"))]
+#[cfg(any(target_os = "linux", target_os = "macos", target_os = "android"))]
 #[macro_use]
 mod platform;
 
-#[cfg(any(target_os = "linux", target_os = "macos"))]
+#[cfg(any(target_os = "linux", target_os = "macos", target_os = "android"))]
 pub use platform::*;
 
 extern "C" {
@@ -12,5 +12,5 @@ extern "C" {
     pub fn ioctl(fd: c_int, req: c_ulong, ...) -> c_int;
 }
 
-#[cfg(not(any(target_os = "linux", target_os = "macos")))]
+#[cfg(not(any(target_os = "linux", target_os = "macos", target_os = "android")))]
 use platform_not_supported;
diff --git a/ioctl-sys/src/platform/mod.rs b/ioctl-sys/src/platform/mod.rs
index 6efdb10..a9665b3 100644
--- a/ioctl-sys/src/platform/mod.rs
+++ b/ioctl-sys/src/platform/mod.rs
@@ -4,7 +4,7 @@ pub const NRBITS: u32 = 8;
 pub const TYPEBITS: u32 = 8;
 
 
-#[cfg(target_os = "linux")]
+#[cfg(any(target_os = "linux", target_os = "android"))]
 #[path = "linux.rs"]
 mod consts;
```

# Cross-compiling, finally!

Now we can build our `rust-touch` project successfully (and continue with the previous [post]({% post_url 2018-11-03-android-virtual-input-with-rust %})):
```bash
$ cargo build --target armv7-linux-androideabi --release
   Compiling rust-touch v0.1.0 (file:///home/bdemartino/Workspace/rust-touch)
    Finished release [optimized] target(s) in 2.11s
```

# Extra step: Overriding with a repository

Having the fix in place you can now go ahead and commit and push the `ioctl` changes to your fork. Once you've done that, you can update the dependency `patch` to use a git repo and branch instead of hardcoding a local path. For example for my fork it would be:
```toml
[patch.crates-io]
ioctl-sys = { git = "https://github.com/brunodmt/ioctl", branch = "add-android-support" }
```
If you build the project now you'll be using that GitHub version (if you aren't sure, you can rename your local copy). The project can now be distributed without needing to manually distribute the updated crate or modifying the sources.
