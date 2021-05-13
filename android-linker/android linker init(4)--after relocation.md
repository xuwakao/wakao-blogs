# Linker重定位后初始化

``linker``重定位后，进行一系列初始化工作，这个阶段，``linker``已经可以引用外部的符号和全局变量了。

## 源码

```cpp
/*
 * This code is called after the linker has linked itself and fixed its own
 * GOT. It is safe to make references to externs and other non-local data at
 * this point. The compiler sometimes moves GOT references earlier in a
 * function, so avoid inlining this function (http://b/80503879).
 */
static ElfW(Addr) __attribute__((noinline))
__linker_init_post_relocation(KernelArgumentBlock& args, soinfo& tmp_linker_so) {
  // Finish initializing the main thread.
  __libc_init_main_thread_late();//继续main thread初始化

  //设置gnu_relro段为read-only
  // We didn't protect the linker's RELRO pages in link_image because we
  // couldn't make system calls on x86 at that point, but we can now...
  if (!tmp_linker_so.protect_relro()) __linker_cannot_link(args.argv[0]);

  // And we can set VMA name for the bss section now
  set_bss_vma_name(&tmp_linker_so);//为bss段的vma设置名字

  // Initialize the linker's static libc's globals
  __libc_init_globals();//调用libc.so的external函数，对libc一些全局变量进行初始化,主要是vdso的初始化

  // Initialize the linker's own global variables
  tmp_linker_so.call_constructors();//linker DSO内部的初始化，执行.init，.init_array等section的初始化函数

  // When the linker is run directly rather than acting as PT_INTERP, parse
  // arguments and determine the executable to load. When it's instead acting
  // as PT_INTERP, AT_ENTRY will refer to the loaded executable rather than the
  // linker's _start.
  const char* exe_to_load = nullptr;
  if (getauxval(AT_ENTRY) == reinterpret_cast<uintptr_t>(&_start)) {
    if (args.argc == 3 && !strcmp(args.argv[1], "--list")) {
      // We're being asked to behave like ldd(1).
      g_is_ldd = true;
      exe_to_load = args.argv[2];
    } else if (args.argc <= 1 || !strcmp(args.argv[1], "--help")) {
      async_safe_format_fd(STDOUT_FILENO,
         "Usage: %s [--list] PROGRAM [ARGS-FOR-PROGRAM...]\n"
         "       %s [--list] path.zip!/PROGRAM [ARGS-FOR-PROGRAM...]\n"
         "\n"
         "A helper program for linking dynamic executables. Typically, the kernel loads\n"
         "this program because it's the PT_INTERP of a dynamic executable.\n"
         "\n"
         "This program can also be run directly to load and run a dynamic executable. The\n"
         "executable can be inside a zip file if it's stored uncompressed and at a\n"
         "page-aligned offset.\n"
         "\n"
         "The --list option gives behavior equivalent to ldd(1) on other systems.\n",
         args.argv[0], args.argv[0]);
      _exit(EXIT_SUCCESS);
    } else {
      exe_to_load = args.argv[1];//执行文件的名字，例如app_process64
      __libc_shared_globals()->initial_linker_arg_count = 1;
    }
  }

  // store argc/argv/envp to use them for calling constructors
  g_argc = args.argc - __libc_shared_globals()->initial_linker_arg_count;
  g_argv = args.argv + __libc_shared_globals()->initial_linker_arg_count;
  g_envp = args.envp;
  __libc_shared_globals()->init_progname = g_argv[0];//执行文件的名字，例如app_process64

  // Initialize static variables. Note that in order to
  // get correct libdl_info we need to call constructors
  // before get_libdl_info().
  sonext = solist = solinker = get_libdl_info(tmp_linker_so);//初始化进程的全局dl info
  g_default_namespace.add_soinfo(solinker);

  ElfW(Addr) start_address = linker_main(args, exe_to_load);//对进程的可执行文件进行一系列操作（例如：加载exe到用户空间，获取exe phdr， 对exe进行符号解释和重定位，初始化环境变量，系统属性等），然后返回可执行文件的入口地址，一般情况下，就是main函数

  if (g_is_ldd) _exit(EXIT_SUCCESS);

  INFO("[ Jumping to _start (%p)... ]", reinterpret_cast<void*>(start_address));

  // Return the address that the calling assembly stub should jump to.
  return start_address;
}
```

``__linker_init_post_relocation``函数，主要完成下列工作：

>1. 继续完成``main thread``的部分初始化工作；（详细参考：[Linker和主线程初始化](xxxx)）
>2. 完成``libc DSO``全局变量的初始化；
>3. 完成``linker``的内部的初始化；
>4. 执行``linker_main``，完成一系列，包括：属性初始化，exe加载、加载依赖，符号解释，重定位等等；（详细参考：[Linker和Exe加载 : linker_main函数解释](xxxx)）


