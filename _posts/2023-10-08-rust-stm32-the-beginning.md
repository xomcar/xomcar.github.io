---
layout: post
title: Using Rust for development on STM32 platforms
lead: Working on ARM Macs!
---

It has come to my attention that the [Rust] programming language is trying to catch up as the new industry stardard for developing low level software as a modern equivalent to the *good old* C programming language, sponsoring many QoL changes.
This morning I got tired of reading yet another Linkedin post regarding this trend, so I tried to jumpstart myself into the topic.

# The objective
The idea was that of being able to setup a simple [Rust] application to run on an STM32 processor.
What I got to at the end is the ability to integrate such application with building/debugging capabilities fully integrated in [Visual Studio Code].

# The equipment
My host machine is Macbook Air M2 (ARM-based platform), and I will be using a Nucleo-64 [STM32L476] development board I borrowed from work.

# Documentation
First and foremost I checkout out the documentation sources about the topic available on the internet.
Luckily, there are a series of very well written books about the issue which one can access from [Rust-embedded book]. The books are mainly based on other board models but can easily be ported.
Secondly, it's always a good idea to have the [reference manual for the board] close at hand.

# Setting up the necessary tools
Theoretically, all that is needed is a [Rust] installation, however some more tooling is needed in order to setup debug and flashing properly.
First, the target architecture toolchain must be made available. In my case, the board has a Cortex M4 with integrated floating point computation: 
```shell
rustup target add thumbv7em-none-eabihf

```
\\
In order to interface gdb with the ST-Link, [openocd] is needed. 
Having brew installed this is an easy feat.
Note that I am using the latest version available: your mileage may vary.
```shell
brew install openocd --HEAD

```
\\
Thankfully [gdb] is finally available on ARM Mac architecture, to install the toolchain:
```shell
brew install armmbed/formulae/arm-none-eabi-gcc

```
\\
As a bonus, one can install the [cargo-generate] utility to jumpstart a project without much hassle
```shell
cargo install cargo-generate

```
<br>


# Setting up the project space
Using [cargo-generate] this is almost trivial. Go to your projects folder in you terminal and type:
```shell
cargo generate --git https://github.com/rust-embedded/cortex-m-quickstart

```
\\
Follow the prompt to give a proper name to your project. 
I choose ```l4760-demo```.
To view and edit the project metadata properties, use the ```Cargo.toml``` file.

# Setting up build, debug and deploy
The project quickstart created a lot of userful configuration for us.
First, we configure the memory layout of the board for the linker in the ```memory.x``` file.
The values for my board are as follows, refer to the manual for yours:
```shell
FLASH : ORIGIN = 0x08000000, LENGTH = 1024K
RAM : ORIGIN = 0x20000000, LENGTH = 96K

```
\\
Proceeding to the configuration of `openocd.cfg`:
```shell
source [find interface/stlink.cfg]

source [find target/stm32l4x.cfg]

```
\\
In the case you want to use a cargo runner with gdb instead of visual studio to debug, you can configure `openocd.gdb`. The default values where fine for me.

For configuring the cargo build and cargo run commands, open ```.cargo/config.toml```
and uncomment the proper setup. For my configuration:
```shell
runner = "arm-none-eabi-gdb -q -x openocd.gdb"

target = "thumbv7em-none-eabihf"

```
<br>

Let's try quickly if the setup is working before setting up the visual studio integration.
Edit ```src/main.rs``` with the following:
```rust
#![no_main]
#![no_std]

use panic_halt as _;

use cortex_m_rt::entry;
use cortex_m_semihosting::hprintln;

#[entry]
fn main() -> ! {
    hprintln!("All your codebase are belong to us").unwrap();

    loop {}
}

```
\\
This simple program will print a string on the host console using semihosting.
For the specifics on the meaning of all the directives and imports, I point you to the
[Rust-embedded book].

Now plug the board to the computer and run openocd:
```shell
openocd

```
<br>

Open another terminal window and use cargo run:
```shell
cargo run

``` 
<br>

The output should stop at the default Reset halt:
```shell
0x08000402 in cortex_m_rt::Reset () at src/lib.rs:497
497     pub unsafe extern "C" fn Reset() -> ! {

```
<br>

Enter `c` or `continue` to proceed to `main` entry-point
```
Breakpoint 4, l476_demo::__cortex_m_rt_main_trampoline () at src/main.rs:13
13      #[entry]

```

<br>

In the case the name of function are mangled, shown as `???`, or if the terminal hangs after continuing, check if the `memory.x` file is configured correctly. When you edit `memory.x`, remember to also run a `cargo clean` since the file is not monitored for incremental builds.

If all is well, by continuing, the terminal where openocd is open shall now show `All your codebase are belong to us`.\\
**Welcome to the embedded rust world!**

# Extra: setting up visual studio debugging
Add the following configuration to the `.vscode/tasks.json` file
```json
{
    "type": "cortex-debug",
    "request": "launch",
    "name": "Debug (OpenOCD)",
    "servertype": "openocd",
    "cwd": "${workspaceRoot}",
    "preLaunchTask": "Cargo Build (debug)",
    "runToEntryPoint": "main",
    "preLaunchCommands": [
        "load",
        "monitor arm semihosting enable"
    ],
    "executable": "./target/thumbv7em-none-eabihf/debug/l476-demo",
    "device": "STM32L476VGT6",
    "configFiles": [
        "interface/stlink.cfg",
        "target/stm32l4x.cfg"
    ],
}

```
<br>

This should be it. If you proceed run the debug configuration of vscode, you shall observe the
debug prints on the `gdb-server` terminal which just spawned. Breaking points, variables, watch, all should be fine and dandy and working flawlessly.

[Rust]: https://www.rust-lang.org/
[Visual Studio Code]: https://code.visualstudio.com/
[STM32L476]: https://www.st.com/en/microcontrollers-microprocessors/stm32l476rg.html
[Rust-embedded book]: https://docs.rust-embedded.org/book/
[reference manual for the board]: https://www.st.com/resource/en/reference_manual/rm0351-stm32l47xxx-stm32l48xxx-stm32l49xxx-and-stm32l4axxx-advanced-armbased-32bit-mcus-stmicroelectronics.pdf
[openocd]: https://openocd.org/
[gdb]: https://www.sourceware.org/gdb/
[cargo-generate]: https://github.com/cargo-generate/cargo-generate