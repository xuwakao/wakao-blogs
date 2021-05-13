# Android IFUNC支持

## IFUNC概念

参考 [GNU_IFUNC](https://sourceware.org/glibc/wiki/GNU_IFUNC)。

## Android支持限制

在 [Android linker changes for NDK developers](https://android.googlesource.com/platform/bionic/+/master/android-changes-for-ndk-developers.md) 中描述，android api>=29才支持该特性：
>Use of IFUNC in libc (True for all API levels on devices running Q)
>>Starting with Android Q (API level 29), libc uses IFUNC functionality in the dynamic linker to choose optimized assembler routines at run time rather than at build time. This lets us use the same libc.so on all devices, and is similar to what other OSes already did. Because the zygote uses the C library, this decision is made long before we know what API level an app targets, so all code sees the new IFUNC-using C library. Most apps should be unaffected by this change, but apps that hook or try to detect hooking of C library functions might need to fix their code to cope with IFUNC relocations. The affected functions are from <string.h>, but may expand to include more functions (and more libraries) in future.

而且到目前为止，只有``<string.h>``的函数才使用了该特性。

这从``linker64 DSO``中就可以看到：

```bash
xuwakao@chiefhsing-PC:~/android-11.0.0_r29/out/target/product/crosshatch$ ~/Android/Sdk/ndk/20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/aarch64-linux-android/bin/readelf -s ./apex/com.android.runtime/bin/linker64 | grep 'IFUNC'
  6993: 00000000000bf404    12 IFUNC   LOCAL  DEFAULT   11 __dl_strchr
  6994: 00000000000bf410    12 IFUNC   LOCAL  DEFAULT   11 __dl_strcmp
  6995: 00000000000bf41c    12 IFUNC   LOCAL  DEFAULT   11 __dl_strlen
  6996: 00000000000bf3f8    12 IFUNC   LOCAL  DEFAULT   11 __dl_memchr
  6997: 00000000000bf428    12 IFUNC   LOCAL  DEFAULT   11 __dl_strncmp
  6998: 00000000000bf434    12 IFUNC   LOCAL  DEFAULT   11 __dl_strnlen

```

## IFUNC在Android Linker中的实现

### llvm与ifunc

``IFUNC``和``llvm``实现有关，在``llvm``的文档``llvm/docs/LangRef.rst``有关于``IFUNC``有一段描述：

>IFuncs
>>IFuncs, like as aliases, don't create any new data or func. They are just a new
>>symbol that dynamic linker resolves at runtime by calling a resolver function.
>>
>>IFuncs have a name and a resolver that is a function called by dynamic linker
>>that returns address of another function associated with the name.
>>
>>IFunc may have an optional :ref:`linkage type <linkage>` and an optional
>>:ref:`visibility style <visibility>`.
>>
>>Syntax::
>>
>>``@<Name> = [Linkage] [Visibility] ifunc <IFuncTy>, <ResolverTy>* @<Resolver>``

``ifunc``方法仅仅是一个``symbol``，``linker``最终会将其解释到一个``resolver``方法中去。具体是什么意思呢？以``strlen``为例子说明。

### 逆向分析

在``linker64 DSO``中，通过``readelf``查看，看到以下内容：

```shell
xuwakao@chiefhsing-PC:~/android-11.0.0_r29/out/target/product/crosshatch$ ~/Android/Sdk/ndk/20.0.5594570/toolchains/aarch64-linux-android-4.9/prebuilt/linux-x86_64/aarch64-linux-android/bin/readelf -s ./apex/com.android.runtime/bin/linker64 | grep 'strlen'
  6995: 00000000000bf41c    12 IFUNC   LOCAL  DEFAULT   11 __dl_strlen
  7002: 00000000000bf41c    12 FUNC    LOCAL  HIDDEN    11 __dl_strlen_resolver
  7071: 00000000000c05b0   320 FUNC    LOCAL  HIDDEN    11 __dl_strlen_default
  7905: 00000000000c9590    52 FUNC    LOCAL  DEFAULT   11 __dl___strlen_chk
```

从上可以看出，``__dl_strlen``和``__dl_strlen_resolver``两个符号，指向同一个地址：``00000000000bf41c``。该地址指向的是同一个方法：

```assembly

00000000000bf41c <__dl_strlen_resolver>:
   bf41c:	b0000000 	adrp	x0, c0000 <__dl_stpcpy+0x20> 
   bf420:	9116c000 	add	x0, x0, #0x5b0
   bf424:	d65f03c0 	ret

```

来详细看看``__dl_strlen_resolver``做了什么。
根据``adrp``指令的解释：![adrp instr](https://github.com/xuwakao/wakao-assets/blob/master/aosp/arm64_instr_ADRP.png?raw=true)。

```math

0xb0000000=1011 0000 0000 0000 0000 0000 0000 0000

immhi = 0000 0000 0000 0000 000
immlo = 01
immhi:immlo:Zeros(12) = 0 0000 0000 0000 0000 0001 0000 0000 0000

imm== SignExtend(immhi:immlo:Zeros(12), 64) = 0x1000

base= bf41c & 0xFF000=0xbf000

X[0] = base + imm = 0xc0000

0xc0000 + 0x5b0 = 0xc05b0

```

这三条指令实际的作用就是，进行一个相对寻址，最终返回``0xc05b0``这个地址，由调用者执行这个地址的代码。

``0xc05b0``指向的位置为：

```assembly

00000000000c05b0 <__dl_strlen_default>:
   c05b0:	92402c04 	and	x4, x0, #0xfff
   c05b4:	b200c3e8 	mov	x8, #0x101010101010101     	// #72340172838076673
   c05b8:	f13fc09f 	cmp	x4, #0xff0
   c05bc:	5400082c 	b.gt	c06c0 <__dl_strlen_default+0x110>
   c05c0:	a9400c02 	ldp	x2, x3, [x0]
   c05c4:	cb080044 	sub	x4, x2, x8
   c05c8:	b200d845 	orr	x5, x2, #0x7f7f7f7f7f7f7f7f
   c05cc:	cb080066 	sub	x6, x3, x8
   c05d0:	b200d867 	orr	x7, x3, #0x7f7f7f7f7f7f7f7f

```

**这个例子看到，``__dl_strlen_resolver``直接跳转到``__dl_strlen_default``，这是因为编译器优化的结果，实际上``__dl_strlen_resolver``的内部逻辑应该是，运行时根据程序定义的逻辑，选择不同的``strlen``实现。**

### ifunc函数定义

以``strlen``为例，该函数对应的``IFUNC``函数定义如下([源码](bionic/libc/arch-arm64/dynamic_function_dispatch.cpp))：

```cpp
typedef size_t strlen_func(const char*);
DEFINE_IFUNC_FOR(strlen) {
    if (supports_mte(arg->_hwcap2)) {
        RETURN_FUNC(strlen_func, strlen_mte);
    } else {
        RETURN_FUNC(strlen_func, strlen_default);
    }
};

#if defined(__aarch64__)
#define IFUNC_ARGS (uint64_t hwcap __attribute__((unused)), \
                    __ifunc_arg_t* arg __attribute__((unused)))
#elif defined(__arm__)
#define IFUNC_ARGS (unsigned long hwcap __attribute__((unused)))
#else
#define IFUNC_ARGS ()
#endif

#define DEFINE_IFUNC_FOR(name) \
    name##_func name __attribute__((ifunc(#name "_resolver"))); \
    __attribute__((visibility("hidden"))) \
    name##_func* name##_resolver IFUNC_ARGS

#define DECLARE_FUNC(type, name) \
    __attribute__((visibility("hidden"))) \
    type name

#define RETURN_FUNC(type, name) { \
        DECLARE_FUNC(type, name); \
        return name; \
    }
```

``__attribute__((ifunc))``就是定义一个``IFUNC``函数，宏展开后，其实就是定义了一个``strlen_resolver``函数，作用是，根据``supports_mte``的支持情况，选择不同的``strlen``实现。这里实现实际是：``strlen_default``，这恰好和前面逆向分析的一致。

## 结论

对于``__attribute__((ifunc))``定义的``IFUNC``函数，编译器最终会把该函数的符号指向一个变体的``resolver``函数，运行时动态选择对应的实现函数。这个好处是，希望所有设备使用同一个``so``文件，避免针对不同设备，要用不同选项的去编译出不同的``so``文件。
