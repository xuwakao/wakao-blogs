# Linker dlopen

``dlopen``的定义：

``` cpp (bionic/libc/include/dlfcn.h)
void* dlopen(const char* __filename, int __flag);
```

```cpp (bionic/libdl/libdl.cpp)
__attribute__((__weak__))
void* dlopen(const char* filename, int flag) {
  const void* caller_addr = __builtin_return_address(0);
  return __loader_dlopen(filename, flag, caller_addr);
}
```

```cpp (bionic/linker/dlfcn.cpp)
void* __loader_dlopen(const char* filename, int flags, const void* caller_addr) {
  return dlopen_ext(filename, flags, nullptr, caller_addr);
}

static void* dlopen_ext(const char* filename,
                        int flags,
                        const android_dlextinfo* extinfo,
                        const void* caller_addr) {
  ScopedPthreadMutexLocker locker(&g_dl_mutex);
  g_linker_logger.ResetState();
  void* result = do_dlopen(filename, flags, extinfo, caller_addr);
  if (result == nullptr) {
    __bionic_format_dlerror("dlopen failed", linker_get_error_buffer());
    return nullptr;
  }
  return result;
}

```

``do_dlopen``是实际执行装载任务的关键函数。

## do_dlopen ：link & load

```cpp
void* do_dlopen(const char* name, int flags,
                const android_dlextinfo* extinfo,
                const void* caller_addr) {
  std::string trace_prefix = std::string("dlopen: ") + (name == nullptr ? "(nullptr)" : name);
  ScopedTrace trace(trace_prefix.c_str());
  ScopedTrace loading_trace((trace_prefix + " - loading and linking").c_str());
  soinfo* const caller = find_containing_library(caller_addr); // 获取caller的soinfo
  android_namespace_t* ns = get_caller_namespace(caller); // 获取caller所属的namespace

  //省略掉不重要部分
  //.............................

  // End Workaround for dlopen(/system/lib/<soname>) when .so is in /apex.

  std::string asan_name_holder;

  const char* translated_name = name;
  if (g_is_asan && translated_name != nullptr && translated_name[0] == '/') {
    char original_path[PATH_MAX];
    if (realpath(name, original_path) != nullptr) {
      asan_name_holder = std::string(kAsanLibDirPrefix) + original_path;
      if (file_exists(asan_name_holder.c_str())) {
        soinfo* si = nullptr;
        if (find_loaded_library_by_realpath(ns, original_path, true, &si)) {
          PRINT("linker_asan dlopen NOT translating \"%s\" -> \"%s\": library already loaded", name,
                asan_name_holder.c_str());
        } else {
          PRINT("linker_asan dlopen translating \"%s\" -> \"%s\"", name, translated_name);
          translated_name = asan_name_holder.c_str();
        }
      }
    }
  }

  ProtectedDataGuard guard;
  soinfo* si = find_library(ns, translated_name, flags, extinfo, caller); 
  loading_trace.End();

  if (si != nullptr) {
    void* handle = si->to_handle();
    LD_LOG(kLogDlopen,
           "... dlopen calling constructors: realpath=\"%s\", soname=\"%s\", handle=%p",
           si->get_realpath(), si->get_soname(), handle);
    si->call_constructors();
    failure_guard.Disable();
    LD_LOG(kLogDlopen,
           "... dlopen successful: realpath=\"%s\", soname=\"%s\", handle=%p",
           si->get_realpath(), si->get_soname(), handle);
    return handle;
  }

  return nullptr;
}

```

关键在``find_library``中：

```cpp （bionic/linker/linker.cpp）
static soinfo* find_library(android_namespace_t* ns,
                            const char* name, int rtld_flags,
                            const android_dlextinfo* extinfo,
                            soinfo* needed_by) {
  soinfo* si = nullptr;

  if (name == nullptr) {
    si = solist_get_somain(); //exe执行文件的情况，此处忽略
  } else if (!find_libraries(ns,
                             needed_by,
                             &name,
                             1,
                             &si,
                             nullptr,
                             0,
                             rtld_flags,
                             extinfo,
                             false /* add_as_children */,
                             true /* search_linked_namespaces */)) {
    if (si != nullptr) {
      soinfo_unload(si);
    }
    return nullptr;
  }

  si->increment_ref_count();

  return si;
}

// add_as_children - add first-level loaded libraries (i.e. library_names[], but
// not their transitive dependencies) as children of the start_with library.
// This is false when find_libraries is called for dlopen(), when newly loaded
// libraries must form a disjoint tree.
bool find_libraries(android_namespace_t* ns,
                    soinfo* start_with,
                    const char* const library_names[],
                    size_t library_names_count,
                    soinfo* soinfos[],
                    std::vector<soinfo*>* ld_preloads,
                    size_t ld_preloads_count,
                    int rtld_flags,
                    const android_dlextinfo* extinfo,
                    bool add_as_children,
                    bool search_linked_namespaces,
                    std::vector<android_namespace_t*>* namespaces) {
  // Step 0: prepare.
  std::unordered_map<const soinfo*, ElfReader> readers_map;//elf reader map
  LoadTaskList load_tasks;//所有需要装载的so的vector

  for (size_t i = 0; i < library_names_count; ++i) {
    const char* name = library_names[i];
    load_tasks.push_back(LoadTask::create(name, start_with, ns, &readers_map)); //添加当前dlopen指定的so文件
  }

  // If soinfos array is null allocate one on stack.
  // The array is needed in case of failure; for example
  // when library_names[] = {libone.so, libtwo.so} and libone.so
  // is loaded correctly but libtwo.so failed for some reason.
  // In this case libone.so should be unloaded on return.
  // See also implementation of failure_guard below.
  //在stack上分配一段空间，存储当前需要装载的soinfo，用于后续如果加载失败的处理
  if (soinfos == nullptr) {
    size_t soinfos_size = sizeof(soinfo*)*library_names_count;
    soinfos = reinterpret_cast<soinfo**>(alloca(soinfos_size));
    memset(soinfos, 0, soinfos_size);
  }

  // list of libraries to link - see step 2.
  size_t soinfos_count = 0;

  auto scope_guard = android::base::make_scope_guard([&]() {
    for (LoadTask* t : load_tasks) {
      LoadTask::deleter(t);
    }
  });

  ZipArchiveCache zip_archive_cache;

  // Step 1: expand the list of load_tasks to include
  // all DT_NEEDED libraries (do not load them just yet)
  for (size_t i = 0; i<load_tasks.size(); ++i) {
    LoadTask* task = load_tasks[i];
    soinfo* needed_by = task->get_needed_by();

    bool is_dt_needed = needed_by != nullptr && (needed_by != start_with || add_as_children);//装载任务是否是DT_NEEDED，对于dlopen来说，为false（needed_by == start_with）
    task->set_extinfo(is_dt_needed ? nullptr : extinfo);
    task->set_dt_needed(is_dt_needed);

    LD_LOG(kLogDlopen, "find_libraries(ns=%s): task=%s, is_dt_needed=%d", ns->get_name(),
           task->get_name(), is_dt_needed);

    // Note: start from the namespace that is stored in the LoadTask. This namespace
    // is different from the current namespace when the LoadTask is for a transitive
    // dependency and the lib that created the LoadTask is not found in the
    // current namespace but in one of the linked namespace.
    //查找当前so依赖的所有DT_NEEDED。并添加到待装载列表
    //
    //【Load四部曲1】从elf文件中读取elf header，program headers，section headers，.dynamic section等
    if (!find_library_internal(const_cast<android_namespace_t*>(task->get_start_from()),
                               task,
                               &zip_archive_cache,
                               &load_tasks,
                               rtld_flags,
                               search_linked_namespaces || is_dt_needed)) {
      return false;
    }

    soinfo* si = task->get_soinfo();

    if (is_dt_needed) {
      needed_by->add_child(si);
    }

    //如果存在ld_preloads，那么前面的ld_preloads_count个soinfo，都应该是ld_preloads指定的library，这种情况只会出现在linker_main中，对于dlopen来说ld_preloads为null
    // When ld_preloads is not null, the first
    // ld_preloads_count libs are in fact ld_preloads.
    if (ld_preloads != nullptr && soinfos_count < ld_preloads_count) {
      ld_preloads->push_back(si);
    }

    if (soinfos_count < library_names_count) {
      soinfos[soinfos_count++] = si;
    }
  }

  // Step 2: Load libraries in random order (see b/24047022)
  LoadTaskList load_list;
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    auto pred = [&](const LoadTask* t) {
      return t->get_soinfo() == si;
    };

    if (!si->is_linked() &&
        std::find_if(load_list.begin(), load_list.end(), pred) == load_list.end() ) {//过滤掉已经link的soinfo
      load_list.push_back(task);
    }
  }

  //除非指定了，ANDROID_DLEXT_RESERVED_ADDRESS_RECURSIVE，否则so的加载顺序会是随机的顺序，而不是按DT_NEEDED指定的顺序，主要是安全的原因
  bool reserved_address_recursive = false;
  if (extinfo) {
    reserved_address_recursive = extinfo->flags & ANDROID_DLEXT_RESERVED_ADDRESS_RECURSIVE;
  }
  if (!reserved_address_recursive) {
    // Shuffle the load order in the normal case, but not if we are loading all
    // the libraries to a reserved address range.
    shuffle(&load_list);
  }

  // Set up address space parameters.
  address_space_params extinfo_params, default_params;
  size_t relro_fd_offset = 0;
  if (extinfo) {
    if (extinfo->flags & ANDROID_DLEXT_RESERVED_ADDRESS) {
      extinfo_params.start_addr = extinfo->reserved_addr;
      extinfo_params.reserved_size = extinfo->reserved_size;
      extinfo_params.must_use_address = true;
    } else if (extinfo->flags & ANDROID_DLEXT_RESERVED_ADDRESS_HINT) {
      extinfo_params.start_addr = extinfo->reserved_addr;
      extinfo_params.reserved_size = extinfo->reserved_size;
    }
  }

  //【Load四部曲2】为segments分配内存空间，并加载segments到内存，并确定base, load bias, phdr_addr等信息
  for (auto&& task : load_list) {
    address_space_params* address_space =
        (reserved_address_recursive || !task->is_dt_needed()) ? &extinfo_params : &default_params;
    if (!task->load(address_space)) {
      return false;
    }
  }

  // Step 3: pre-link all DT_NEEDED libraries in breadth first order.
  //【Load四部曲3】prelink image : 详细读取.dynamic section信息
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    if (!si->is_linked() && !si->prelink_image()) {
      return false;
    }
    register_soinfo_tls(si);
  }

  // Step 4: Construct the global group. Note: DF_1_GLOBAL bit of a library is
  // determined at step 3.

  // Step 4-1: DF_1_GLOBAL bit is force set for LD_PRELOADed libs because they
  // must be added to the global group
  //ld_preloads的库需要添加到全部的linked namespaces中去
  if (ld_preloads != nullptr) {
    for (auto&& si : *ld_preloads) {
      si->set_dt_flags_1(si->get_dt_flags_1() | DF_1_GLOBAL);
    }
  }

  // Step 4-2: Gather all DF_1_GLOBAL libs which were newly loaded during this
  // run. These will be the new member of the global group
  soinfo_list_t new_global_group_members;
  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    if (!si->is_linked() && (si->get_dt_flags_1() & DF_1_GLOBAL) != 0) {
      new_global_group_members.push_back(si);
    }
  }

  // Step 4-3: Add the new global group members to all the linked namespaces
  if (namespaces != nullptr) {
    for (auto linked_ns : *namespaces) {
      for (auto si : new_global_group_members) {
        if (si->get_primary_namespace() != linked_ns) {
          linked_ns->add_soinfo(si);
          si->add_secondary_namespace(linked_ns);
        }
      }
    }
  }

  // Step 5: Collect roots of local_groups.
  // Whenever needed_by->si link crosses a namespace boundary it forms its own local_group.
  // Here we collect new roots to link them separately later on. Note that we need to avoid
  // collecting duplicates. Also the order is important. They need to be linked in the same
  // BFS order we link individual libraries.
  std::vector<soinfo*> local_group_roots;
  if (start_with != nullptr && add_as_children) {
    local_group_roots.push_back(start_with);
  } else {
    CHECK(soinfos_count == 1);
    local_group_roots.push_back(soinfos[0]);
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    soinfo* needed_by = task->get_needed_by();
    bool is_dt_needed = needed_by != nullptr && (needed_by != start_with || add_as_children);
    android_namespace_t* needed_by_ns =
        is_dt_needed ? needed_by->get_primary_namespace() : ns;

    if (!si->is_linked() && si->get_primary_namespace() != needed_by_ns) {
      auto it = std::find(local_group_roots.begin(), local_group_roots.end(), si);
      LD_LOG(kLogDlopen,
             "Crossing namespace boundary (si=%s@%p, si_ns=%s@%p, needed_by=%s@%p, ns=%s@%p, needed_by_ns=%s@%p) adding to local_group_roots: %s",
             si->get_realpath(),
             si,
             si->get_primary_namespace()->get_name(),
             si->get_primary_namespace(),
             needed_by == nullptr ? "(nullptr)" : needed_by->get_realpath(),
             needed_by,
             ns->get_name(),
             ns,
             needed_by_ns->get_name(),
             needed_by_ns,
             it == local_group_roots.end() ? "yes" : "no");

      if (it == local_group_roots.end()) {
        local_group_roots.push_back(si);
      }
    }
  }

  // Step 6: Link all local groups
  for (auto root : local_group_roots) {
    soinfo_list_t local_group;
    android_namespace_t* local_group_ns = root->get_primary_namespace();

    walk_dependencies_tree(root,
      [&] (soinfo* si) {
        if (local_group_ns->is_accessible(si)) {
          local_group.push_back(si);
          return kWalkContinue;
        } else {
          return kWalkSkip;
        }
      });

    soinfo_list_t global_group = local_group_ns->get_global_group();
    SymbolLookupList lookup_list(global_group, local_group);
    soinfo* local_group_root = local_group.front();

    bool linked = local_group.visit([&](soinfo* si) {
      // Even though local group may contain accessible soinfos from other namespaces
      // we should avoid linking them (because if they are not linked -> they
      // are in the local_group_roots and will be linked later).
      if (!si->is_linked() && si->get_primary_namespace() == local_group_ns) {
        const android_dlextinfo* link_extinfo = nullptr;
        if (si == soinfos[0] || reserved_address_recursive) {
          // Only forward extinfo for the first library unless the recursive
          // flag is set.
          link_extinfo = extinfo;
        }
        if (__libc_shared_globals()->load_hook) {
          __libc_shared_globals()->load_hook(si->load_bias, si->phdr, si->phnum);
        }

        //【Load四部曲4】link image : 执行重定位
        lookup_list.set_dt_symbolic_lib(si->has_DT_SYMBOLIC ? si : nullptr);
        if (!si->link_image(lookup_list, local_group_root, link_extinfo, &relro_fd_offset) ||
            !get_cfi_shadow()->AfterLoad(si, solist_get_head())) {
          return false;
        }
      }

      return true;
    });

    if (!linked) {
      return false;
    }
  }

  // Step 7: Mark all load_tasks as linked and increment refcounts
  // for references between load_groups (at this point it does not matter if
  // referenced load_groups were loaded by previous dlopen or as part of this
  // one on step 6)
  if (start_with != nullptr && add_as_children) {
    start_with->set_linked();
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    si->set_linked();
  }

  for (auto&& task : load_tasks) {
    soinfo* si = task->get_soinfo();
    soinfo* needed_by = task->get_needed_by();
    if (needed_by != nullptr &&
        needed_by != start_with &&
        needed_by->get_local_group_root() != si->get_local_group_root()) {
      si->increment_ref_count();
    }
  }


  return true;
}
```

## 结论

``dlopen``函数负责对``elf``文件进行装载，整个装载过程，可概括成总共四个步骤：
1. [Android linker & loader (1) -- Read ELF file](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(1)--read%20ELF.md) : 读取并映射``elf``基础信息，包括读取并校验``elf header``，映射``program header``，映射``section header``，映射``.dynamic section``和``.strtab section``；
2. [Android linker & loader (2) -- Segment Load](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(2)--segment%20load.md) : 分配内存空间，装载``loadable segment``；
3. [Android linker & loader (3) -- prelink](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(3)--image%20prelink.md) : 从``.dynamic section``进一步获取各个段的详细信息；
4. [Android linker & loader (4) -- Relocate](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(4)--image%20link%20%26%20relocate.md) : ``link image``，进行``relocate``的工作；
