# Android Linker Bootstrap

``linker``作为动态链接器，但本身也是一个共享库，那么它由谁来链接？

## 1. linker自举

``linker``的链接是由它本身完成的，称为自举（``bootstrap``）。
``linker``不能依赖其他共享库，其内部的全局和静态变量等的``relocation``必须由其自身完成。

## 2. 自举实现

``linker``的入口函数，即为其自举执行的入口，即``_start``：

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

``mov x0, sp`` ：以栈指针作为参数，借用x0寄存器传参给``__linker_init``，以便后续函数获取参数和环境变量等；

``bl __linker_init`` : 跳转到``_linker_init``函数，开始进行实际的初始化工作；

``br x0`` : 用于执行完``linker``初始化后，回到执行进程的实际入口；

## 3. linker初始化

### 3.1 Elf format

在开始读代码前，要牢牢记住``ELF``文件的格式和数据结构：

![ELF format](https://img-blog.csdnimg.cn/20190515230041942.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1RheWxvclBvdHRlcg==,size_16,color_FFFFFF,t_70)

### 3.2 linker执行前的堆栈

在``linker``实际进行初始化前，也就是在``linker elf``被``kernel``装载后，进程堆栈内已经存在一些必要的信息供后续``linker``使用，在此不展开详细讨论，只说明当前进程堆栈的情况（具体可以参照：）

![Init stack iamge](https://github.com/xuwakao/wakao-assets/blob/master/aosp/elf_init_stack_image.png?raw=true)

### 3.3 初始化执行

``__linker_init``实际进行``linker``的初始化，符号解释，重定位等一系列操作：

```cpp
/*
 * This is the entry point for the linker, called from begin.S. This
 * method is responsible for fixing the linker's own relocations, and
 * then calling __linker_init_post_relocation().
 *
 * Because this method is called before the linker has fixed it's own
 * relocations, any attempt to reference an extern variable, extern
 * function, or other GOT reference will generate a segfault.
 */
extern "C" ElfW(Addr) __linker_init(void* raw_args) {
  // Initialize TLS early so system calls and errno work.
  //初始化主线程一些信息，主要是tcb/tls
  KernelArgumentBlock args(raw_args);   //读取压入栈中的参数和环境变量，堆栈情况参看上面3.2图片
  bionic_tcb temp_tcb __attribute__((uninitialized));
  linker_memclr(&temp_tcb, sizeof(temp_tcb));
  __libc_init_main_thread_early(args, &temp_tcb);

  // When the linker is run by itself (rather than as an interpreter for
  // another program), AT_BASE is 0.
  ElfW(Addr) linker_addr = getauxval(AT_BASE);//通过辅助向量获取linker interpreter的装载地址，对于linker自举来说，linker_addr为0（linker没有interpreter），如果为0，则通过进程执行文件的program header获取基址
  if (linker_addr == 0) {
    // The AT_PHDR and AT_PHNUM aux values describe this linker instance, so use
    // the phdr to find the linker's base address.
    ElfW(Addr) load_bias;
    get_elf_base_from_phdr(
      reinterpret_cast<ElfW(Phdr)*>(getauxval(AT_PHDR)), getauxval(AT_PHNUM),
      &linker_addr, &load_bias);//计算linker elf的基址和load bias（对于linker来说，load bias为0）
  }

  ElfW(Ehdr)* elf_hdr = reinterpret_cast<ElfW(Ehdr)*>(linker_addr);//对于linker来说，linker_addr == elf_hdr
  ElfW(Phdr)* phdr = reinterpret_cast<ElfW(Phdr)*>(linker_addr + elf_hdr->e_phoff);//linker程序头表地址

  // string.h functions must not be used prior to calling the linker's ifunc resolvers.
  const ElfW(Addr) load_bias = get_elf_exec_load_bias(elf_hdr);//对于linker来说，load bias为0
  //与ifunc的重定位相关，现在只有string.h有相关的ifunc实现（获取IFUNC resolver的地址，重定位.rela.iplt），详细可以看：https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android-ifunc.md
  call_ifunc_resolvers(load_bias);

  soinfo tmp_linker_so(nullptr, nullptr, nullptr, 0, 0);

  tmp_linker_so.base = linker_addr;
  tmp_linker_so.size = phdr_table_get_load_size(phdr, elf_hdr->e_phnum);
  tmp_linker_so.load_bias = load_bias;
  tmp_linker_so.dynamic = nullptr;
  tmp_linker_so.phdr = phdr;
  tmp_linker_so.phnum = elf_hdr->e_phnum;
  tmp_linker_so.set_linker_flag();

  // Prelink the linker so we can access linker globals.
  //prelink_image的作用比较简单，就是解释.dynamic section，获取动态链接符号表位置、重定位表位置、so名字等等一些基础全局信息，方便后续使用
  if (!tmp_linker_so.prelink_image()) __linker_cannot_link(args.argv[0]);
  //link_image进行符号解释和重定位（代码太多，就不一一展开了，就是根据符号表，重定位表等等循环解释全部的符号，包括全局变量，函数）
  if (!tmp_linker_so.link_image(SymbolLookupList(&tmp_linker_so), &tmp_linker_so, nullptr, nullptr)) __linker_cannot_link(args.argv[0]);

  return __linker_init_post_relocation(args, tmp_linker_so);
}
```

整个初始化流程为：

> 1. 初始化``main thread``，主要是``TCB/TLS``。详细参考：[Linker和主线程初始化](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20linker%20init(3)--main%20thread%20init.md)。
>
> 2. 计算一些地址相关，``elf_hdr``，``phdr``等。
>
> 3. ``ifunc``解释。详细参考：[Android IFUNC支持](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android-ifunc.md)。
>
> 4. 执行``linker``的符号解释，重定位等工作。
>
> 5. ``linker``重定位完成后的，最终完成初始化``linker``，加载和解释``exe``等工作，并返回``exe main``。详细参考：[Linker重定位后初始化](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20linker%20init(4)--after%20relocation.md)。


