# Modular Computing Platform (MCP)

It has nothing to do with "Model Context Protocol". I named this before
that acronym existed.

This is a large project that I will write more about when it's further along.
In the process of doing this project, I have created some things that have
great freestanding merit.

- [Forth](#Forth)
- [Beeper](#Beeper)


## Common Getting Started Instructions

These steps apply for all of the demos in this project.

1.  Install the
    [NuttX prerequisites](https://nuttx.apache.org/docs/latest/quickstart/install.html#prerequisites).

2.  Install `ccache` (`sudo apt install ccache`).

3.  Clone this repo and its nested submodules

    ```shell
    git clone https://github.com/liamHowatt/mcp_nuttxspace.git
    # the qemu submodule is not needed
    git submodule update --init --recursive nuttx apps mcp_apps mcp_boards
    ```

4.  ```shell
    cd nuttx
    ```

Continue following the steps for one of the demos below.


# Forth

## Preamble

Forth is a programming language from the 1970s which has some properties that allowed
me to make a programming option for NuttX which allows on-device editing with the
NuttX `vi` app, compiling to native machine code NuttX ELF files, and running.

-   *Easy FFI* - It calls C functions and creates C callbacks with no per-function
    wrappers.
-   *Simple compiler* - No AST. It compiles Forth "words" to snippets of CPU
    instructions or bytecode.
-   *Minimal runtime* - The runtime only sets up registers before jumping in and out
    of the compiled code.
-   *Fast* - Forth can be used to process data in tight loops that the programmer can
    anticipate will compile to only a handful of CPU instructions with no calls.
-   *Minimal runtime memory* - Memory is needed to load the ELF, and
    then a fixed 2 KB is enough for the two Forth stacks, the static variables, and
    the rest is used for the flexible Forth "data space".
-   *Multithreading* - There is a runtime library for spawning pthreads. (not available
    in the simulator due to no TLS)

To make FFI ABI predictable and easy, it's strictly a 32-bit Forth. In all
32-bit C ABIs I cared to review, I observed that pointers, all integer types
(less or equal 32 bits), and bools are always passed on the stack in 32-bit units.
As Forth does not have data types, this is critical.

The supported architectures are:

-   *Bytecode VM* - Bytecode executed by a software interpreter. Highly portable.
    Still requires a 32-bit environment though.
-   *x86-32* - To my knowledge, it works with any 32-bit x86 since Pentium Pro.
-   *ESP32S3* - It may work on ESP32 too. There are no ESP32S3 extensions being used.
    It cannot work on ESP8266 as it does not have register windows.

I plan to support the ARM Cortex-M0 instruction set next.

The NuttX loader will load the binary into a region of the address space that is
guaranteed to be executable on the target platform. On ESP32S3, this is critical,
as it is technically a Harvard architecture CPU. NuttX knows how to translate
the address for the memory region it loads the binary to.

Note that the NuttX ELFs produced are not executables that automatically do relocations.
The ELFs are loaded by `dlopen` in the mcp_forth app and then dependencies are resolved
from arrays of string-to-function pointer pairs, and could absolutely leverage the
NuttX embedded symbol table in the very near future.

Find more information about mcp_forth in the
[README for mcp_forth](https://github.com/liamHowatt/mcp_forth/blob/master/README.md).

## Usage

You can try out NuttX mcp_forth either with the NuttX simulator config or on an ESP32S3
on real hardware or in the qemu emulator.

Follow the [Common Getting Started Instructions](#Common-Getting-Started-Instructions) first.

### Simulator

64-bit Apple M processor Macs cannot run 32-bit programs so it cannot run this demo,
even the bytecode VM, because it assumes `sizeof(int) == sizeof(void *)`. You may use
the ESP32S3 Emulator in the next section, however.

-   If your PC is an x86 machine, you can use the mcp_forth x86-32 machine code backend.
    You will need to install NASM and 32-bit zlib for NuttX.

    ```shell
    dpkg --add-architecture i386
    sudo apt update
    sudo apt install nasm zlib1g-dev:i386
    ```

-   If your PC is not an x86 machine, it will at least need to be able to run NuttX in a
    32-bit mode to use the bytecode VM.

    Disable `CONFIG_MCP_APPS_MCP_FORTH_NATIVE_X86_32`.

    You may have to install the 32-bit version of zlib on an ARM machine too. I am not sure.

Build it.

```shell
./tools/configure.sh -l sim:mcp_forth
make -j$(nproc) nuttx
```

It mounts the host's `mcp_nuttxspace/nuttx` inside the NuttX simulator. Try running a
sample program from `../mcp_apps/mcp_forth/mcp_forth/forth_programs/simple/`.
Copy `hello.fs` to your `nuttx` dir so the simulator can access it.

```shell
cp ../mcp_apps/mcp_forth/mcp_forth/forth_programs/simple/hello.fs .
```

Now you can start NuttX. At the NSH prompt, run the forth program.

```
./nuttx
nsh> mcp_forth data/hello.fs
hello world
```

It runs it with the bytecode VM by default. The bytecode VM will prevent stack overflows
and "data space" `allot` exhaustion while the x86 mode will not detect these at all.
To compile and run it with x86 machine code, add the `-O` flag.

```
nsh> mcp_forth -O data/hello.fs
hello world
```

The only sign that it created, loaded, and deleted an ELF file is `/tmp/m4_dl_ctr` and
`/var/sem/m4_dl_ctr`. They are used to create unique temporary names for the ELF files
and atomically update the counter. This is useful for starting multiple mcp_forth tasks
concurrently. It's important to note, though, that that the simulator config only
supports one mcp_forth instance at a time creating C callbacks with `c'` because there
is no thread-local storage (TLS) available in the sim.

Try writing your own Forth program with `vi` in `/tmp` inside NuttX and run it!

Here is a simple Forth program that counts to 3.

```forth
4 1 do           \ Loop from 1 to 3.
    i . cr       \ Print the loop counter and a newline.
    i 3 < if     \ If the loop counter is less than 3,
        1000 ms  \ sleep for 1000 milliseconds.
    then
loop
." done" cr      \ Print "done" and a newline.
```

For a really amazing demo, try out the Conway's Game of Life program. You will need to
increase the size of the program memory in `../mcp_apps/mcp_forth/mcp_forth.c`. Change
`MEMORY_SIZE` from 2048 to 16384.
It receives the starting board through `stdin`.

```
nsh> mcp_forth forth_programs/life/life.fs < forth_programs/life/starting_board.txt
```

Tip: clean up the sim gcov files after quitting NuttX with
`rm $(find .. -name '*.gcno' -o -name '*.gcda')`.
The defconfig is based on the nsh defconfig and I didn't bother to disable gcov.

### ESP32S3

[Get the ESP32S3 Toolchain](https://nuttx.apache.org/docs/latest/platforms/xtensa/esp32s3/index.html#esp32-s3-toolchain).

Get esptool. It may just require `apt install python3-pip`, `pip install esptool`.
esptool is also required to create the qemu image.

```shell
./tools/configure.sh -l ../mcp_boards/mcp-computer-4/configs/mcp_forth
make -j$(nproc) nuttx
```

You can use it on a real ESP32S3 or in the qemu emulator. This config is intended
for use with the board ESP32S3-DevKit. Here I show how to use it with qemu.

Initialize the qemu submodule in this repo.

```shell
cd ..
git submodule update --init qemu
cd qemu
```

Ensure python3 venv is available (`sudo apt install python3-venv`).

Follow the steps in the official
[docs for compiling the Espressif fork of qemu](https://github.com/espressif/esp-toolchain-docs/blob/main/qemu/esp32s3/README.md).
The package `libgcrypt-devel` on Ubuntu is actually `libgcrypt-dev`.
Don't neglect to install the [general qemu prerequisites](https://wiki.qemu.org/Hosts/Linux).
Some of the "Recommended additional packages" are needed too. I installed all of them,
for good measure, in my NuttX development Docker container.

In summary, the full installation from that guide was the below, for me. I did not actually
need `sudo` in the Docker container.

```
sudo apt-get install python3-venv libgcrypt-dev \
    git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build \
    git-email \
    libaio-dev libbluetooth-dev libcapstone-dev libbrlapi-dev libbz2-dev \
    libcap-ng-dev libcurl4-gnutls-dev libgtk-3-dev \
    libibverbs-dev libjpeg8-dev libncurses5-dev libnuma-dev \
    librbd-dev librdmacm-dev \
    libsasl2-dev libsdl2-dev libseccomp-dev libsnappy-dev libssh-dev \
    libvde-dev libvdeplug-dev libvte-2.91-dev libxen-dev liblzo2-dev \
    valgrind xfslibs-dev \
    libnfs-dev libiscsi-dev
./configure --target-list=xtensa-softmmu \
    --enable-gcrypt \
    --enable-slirp \
    --enable-debug \
    --enable-sdl \
    --disable-strip --disable-user \
    --disable-capstone --disable-vnc \
    --disable-gtk
ninja -C build
```

Now that qemu is built, go back to your `nuttx` dir and run the nuttx binary on qemu.

```shell
cd ../nuttx
../qemu/build/qemu-system-xtensa -nographic -machine esp32s3 -drive file=nuttx.merged.bin,if=mtd,format=raw
```

Now at the NSH prompt, you can mount the Forth sample programs romdisk. Simply run `mcp_forth -m`
which will mount the romdisk and `/forth_programs/` will appear in the VFS.

```
nsh> ls
/:
 dev/
 proc/
 tmp/
nsh> mcp_forth -m
nsh> ls
/:
 dev/
 forth_programs/
 proc/
 tmp/
```

You can try out the hello world. Use the `-O` flag to compile to native ESP32S3 Xtensa
instructions.

```
nsh> mcp_forth forth_programs/simple/hello.fs
hello world
nsh> mcp_forth -O forth_programs/simple/hello.fs
hello world
```

Unlike in the simulator, thread-local storage (TLS) is available, so you can
use/write Forth programs that use multithreading. Try running `forth_programs/thread_test.fs`.

For a really amazing demo, try out the Conway's Game of Life program. You will need to
increase the size of the program memory in `../mcp_apps/mcp_forth/mcp_forth.c`. Increase
`MEMORY_SIZE` from 2048 to 16384.
It receives the starting board through `stdin`.

```
nsh> mcp_forth forth_programs/life/life.fs < forth_programs/life/starting_board.txt
```

`vi` is enabled in this defconfig so you can write Forth programs in `/tmp` and run them.

You can stop qemu from another terminal with `pkill qemu`.

qemu for ESP32S3 is flawed in that it is not aware of instruction/data bus
address space translations. The only region of the address space where instruction addresses
are equal to data addresses is the 8 KB RTC memory. Thus, `CONFIG_ESP32S3_RTC_HEAP` is enabled
in the config so that the emulator can work. If a Forth program binary ends up being bigger
than 8 KB, it will not be able to allocate room for it from the RTC heap and it will not
run. This is *not* a limitation on real ESP32S3 hardware.

# Beeper

## Preamble

This is a [Beeper](https://www.beeper.com/) client. You can think of it like the Beeper
Android or iOS app. This is a Beeper NuttX+LVGL app targeting ESP32S3 or similar.

It's also a Matrix client. It only uses
[standard Matrix specification APIs](https://spec.matrix.org/v1.16/client-server-api)
(with the exception of [MSC 2346](https://github.com/matrix-org/matrix-spec-proposals/blob/hs/msc-bridge-inf/proposals/2346-bridge-info-state-event.md))
and no Beeper extensions. However, it has only been tested with Beeper Matrix servers
and assumes certain things such as all chat rooms being group chats, even direct-message
chats, because that's how they are presented on bridge-based Matrix servers.

This may be the most versatile Matrix [client](https://matrix.org/ecosystem/clients/)
or even [SDK](https://matrix.org/ecosystem/sdks/) in existence. Being strictly C source
code (and C++ for libolm), it can be embedded anywhere. Even the simplest
Matrix C SDK, libcmatrix, has a hard dependency on GObject - a GNU/Linux thing.
Of course this project has hard dependencies (WolfSSL, two JSON libraries, LVGL) but
these may be swapped out easily. As this app changes and becomes more generic and
portable, an inviolable design goal will always be to keep the RAM usage low so it can
continue to work on MCUs like ESP32. One of the ways this is currently achieved is by
receiving `sync` responses incrementally. The initial sync is a 1 MB JSON on my account,
so parsing the entire JSON at once is not an option when the memory overhead of the
internal representation may exceed the 8 MB RAM of ESP32S3.

WARNING. This app cuts corners when validating received messages. There is a reliable
procedure for cryptographically validating sender identity which this app does not fully
implement yet.

WARNING. This app may have security vulnerabilities.

## Usage

I have it working in the simulator and on a custom board based on the ESP32S3-DevKit.
The defconfig for that is at `mcp_boards/mcp-computer-4/configs/mcp/defconfig`,
for reference. Note that it has a lot of extra functionality enabled which is not
strictly for the Beeper app.

Below is a guide for using it in the **simulator**.

Follow the [Common Getting Started Instructions](#Common-Getting-Started-Instructions) first.

It may only work with XOrg based desktops. I am not sure. I have only tried on an
Ubuntu 20.04 host where NuttX is running inside an Ubuntu 22.04 or 24.04 Docker container.
The Docker container is only able to open an XOrg window while running `./nuttx` from
a VS Code terminal which is attached to the container.

1.  Decide where you want Beeper app data to be persisted on your host PC.
    Open `../apps/mcp_apps/sim_lvgl/sim_lvgl.c` and change the `mount`
    invocation so that the `fs=` option is the path on your PC you want to use.
    From now on referred to as the hostfs path.

2.  `/dev/random` is needed for cryptography. I couldn't enable it in the sim, so instead I put
    a symlink to `/dev/random` in the hostfs path.

    ```
    cd /your/hostfs/path
    ln -s /dev/random
    ```

3. Create a directory called "appdata" which this app will assume the existence of.

    ```
    cd /your/hostfs/path
    mkdir appdata
    ```

4.  Open `../mcp_apps/beeper/beeper_private.h` and comment out the current `#define` of
    `BEEPER_RANDOM_PATH` and uncomment `#define BEEPER_RANDOM_PATH "/mnt/host/random"`.

5.  Now you need to patch the random path in WolfSSL. You need to run
    `configure.sh` and `make` first. **You will need to do this next step after
    every `make distclean` + configure**.

    Load the config and build NuttX once.

    ```shell
    ./tools/configure.sh -l sim:mcp_sim_lvgl
    make -j$(nproc) nuttx
    ```

    Now you can apply a patch to WolfSSL which changes the path to the `random` device
    it uses.

    ```shell
    cd ../apps/crypto/wolfssl/wolfssl
    git apply ../host_random.patch
    ```

6.  Build NuttX again and run it.

    ```shell
    make -j$(nproc) nuttx
    ./nuttx
    ```

    If you get the below XOrg error, simply try run `./nuttx` again. It's a
    random failure at launch that I don't fully understand.

    ```
    X Error of failed request:  BadValue (integer parameter out of range for operation)
      Major opcode of failed request:  130 (MIT-SHM)
      Minor opcode of failed request:  3 (X_ShmPutImage)
      Value in failed request:  0x280
      Serial number of failed request:  29
      Current serial number in output stream:  30
    ```

7.  Click "Beeper".

8.  Log in to your Beeper or Matrix account (not tested with any other Matrix servers
    than Beeper). There's an issue where if you type too fast, keypresses will be missed.
    Beware of this while typing your password which is partially hidden.

    Your username and password will be persisted in a file in your hostfs path at
    `/your/hostfs/path/appdata/beeper/login` and a directory for the username you signed
    in with will be created at `/your/hostfs/path/appdata/beeper/your-username`.
    You can delete `login` to effectively sign out while preserving the local data
    for that account until you sign into it again.

9.  Verify the device with SAS authentication. Initiate verification from another Matrix
    device. At the time of writing, this app cannot initiate verification. Follow the
    instructions on screen. It will show seven emojis and ask you to compare them with
    the ones shown on your other device.

    The official Beeper mobile app cannot initiate verification so you must use one of
    the other Beeper clients like the official desktop app or I recommend using the
    Element Web generic Matrix client which can initiate verification and works from a
    web browser.

10. View and send messages. The UI is made to work with mouse or keyboard navigation.
    Use the arrow keys, the enter key, and the tab key to change focus.

    After verifying the device, it may take some time for trust in your device to
    propagate across the Matrix network. It may even be worthwhile to wait half a day
    before concluding that something is wrong with sending messages. In my experience,
    the Facebook Messenger bridgebot never sent the app room keys to decrypt messages
    older than when this device account joined the network. You should carefully manage
    your hostfs appdata directory if you want to continue to view previous messages.

11. If you close the window and the process doesn't stop, use `pkill nuttx`
    in a separate terminal.
