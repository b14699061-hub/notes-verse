

```bash

pip install frida-tools

```


## frida server

frida vesion should be identical to the frida-server (or at least the same major version)

```bash

frida --version

```


we need to install one of the following:
1. frida-server-[VERSION]-android-arm64 for real phone (arm/arm64)
2. frida-server-[VERSION]-android-x86 (emulator)

plce frida server in /data/local/tmp

> Run

```bash

/data/local/tmp/frida-server-[VERSION]-[ARCH] -DP

```

1. -D - daemonize (detach from shell and become a daemon)
2. -P - disables preload, which disables preload optimization

and finally check that everything works:

```bash

frida-ps -Uai

```

> TODO: read about ptrace and how it is related to frida

## JS API

the api allows writing scripts in javascript that run using frida to modify and analyze running processes.

> running a script on a spawned app

1. java:
    * java.perform(fn: function)
    * java.use(className: string)
    * java.cast(handle: NativePointer, class: klass)
    * Hooking a function - function.implementation

---

- tips:
    * when trying to understand code flows in the app, it is useful to hook an interesting function and make it log it's backtrace (the function stack that lead to the current function - stack trace)

    * when hooking a function that has multiple overloads, the specific overload has to be specified.

    * its very important to remember - in order to access a java class instance`s field, always use .value

---

2. native backtrace:
    * memory.scan(address, size, pattern, callbacks)
    * memory.scanSync(address, size, pattern)
    * working with native pointer:
        a. readU16
        b. writeU16
        c. readPointer
        d. readByteArray

    * some usefull stuff:
        a. DebugSymbol.fromAddress(address)
        b. hexdump


## running on the device — Use an Android emulator

1. Create an AVD (Android Studio → AVD Manager). Prefer an **x86/x86_64** system image (emulators run x86 faster).
2. Start the emulator.
3. Make sure the emulator is rooted (stock emulator images are usually rootable). Check:

   ```bash
   adb root
   adb shell getprop ro.product.cpu.abi
   ```
4. Download the matching frida-server binary for the emulator ABI (e.g. `frida-server-*-android-x86_64` for x86_64).
5. Push & run it:

   ```bash
   adb push frida-server-<version>-android-<abi> /data/local/tmp/
   adb shell "chmod 755 /data/local/tmp/frida-server-<version>-android-<abi>"
   adb shell "/data/local/tmp/frida-server-<version>-android-<abi> &"
   ```
6. From your host, run Frida tools (they’ll connect to the running frida-server):
   e.g. `frida-ps -U` or just `frida-ps` — the daemon should be discoverable once frida-server is up.

7. killing the frida-server process on the emulator is best performed by:
    ```bash

    pidof frida-server...
    kill -9 [PID]

    ```

Notes: emulators are the typical way to run Frida when you don’t want/own a real device.

## frida:

1. when running frida-trace a __handlers__ directory is created with cached hooks for all the -i libraries and functions you traced.
you can edit these files for your own logic of hooking.

2. -S flag will both run your js code and trace the appliction (AMAZING).

## for logging tls keys (ssl key logging):
    codeshare.re/@k0nserv/tls-keylogger