---
layout: post
title: Raspberry Pi Bare Metal Programming with Rust
permalink: raspberry-pi-bare-metal-programming-with-rust/
---

After reading the excellent [http://intermezzos.github.io/](http://intermezzos.github.io/), I felt excited again to work on my small kernel, but I was decided to focus on the ARM arch, since it’s been awhile since I bought my Raspberry Pi B+, but never spent some time playing with it. So I started to read more about it on osdev wiki and how to get started with bare metal raspberry development. There isn’t a lot of references for this, especially if you’re using Rust. Most of the tutorials/examples uses C or plain assembly for the task, so it would be fun to try to build these things just using Rust.
So, to get started we can do the embedded systems hello world: blink a led. My goal was to achieve that with as little assembly as possible, bringing all the cool stuff to Rust, and to my surprise, this was something fairly easy to achieve. To get started we’re going to need `arm-none-eabi` toolchain, for cross-compiling, Rust nightly with `libcore` compiled for this architecture, Raspberry’s boot files and a Raspberry Pi B+ to run our code. I’ll walk-through the process.
Setup the cross-compiler
The steps below were done on OSX, but you can easily adapt to Linux. To install the arm-none-eabi toolchain you can either use homebrew or download the tarball, for OSX, Linux and Windows.
With homebrew you just have to:
{% highlight sh %}
$ brew tap nitsky/stm32
$ brew install arm-none-eabi-gcc
$ arm-none-eabi-gcc --version
{% endhighlight %}
arm-none-eabi-gcc (GNU Tools for ARM Embedded Processors) 4.9.3 20150529 (release) [ARM/embedded-4_9-branch revision 227977]
If you’re using the tarball, just download it and add it to your `$PATH`.

## Setup Rust

Some of the Rust’s features we’re going to use are behind a feature gate, so we need to use Rust nightly. To install it you can run:

{% highlight bash %}
$ curl -sSf https://static.rust-lang.org/rustup.sh | sudo sh -s — — channel=nightly -- yes
$ rustc --version
rustc 1.6.0-nightly (bdfb13518 2015–11–14)
{% endhighlight %}


Now we need to compile Rust’s `libcore` to our target architecture. This is a minimal dependency-free foundation of the standard library. For that we need to compile it from source:

{% highlight sh %}
$ git clone https://github.com/rust-lang/rust.git
$ cd rust
$ git checkout bdfb13518
$ mkdir -p /usr/local/lib/rustlib/arm-unknown-linux-gnueabihf/lib
$ rustc --target arm-unknown-linux-gnueabihf -O -Z no-landing-pads src/libcore/lib.rs --out-dir /usr/local/lib/rustlib/arm-unknown-linux-gnueabihf/lib
{% endhighlight %}

Note that the `git checkout` hash I used is the same I get from `rustc — version`, you should use your own to avoid obscure errors.
Raspberry Pi boot files
Those files can be found here: https://github.com/raspberrypi/firmware/tree/master/boot
You’re going to need `bootcode.bin` and `start.elf`. The `kernel.img` is going to be written by us. 
Having a bootable SD card, you can delete all files and just put the `bootcode.bin` and `start.elf` there. Unfortunately, we don’t have access to the code of these files and we have to use the compiled version for now. I’m planning to research more about that and try to figure out how we can write our own boot files, but this is going to be a topic for another post.
Getting started
With all that set, we can start writing code. We’re going to make the ACT led blink, it’s the green one beside the power led. So our program basically need to turn it on, wait for some time, turn it off and loop. So fire up your editor and create our `kernel.rs` file.

{% highlight rust %}

// kernel.rs
fn main() {
 println!(“Hello World!”);
}

{% endhighlight %}

This would be a standard hello world example, but the `println!` macro is part of the rust standard library and uses the platform I/O to print something to the screen and right now we _don’t_ have a platform to run on top! So, we need to inform the compiler that our code freestanding will be freestanding:

{% highlight rust %}
#![feature(no_std)]
#![crate_type = “staticlib”]
#![no_std]
// kernel.rs
fn main() {
 println!(“Hello World!”);
}
{% endhighlight %}

In the first line, we’re informing the compiler that we’re going to use a gated feature `no_std`, it’s with the `#![feature()]` attribute that we inform such things to the compiler. With the `#![crate_type]` we say that we’re compiling a static library, which basically means that all our code is self-contained and finally, with the `#![no_std]` we’re saying that we’re not going to use the standard library, so it doesn’t get automatically loaded in the prelude process. Now, if we try to compile that, we’ll get an error:


{% highlight bash %}
$ rustc --target arm-unknown-linux-gnueabihf kernel.rs
kernel.rs:8:5: 8:12 error: macro undefined: ‘println!’
kernel.rs:8 println!(“Hello World!”);
 ^~~~~~~
error: aborting due to previous error
{% endhighlight %}

As expected, there is no defined `println!` macro, because there is no implemented I/O and runtime, we’re on our own!
If we remove the `println!` call and try to compile again we’ll get another error:

{% highlight bash %}
$ rustc --target arm-unknown-linux-gnueabihf kernel.rs
error: language item required, but not found: `panic_fmt`
error: language item required, but not found: `eh_personality`
error: aborting due to 2 previous errors
{% endhighlight %}

This means that the minimal Rust runtime expects these two functions to be defined. Both functions are used by the failure mechanism/handling of the compiler. For a simple freestanding hello world, we can just define them as empty implementations:

{% highlight rust %}
// kernel.rs
#![feature(no_std, lang_items)]
#![crate_type = “staticlib”]
#![no_std]
pub extern fn main() {
 loop {}
}
#[lang = “eh_personality”]
extern fn eh_personality() {}
#[lang = “panic_fmt”]
extern fn panic_fmt() {}
{% endhighlight %}

The `#[lang]` attribute informs the compiler that this function is being defined for the runtime. Since this attribute is also feature gated, we need to add it to our `#![feature()]` attribute list. Now to compile, we also need to inform the compiler that we just want it to emit an object file. An object file is basically your compiled code with the defined symbols. We’re going to use the generated object file to link with `arm-none-eabi-gcc` and generate our `kernel.elf` file. We’ll also add the `-O` flag to remove some unnecessary symbols and do some optimizations.
We also changed our `main` function to be public and extern. This informs the compiler that our function will be used externally.
So now we run:
{% highlight sh %}
$ rustc — target arm-unknown-linux-gnueabihf -O — emit=obj kernel.rs
{% endhighlight %}
This will generate our `kernel.o` file. We can make sure our cross-compiling is working and emiting ARM compatible machine code using the `file` command:
{% highlight sh %}
$ file kernel.o
kernel.o: ELF 32-bit LSB relocatable, ARM, version 1 (GNU/Linux), not stripped
{% endhighlight %}
We can inspect our object file symbols using `arm-none-eabi-nm`:
{% highlight sh %}
$ arm-none-eabi-nm kernel.o
00000000 T _ZN4main20h0197e2cdb53c0194haaE
00000000 T rust_begin_unwind
00000000 T rust_eh_personality
{% endhighlight %}
This show us the three defined symbols by our source code. The weird one is our main function. This happens because of something called name mangling, which is something the compiler does to avoid name collisions. We want our symbols to be clear, so external tools knows how to reference to it. To tell the compiler to not mangle our main function, we need to:

{% highlight rust %}
// kernel.rs
#![feature(no_std, lang_items)]
#![crate_type = “staticlib”]
#![no_std]
#[no_mangle]
pub extern fn main() {
 loop {}
}
#[lang = “eh_personality”]
extern fn eh_personality() {}
#[lang = “panic_fmt”]
extern fn panic_fmt() {}
{% endhighlight %}
If we compile again and check our symbols we’ll see:
{% highlight sh %}
$ arm-none-eabi-nm kernel.o
00000000 T main
00000000 T rust_begin_unwind
00000000 T rust_eh_personality
{% endhighlight %}
They’re all clear! This is going to be useful when we need to call it from assembly.
Making the LED blink
Raspberry’s Pi B+ GPIO base address is `0x20200000`. The register that will allow us to turn our LED on is at the base address + `0x8` offset and we need to set the 15th bit to 1 to turn it on. To turn it off, we need to set the 15th bit to 1 on the base address + `0xB` offset. Between these changes, we need to block the program for some time. Our code will look like:

{% highlight rust %}
// kernel.rs
#![feature(no_std, lang_items, asm)]
#![crate_type = “staticlib”]
#![no_std]
const GPIO_BASE: u32 = 0x20200000;
fn sleep(value: u32) {
 for _ in 1..value {
 unsafe { asm!(“”); }
 }
}
#[no_mangle]
pub extern fn main() {
 let gpio = GPIO_BASE as *const u32;
 let led_on = unsafe { gpio.offset(8) as *mut u32 };
 let led_off = unsafe { gpio.offset(11) as *mut u32 };
loop {
 unsafe { *(led_on) = 1 << 15; }
 sleep(500000);
 unsafe { *(led_off) = 1 << 15; }
 sleep(500000);
 }
}
#[lang = “eh_personality”] extern fn eh_personality() {}
#[lang = “panic_fmt”] extern fn panic_fmt() {}
{% endhighlight %}

We added an `sleep` function which basically is an empty loop for a given range. The `unsafe { asm!(“”) }` part was the way I found to avoid the compiler to strip this part of the code out during the optimization phase, as it doesn’t seem to do anything. In the main function, we create a raw pointer to the `GPIO_BASE` address, then we get a mutable pointer to the locations we want to change, which are the one to turn the led on and to turn it off. Since the offset call involves dereferencing a raw pointer, its an unsafe operation, thus the `unsafe` blocks. With that we can just do an infinity loop that turns the led on, wait, turn it off and wait.
Preparing your code to run on the Raspberry PI
Now we can compile our code to run on our raspberry:
{% highlight sh %}
$ rustc — target arm-unknown-linux-gnueabihf -O — emit=obj kernel.rs
With our `kernel.o` file generated, we need to link our code using `arm-none-eabi-gcc`, so it can set our `main` symbol as the entry point for our program, generate our `kernel.elf` file and then use `arm-none-eabi-objcopy` to generate our final `kernel.img` binary:
$ arm-none-eabi-gcc -O0 -mfpu=vfp -mfloat-abi=hard -march=armv6zk -mtune=arm1176jzf-s -nostartfiles kernel.o -o kernel.elf
$ arm-none-eabi-objcopy kernel.elf -O binary kernel.img
{% endhighlight %}
With our `kernel.img` built, we just need to copy it to our Raspberry SD together with the other boot files (`start.elf` and `bootcode.bin`), stick our micro SD back into the Raspberry PI and turn it on to see our tiny kernel running and the LED blinking!
There is a lot of other cool stuff we can do, like using the system timer on chip to implement our `sleep` function or create some abstractions of top of the GPIO addresses to encapsulate all the `unsafe` calls in a more elegant API. But I think for now this is a good way to get started with bare metal programming with Rust.
