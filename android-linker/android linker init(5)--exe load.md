# Linker和Exe加载 : linker_main函数解释

``linker main``执行的任务前，在``linker``重定位，``libc``初始化，主线程初始化等一些列操作完成后，进程的用户空间已经具备动态链接的能力，开始可以动态链接exe相关的依赖了。

## linker main源码

```cpp
static ElfW(Addr) linker_main(KernelArgumentBlock& args, const char* exe_to_load) {
  ProtectedDataGuard guard;

#if TIMING
  struct timeval t0, t1;
  gettimeofday(&t0, 0);
#endif

  // Sanitize the environment.
  //主要是stdin等重定向到/dev/null，排除掉非法/不安全的环境变量
  __libc_init_AT_SECURE(args.envp);

  // Initialize system properties
  //系统属性初始化
  __system_properties_init(); // may use 'environ'

  // Register the debuggerd signal handler.
  linker_debuggerd_init();

  g_linker_logger.ResetState();

  // Get a few environment variables.
  const char* LD_DEBUG = getenv("LD_DEBUG");
  if (LD_DEBUG != nullptr) {
    g_ld_debug_verbosity = atoi(LD_DEBUG);
  }

#if defined(__LP64__)
  INFO("[ Android dynamic linker (64-bit) ]");
#else
  INFO("[ Android dynamic linker (32-bit) ]");
#endif

  // These should have been sanitized by __libc_init_AT_SECURE, but the test
  // doesn't cost us anything.
  const char* ldpath_env = nullptr;
  const char* ldpreload_env = nullptr;
  if (!getauxval(AT_SECURE)) {
    ldpath_env = getenv("LD_LIBRARY_PATH");
    if (ldpath_env != nullptr) {
      INFO("[ LD_LIBRARY_PATH set to \"%s\" ]", ldpath_env);
    }
    ldpreload_env = getenv("LD_PRELOAD");
    if (ldpreload_env != nullptr) {
      INFO("[ LD_PRELOAD set to \"%s\" ]", ldpreload_env);
    }
  }

  //动态加载exe elf，由于exe实际上已经被内核加载了，这里仅仅是读取elf的信息和mmap文件到用户空间，而没有加载exe的依赖和进行相关的重定位
  const ExecutableInfo exe_info = exe_to_load ? load_executable(exe_to_load) :
                                                get_executable_info();

  INFO("[ Linking executable \"%s\" ]", exe_info.path.c_str());

  //初始化代表exe elf的soinfo
  // Initialize the main exe's soinfo.
  soinfo* si = soinfo_alloc(&g_default_namespace,
                            exe_info.path.c_str(), &exe_info.file_stat,
                            0, RTLD_GLOBAL);
  somain = si;
  si->phdr = exe_info.phdr;
  si->phnum = exe_info.phdr_count;
  get_elf_base_from_phdr(si->phdr, si->phnum, &si->base, &si->load_bias);
  si->size = phdr_table_get_load_size(si->phdr, si->phnum);
  si->dynamic = nullptr;
  si->set_main_executable();
  init_link_map_head(*si);

  //设置exe bss的vma的名字
  set_bss_vma_name(si);

  //查找exe elf对应的.interp段（对于app_process64来说，就是linker64）
  // Use the executable's PT_INTERP string as the solinker filename in the
  // dynamic linker's module list. gdb reads both PT_INTERP and the module list,
  // and if the paths for the linker are different, gdb will report that the
  // PT_INTERP linker path was unloaded once the module list is initialized.
  // There are three situations to handle:
  //  - the APEX linker (/system/bin/linker[64] -> /apex/.../linker[64])
  //  - the ASAN linker (/system/bin/linker_asan[64] -> /apex/.../linker[64])
  //  - the bootstrap linker (/system/bin/bootstrap/linker[64])
  const char *interp = phdr_table_get_interpreter_name(somain->phdr, somain->phnum,
                                                       somain->load_bias);
  if (interp == nullptr) {
    // This case can happen if the linker attempts to execute itself
    // (e.g. "linker64 /system/bin/linker64").
    interp = kFallbackLinkerPath;
  }
  solinker->set_realpath(interp);
  init_link_map_head(*solinker);

  //gdb相关
  // Register the main executable and the linker upfront to have
  // gdb aware of them before loading the rest of the dependency
  // tree.
  //
  // gdb expects the linker to be in the debug shared object list.
  // Without this, gdb has trouble locating the linker's ".text"
  // and ".plt" sections. Gdb could also potentially use this to
  // relocate the offset of our exported 'rtld_db_dlactivity' symbol.
  //
  insert_link_map_into_debug_map(&si->link_map_head);
  insert_link_map_into_debug_map(&solinker->link_map_head);

  //vdso相关，暂且不表
  add_vdso();

  ElfW(Ehdr)* elf_hdr = reinterpret_cast<ElfW(Ehdr)*>(si->base);

  // We haven't supported non-PIE since Lollipop for security reasons.
  if (elf_hdr->e_type != ET_DYN) {
    // We don't use async_safe_fatal here because we don't want a tombstone:
    // even after several years we still find ourselves on app compatibility
    // investigations because some app's trying to launch an executable that
    // hasn't worked in at least three years, and we've "helpfully" dropped a
    // tombstone for them. The tombstone never provided any detail relevant to
    // fixing the problem anyway, and the utility of drawing extra attention
    // to the problem is non-existent at this late date.
    async_safe_format_fd(STDERR_FILENO,
                         "\"%s\": error: Android 5.0 and later only support "
                         "position-independent executables (-fPIE).\n",
                         g_argv[0]);
    _exit(EXIT_FAILURE);
  }

  // Use LD_LIBRARY_PATH and LD_PRELOAD (but only if we aren't setuid/setgid).
  parse_LD_LIBRARY_PATH(ldpath_env);//解释//LD_LIBRARY_PATH
  parse_LD_PRELOAD(ldpreload_env);//解释LD_PRELOAD

  //android namespace相关：参考：https://source.android.com/devices/architecture/vndk/linker-namespace
  std::vector<android_namespace_t*> namespaces = init_default_namespaces(exe_info.path.c_str());

  //exe文件解释.dynamic段
  if (!si->prelink_image()) __linker_cannot_link(g_argv[0]);

  // add somain to global group
  si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL);
  // ... and add it to all other linked namespaces
  for (auto linked_ns : namespaces) {
    if (linked_ns != &g_default_namespace) {
      linked_ns->add_soinfo(somain);
      somain->add_secondary_namespace(linked_ns);
    }
  }

  //设置exe相关的tls，使其注册到linker全局的g_tls_modules
  linker_setup_exe_static_tls(g_argv[0]);

  //加载所有的ld_preloads和依赖，link全部的so
  // Load ld_preloads and dependencies.
  std::vector<const char*> needed_library_name_list;
  size_t ld_preloads_count = 0;

  for (const auto& ld_preload_name : g_ld_preload_names) {
    needed_library_name_list.push_back(ld_preload_name.c_str());
    ++ld_preloads_count;
  }

  for_each_dt_needed(si, [&](const char* name) {
    needed_library_name_list.push_back(name);
  });

  const char** needed_library_names = &needed_library_name_list[0];
  size_t needed_libraries_count = needed_library_name_list.size();

  if (needed_libraries_count > 0 &&
      !find_libraries(&g_default_namespace,
                      si,
                      needed_library_names,
                      needed_libraries_count,
                      nullptr,
                      &g_ld_preloads,
                      ld_preloads_count,
                      RTLD_GLOBAL,
                      nullptr,
                      true /* add_as_children */,
                      true /* search_linked_namespaces */,
                      &namespaces)) {
    __linker_cannot_link(g_argv[0]);
  } else if (needed_libraries_count == 0) {
    if (!si->link_image(SymbolLookupList(si), si, nullptr, nullptr)) {
      __linker_cannot_link(g_argv[0]);
    }
    si->increment_ref_count();
  }

  //完成tls相关工作（内部知识round up一下offset）
  linker_finalize_static_tls();
  //完成main thread的初始化，参考前面文章
  __libc_init_main_thread_final();

  if (!get_cfi_shadow()->InitialLinkDone(solist)) __linker_cannot_link(g_argv[0]);

  //执行exe文件的DT_PREINIT_ARRAY、DT_INIT、DT_INIT_ARRAY段内容
  si->call_pre_init_constructors();
  si->call_constructors();

#if TIMING
  gettimeofday(&t1, nullptr);
  PRINT("LINKER TIME: %s: %d microseconds", g_argv[0],
        static_cast<int>(((static_cast<long long>(t1.tv_sec) * 1000000LL) +
                          static_cast<long long>(t1.tv_usec)) -
                         ((static_cast<long long>(t0.tv_sec) * 1000000LL) +
                          static_cast<long long>(t0.tv_usec))));
#endif
#if STATS
  print_linker_stats();
#endif
#if TIMING || STATS
  fflush(stdout);
#endif

  // We are about to hand control over to the executable loaded.  We don't want
  // to leave dirty pages behind unnecessarily.
  purge_unused_memory();

  //获取exe的入口函数，并返回
  ElfW(Addr) entry = exe_info.entry_point;
  TRACE("[ Ready to execute \"%s\" @ %p ]", si->get_realpath(), reinterpret_cast<void*>(entry));
  return entry;
}
```

``linker_main``主要作用就是，初始化系统属性，加载和重定位``exe``文件的``preloads``和依赖，并执行自身的初始化函数，并最终返回其入口函数。