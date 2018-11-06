---
layout: post
title: Creating an Android virtual input device using rust and uinput
category: rust 
tags: [rust, android, uinput]
---

In this post we will create a virtual input device that injects touch events in the system in [rust](https://www.rust-lang.org) using [uinput](https://www.kernel.org/doc/html/v4.12/input/uinput.html) and then we'll cross compile it to run on Android devices.

This tutorial is focused on how to get the project built and running in Android devices and doesn't go deep about [rust](https://www.rust-lang.org), [uinput](https://www.kernel.org/doc/html/v4.12/input/uinput.html) nor the [Linux Multi-touch (MP) Protocol](https://www.kernel.org/doc/html/v4.16/input/multi-touch-protocol.html), so I recommend that if your not familiar with them you give a quick visit to the links first. Since we'll be using uinput you'll need a Linux computer to be able to test your code.

# Initializing the project

We first need to create a rust project and add some dependencies as usual. Luckily for us there's already a uinput ~~library~~ [crate](https://crates.io/crates/uinput) that wraps the C calls and exposes a high level interface. The source with some examples can be found in its [repo](https://crates.io/crates/uinput).

Initialize the project:
```bash
$ cargo init rust-touch  
    Created binary (application) project
```

Then open the generated rust-touch/Cargo.toml file and add the dependency at the end:
```toml
[dependencies]
uinput = "0.1.3"
```

# Coding

We'll now write the code that initializes our virtual device and sends some hardcoded touches. To start open your src/main.rs.

First you need to tell rust that our main will be using the crate we added as dependency, and then import all the names we need to avoid repeating them all over the code:
```rust
extern crate uinput;
 
use uinput::event::Absolute::Multi;
use uinput::event::absolute::Multi::{PositionX,PositionY,Slot,TrackingId};
use uinput::event::Controller::Digi;
use uinput::event::controller::Digi::Touch;
use uinput::event::Event::Absolute;
use uinput::event::Event::Controller;
```

We'll also import some rust std names to add delays between each input event:
```rust
use std::thread;
use std::time::Duration;
```

Now our main function, the explanation is inline:

```rust
fn main() {
    // Initialize a device
    let mut device = uinput::default().unwrap()
        // Set the device name
        .name("Rust Touch Device").unwrap()
        // Now add the different events that will be reported
        // The first line says the device can report PositionX events as part
        // of an Absolute device of Multi-touch (MT) type
        .event(Absolute(Multi(PositionX))).unwrap()
        // For these events the bounds are
        .min(0).max(1000)
        // Same goes for PositionY
        .event(Absolute(Multi(PositionY))).unwrap()
        .min(0).max(1000)
        // Being an MT device it needs to report what Slot is using
        .event(Absolute(Multi(Slot))).unwrap()
        // In this case only one touch at the same time will be used
        .min(0).max(0)
        // Also a TrackingId needs to be reported to identify a contact
        .event(Absolute(Multi(TrackingId))).unwrap()
        // Finally, the BTN_TOUCH event needs to be send for the OS to actually
        // consider the touch. The used library puts this touch Event as part
        // of a Controller of type Digi, so that's what the following line
        // adds
        .event(Controller(Digi(Touch))).unwrap()
        // Once the device is set up properly, initialize it in the OS
        .create().unwrap();
 
    // Wait for the device to be fully initialized 
    thread::sleep(Duration::from_secs(1));

    // In this example, the touch can be always active, so set it as pressed
    // for the lifetime of the execution
    device.press(&Touch).unwrap();
 
    // Although only one touch is used in the example, the full MT protocol is
    // followed here, so we can extend the example later

    // Select the slot that will be used for the upcoming events
    device.position(&Slot,0).unwrap();

    // Set a contact to that slot
    device.position(&TrackingId,1).unwrap();
    // Horizontally center the touch in the device
    device.position(&PositionX, 500).unwrap();

    // Now move the touch vertically from top to bottom
    for i in 0..1000 {
        device.position(&PositionY, i).unwrap();
        // Send a SYNC so that the OS process the events
        device.synchronize().unwrap();
        // And wait 10ms in between to get a visible effect
        thread::sleep(Duration::from_millis(10));
    }

    // Finally release the contact
    device.position(&TrackingId,-1).unwrap();
    device.synchronize().unwrap();
 
}

```

You'll notice we've used a lot of unwrap calls, since this is an example we don't care much about error handling, we only need to panic if something goes wrong. You can find the final code in [GitHub](https://github.com/brunodmt/rust-touch).

Now, running `cargo run` will download the dependencies, build and run the code. Once it is running you'll see the mouse pointer going from the top to the bottom of the screen.

# Setting up the cross-compilation environment

In this section we'll set up our environment to be able to cross-compile for Android. The instructions are based on [Building and Deploying a Rust library on Android](https://mozilla.github.io/firefox-browser-architecture/experiments/2017-09-21-rust-on-android.html), but adapted to just build a binary.

First, get the required dependencies from the Android SDK, you can do it by going to **File > Settings > Appearance & Behavior > System Settings > Android SDK > SDK Tools** in Android Studio, checking the following options and clicking **OK**:
* Android SDK Tools
* NDK

Once everything is installed the SDK and NDK paths have to be defined as environment variables, for example:
```bash
export ANDROID_HOME=/home/$USER/Android/Sdk
export NDK_HOME=$ANDROID_HOME/ndk-bundle
```

Now we can create our standalone NDK to use with Rust. In your project folder run:
```bash
$ mkdir NDK
$ ${NDK_HOME}/build/tools/make_standalone_toolchain.py --api 26 --arch arm --install-dir NDK/arm
```

Here we're only creating the toolchain for **arm**, but if you need **arm64** or **x86** you can create those too adjusting the parameters. You should also adjust the **api** parameter to your needs.

Next we need to create a **cargo** configuration to use that toolchain, do it by creating a `cargo-config.toml` file with:
```toml
[target.armv7-linux-androideabi]
ar = "<project path>/NDK/arm/bin/arm-linux-androideabi-ar"
linker = "<project path>/NDK/arm/bin/arm-linux-androideabi-clang"
```

Replacing `<project path>` with your project's absolute path. Again, if you want to use different architectures you can do it here adapting the sample configuration.

Then, install the config file doing:
```bash
$ cp cargo-config.toml ~/.cargo/config
```

Finally install the **rust standard library** for the desired architecture doing:
```bash
$ rustup target add armv7-linux-androideabi
```

We are now ready to cross-compile.

# Cross-compiling, attempt 1

Since we now have all the tools in place, let's build and run in Android:
```bash
$ cargo build --target armv7-linux-androideabi --release
```

Unfortunately the build fails:
```bash
error: failed to run custom build command for `libudev-sys v0.1.4`
process didn't exit successfully: `/home/bdemartino/Workspace/rust-touch/target/release/build/libudev-sys-42934ef269061d82/build-script-build` (exit code: 101)
--- stderr
thread 'main' panicked at 'called `Result::unwrap()` on an `Err` value: "Cross compilation detected. Use PKG_CONFIG_ALLOW_CROSS=1 to override"', libcore/result.rs:945:5
note: Run with `RUST_BACKTRACE=1` for a backtrace.
```

We see there's a problem with **udev**, which is reasonable since Android doesn't have **udev**. If we want more information we can run the build with the suggested flags:
```bash
$ PKG_CONFIG_ALLOW_CROSS=1 RUST_BACKTRACE=1 cargo build --target armv7-linux-androideabi --release
```

There we confirm that the problem is that the **udev** library is not available. We also see that the **uinput** crate is the one looking for it:
```bash
/home/bdemartino/Workspace/rust-uinput/NDK/arm/bin/../lib/gcc/arm-linux-androideabi/4.9.x/../../../../arm-linux-androideabi/bin/ld: error: cannot find -ludev
          /home/bdemartino/Workspace/rust-touch/target/armv7-linux-androideabi/release/deps/libuinput-729bb1148dfb6b8f.rlib(uinput-729bb1148dfb6b8f.uinput5-ca35924e8a5ca31ff9d61156babea35.rs.rcgu.o):uinput5-ca35924e8a5ca31ff9d61156babea35.rs:function uinput::device::builder::Builder::default::h8e87e40706b1a7c5: error: undefined reference to 'udev_enumerate_add_match_subsystem'

```

To fix it, we need to go to the **uinput** source and see how we can disable the **udev** dependency. Luckily for us if we go to the [Cargo.toml](https://github.com/meh/rust-uinput/blob/master/Cargo.toml) file of the crate, we see that the **udev** dependency is declared as a **[feature](https://doc.rust-lang.org/cargo/reference/manifest.html#the-features-section)**, meaning we can enable/disable it as we want:

```toml
[dependencies.libudev]
optional = true
version  = "0.2"

[features]
default = ["udev"]
udev = ["libudev"]
```

So we can update our `Cargo.toml` dependency section to disable that feature:
```toml
[dependencies.uinput]
version = "0.1.3"
default-features = false
```

# Cross-compile, attempt 2

Let's run the build again:
```bash
$ cargo build --target armv7-linux-androideabi --release
```

If everything is OK we'll get something like:
```bash
Finished release [optimized] target(s) in 32.23s 
```

# Running on target

Once the buid is OK, you can find the executable in :
```
target/armv7-linux-androideabi/release
```

So you can install it on Android by doing:
```bash
$ adb push target/armv7-linux-androideabi/release/rust-touch /data/local/tmp
```

and run then run it with:
```bash
$ adb shell /data/local/tmp/rust-touch
```

You should see the same effect as if you were moving your finger from the top to the bottom of the screen. Depending what you have on the screen at the moment it will be more or less visible, but you can always enable the **Show taps** option from **Developer Options**

If you find any permission issue, check that the binary is executable after pushing it or try running it as **root**.
