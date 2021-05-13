# Android Linker Entry

通过实际编译的库文件，来反推``linker entry``。

## app进程入口

众所周知，``android``的``app``进程，都是通过``zygote fork``出来的，而``zygote``进程的``exec``文件为（64位系统）：``app_process64``。
从``aosp``编译出的系统，通过``readelf``查看``app_process64``的``program headers``可见：

```bash
xuwakao@chiefhsing-PC:~/android-11.0.0_r29/out/target/product/crosshatch$ ~/Android/Sdk/ndk/20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/aarch64-linux-android/bin/readelf -l ./system/bin/app_process64

Elf file type is DYN (Shared object file)
Entry point 0x3000
There are 12 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  PHDR           0x0000000000000040 0x0000000000000040 0x0000000000000040
                 0x00000000000002a0 0x00000000000002a0  R      8
  INTERP         0x00000000000002e0 0x00000000000002e0 0x00000000000002e0
                 0x0000000000000015 0x0000000000000015  R      1
      [Requesting program interpreter: /system/bin/linker64]
  LOAD           0x0000000000000000 0x0000000000000000 0x0000000000000000
                 0x00000000000027c4 0x00000000000027c4  R      1000
  LOAD           0x0000000000003000 0x0000000000003000 0x0000000000003000
                 0x0000000000003260 0x0000000000003260  R E    1000
  LOAD           0x0000000000007000 0x0000000000007000 0x0000000000007000
                 0x00000000000005c0 0x00000000000005c0  RW     1000
```

从上面的信息（``[Requesting program interpreter: /system/bin/linker64]``）可知，``app_process64``在真正执行之前，先由``linux``去装载指定的``linker``，即``/system/bin/linker64``。

## linker64入口

从``aosp``编译出的系统，通过``readelf``查看``linker64``的``elf header``可见``entry point``头信息：

```bash
xuwakao@chiefhsing-PC:~/android-11.0.0_r29/out/target/product/crosshatch$ ~/Android/Sdk/ndk/20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/aarch64-linux-android/bin/readelf -h ./apex/com.android.runtime/bin/linker64
ELF Header:
  Magic:   7f 45 4c 46 02 01 01 00 00 00 00 00 00 00 00 00
  Class:                             ELF64
  Data:                              2's complement, little endian
  Version:                           1 (current)
  OS/ABI:                            UNIX - System V
  ABI Version:                       0
  Type:                              DYN (Shared object file)
  Machine:                           AArch64
  Version:                           0x1
  Entry point address:               0x4cbb0
  Start of program headers:          64 (bytes into file)
  Start of section headers:          1440184 (bytes into file)
  Flags:                             0x0
  Size of this header:               64 (bytes)
  Size of program headers:           56 (bytes)
  Number of program headers:         10
  Size of section headers:           64 (bytes)
  Number of section headers:         25
  Section header string table index: 23
```

``linker64``的``entry point``地址为``0x4cbb0``，查看一下该地址对应的地方：

```bash
xuwakao@chiefhsing-PC:~/android-11.0.0_r29/out/target/product/crosshatch$ ~/Android/Sdk/ndk/20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/aarch64-linux-android/bin/objdump -d ./apex/com.android.runtime/bin/linker64 | grep '4cbb0'
000000000004cbb0 <__dl__start>:
   4cbb0:       910003e0        mov     x0, sp
```

``dump``出该处符号的完整汇编代码如下：

```bash
000000000004cbb0 <__dl__start>:
   4cbb0:    910003e0     mov    x0, sp
   4cbb4:    9400cc09     bl    7fbd8 <__dl___linker_init>
   4cbb8:    d61f0000     br    x0
   4cbbc:    00000000     .inst    0x00000000 ; undefined
```

## __dl__start源码

 根据``bionic/linker/Android.bp``，可知``__dl_``为符号前缀：

```makefile
    // Insert an extra objcopy step to add prefix to symbols. This is needed to prevent gdb
    // looking up symbols in the linker by mistake.
    prefix_symbols: "__dl_",
```

所以，实际符号为``_start``，该符号定义在（``bionic/linker/arch/arm64/begin.S``）：

```assembly
ENTRY(_start)
  // Force unwinds to end in this function.
  .cfi_undefined x30

  mov x0, sp
  bl __linker_init

  /* linker init returns the _entry address in the main image */
  br x0
END(_start)

```

``_start``符号实际代表的就是，``linker``的入口。

## ld script

``_start``作为程序入口，其实是通过``gcc ld script``的``ENTRY``指定的（prebuilts/gcc/linux-x86/aarch64/aarch64-linux-android-4.9/aarch64-linux-android/lib/ldscripts/aarch64elf.x）。



