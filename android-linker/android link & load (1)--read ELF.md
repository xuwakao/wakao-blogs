# Android linker & loader (1) -- Read ELF file

``dlopen``函数负责对``elf``文件进行装载，整个装载过程，可概括成总共四个步骤：
1. [Android linker & loader (1) -- Read ELF file](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(1)--read%20ELF.md) : 读取并映射``elf``基础信息，包括读取并校验``elf header``，映射``program header``，映射``section header``，映射``.dynamic section``和``.strtab section``；
2. [Android linker & loader (2) -- Segment Load](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(2)--segment%20load.md) : 分配内存空间，装载``loadable segment``；
3. [Android linker & loader (3) -- prelink](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(3)--image%20prelink.md) : 从``.dynamic section``进一步获取各个段的详细信息；
4. [Android linker & loader (4) -- Relocate](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(4)--image%20link%20%26%20relocate.md) : ``link image``，进行``relocate``的工作；


本文分析四部曲的第一部。

## ELF file read
在``dlopen``过程中，有``find_library_internal``，从函数名和参数列表来看，该函数作用就是从指定``namespace``（或者是该``namespace``的``linked namespace``）中，查找加载的``library``，实际上，它不止完成查找，还负责``elf``文件的信息读取。

```cpp
static bool find_library_internal(android_namespace_t* ns,
                                  LoadTask* task,
                                  ZipArchiveCache* zip_archive_cache,
                                  LoadTaskList* load_tasks,
                                  int rtld_flags,
                                  bool search_linked_namespaces) {
  soinfo* candidate;

  //根据soname, 在已加载的列表中查找，找到就直接返回对应soinfo
  if (find_loaded_library_by_soname(ns, task->get_name(), search_linked_namespaces, &candidate)) {
    LD_LOG(kLogDlopen,
           "find_library_internal(ns=%s, task=%s): Already loaded (by soname): %s",
           ns->get_name(), task->get_name(), candidate->get_realpath());
    task->set_soinfo(candidate);
    return true;
  }

  // Library might still be loaded, the accurate detection
  // of this fact is done by load_library.
  TRACE("[ \"%s\" find_loaded_library_by_soname failed (*candidate=%s@%p). Trying harder... ]",
        task->get_name(), candidate == nullptr ? "n/a" : candidate->get_realpath(), candidate);

  //关键函数：根据soname没有找到，就通过该函数继续查找（通过inode），还是找不到才加载
  if (load_library(ns, task, zip_archive_cache, load_tasks, rtld_flags, search_linked_namespaces)) {
    return true;
  }

  // TODO(dimitry): workaround for http://b/26394120 (the grey-list)
  if (ns->is_greylist_enabled() && is_greylisted(ns, task->get_name(), task->get_needed_by())) {
    // For the libs in the greylist, switch to the default namespace and then
    // try the load again from there. The library could be loaded from the
    // default namespace or from another namespace (e.g. runtime) that is linked
    // from the default namespace.
    LD_LOG(kLogDlopen,
           "find_library_internal(ns=%s, task=%s): Greylisted library - trying namespace %s",
           ns->get_name(), task->get_name(), g_default_namespace.get_name());
    ns = &g_default_namespace;
    if (load_library(ns, task, zip_archive_cache, load_tasks, rtld_flags,
                     search_linked_namespaces)) {
      return true;
    }
  }
  // END OF WORKAROUND

  //找不到，就在linked namespace中查找
  if (search_linked_namespaces) {
    // if a library was not found - look into linked namespaces
    // preserve current dlerror in the case it fails.
    DlErrorRestorer dlerror_restorer;
    LD_LOG(kLogDlopen, "find_library_internal(ns=%s, task=%s): Trying %zu linked namespaces",
           ns->get_name(), task->get_name(), ns->linked_namespaces().size());
    for (auto& linked_namespace : ns->linked_namespaces()) {
      if (find_library_in_linked_namespace(linked_namespace, task)) {
        if (task->get_soinfo() == nullptr) {
          // try to load the library - once namespace boundary is crossed
          // we need to load a library within separate load_group
          // to avoid using symbols from foreign namespace while.
          //
          // However, actual linking is deferred until when the global group
          // is fully identified and is applied to all namespaces.
          // Otherwise, the libs in the linked namespace won't get symbols from
          // the global group.
          if (load_library(linked_namespace.linked_namespace(), task, zip_archive_cache, load_tasks, rtld_flags, false)) {
            LD_LOG(
                kLogDlopen, "find_library_internal(ns=%s, task=%s): Found in linked namespace %s",
                ns->get_name(), task->get_name(), linked_namespace.linked_namespace()->get_name());
            return true;
          }
        } else {
          // lib is already loaded
          return true;
        }
      }
    }
  }

  return false;
}
```

主要逻辑，在``load_library``中进行。

## load_library ： find & load

```cpp
static bool load_library(android_namespace_t* ns,
                         LoadTask* task,
                         ZipArchiveCache* zip_archive_cache,
                         LoadTaskList* load_tasks,
                         int rtld_flags,
                         bool search_linked_namespaces) {
  const char* name = task->get_name();
  soinfo* needed_by = task->get_needed_by();
  const android_dlextinfo* extinfo = task->get_extinfo();

  off64_t file_offset;
  std::string realpath;
  if (extinfo != nullptr && (extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD) != 0) {
    file_offset = 0;
    if ((extinfo->flags & ANDROID_DLEXT_USE_LIBRARY_FD_OFFSET) != 0) {
      file_offset = extinfo->library_fd_offset;
    }

    if (!realpath_fd(extinfo->library_fd, &realpath)) {
      if (!is_first_stage_init()) {
        PRINT(
            "warning: unable to get realpath for the library \"%s\" by extinfo->library_fd. "
            "Will use given name.",
            name);
      }
      realpath = name;
    }

    task->set_fd(extinfo->library_fd, false);
    task->set_file_offset(file_offset);
    return load_library(ns, task, load_tasks, rtld_flags, realpath, search_linked_namespaces);
  }

  LD_LOG(kLogDlopen,
         "load_library(ns=%s, task=%s, flags=0x%x, search_linked_namespaces=%d): calling "
         "open_library with realpath=%s",
         ns->get_name(), name, rtld_flags, search_linked_namespaces, realpath.c_str());

  // Open the file.
  //打开文件
  int fd = open_library(ns, zip_archive_cache, name, needed_by, &file_offset, &realpath);
  if (fd == -1) {
    if (task->is_dt_needed()) {
      if (needed_by->is_main_executable()) {
        DL_OPEN_ERR("library \"%s\" not found: needed by main executable", name);
      } else {
        DL_OPEN_ERR("library \"%s\" not found: needed by %s in namespace %s", name,
                    needed_by->get_realpath(), task->get_start_from()->get_name());
      }
    } else {
      DL_OPEN_ERR("library \"%s\" not found", name);
    }
    return false;
  }

  task->set_fd(fd, true);
  task->set_file_offset(file_offset);
  
  return load_library(ns, task, load_tasks, rtld_flags, realpath, search_linked_namespaces);
}

static bool load_library(android_namespace_t* ns,
                         LoadTask* task,
                         LoadTaskList* load_tasks,
                         int rtld_flags,
                         const std::string& realpath,
                         bool search_linked_namespaces) {
  off64_t file_offset = task->get_file_offset();
  const char* name = task->get_name();
  const android_dlextinfo* extinfo = task->get_extinfo();

  LD_LOG(kLogDlopen,
         "load_library(ns=%s, task=%s, flags=0x%x, realpath=%s, search_linked_namespaces=%d)",
         ns->get_name(), name, rtld_flags, realpath.c_str(), search_linked_namespaces);

  if ((file_offset % PAGE_SIZE) != 0) {
    DL_OPEN_ERR("file offset for the library \"%s\" is not page-aligned: %" PRId64, name, file_offset);
    return false;
  }
  if (file_offset < 0) {
    DL_OPEN_ERR("file offset for the library \"%s\" is negative: %" PRId64, name, file_offset);
    return false;
  }

  struct stat file_stat;
  if (TEMP_FAILURE_RETRY(fstat(task->get_fd(), &file_stat)) != 0) {
    DL_OPEN_ERR("unable to stat file for the library \"%s\": %s", name, strerror(errno));
    return false;
  }
  if (file_offset >= file_stat.st_size) {
    DL_OPEN_ERR("file offset for the library \"%s\" >= file size: %" PRId64 " >= %" PRId64,
        name, file_offset, file_stat.st_size);
    return false;
  }

  // Check for symlink and other situations where
  // file can have different names, unless ANDROID_DLEXT_FORCE_LOAD is set
  if (extinfo == nullptr || (extinfo->flags & ANDROID_DLEXT_FORCE_LOAD) == 0) {
    soinfo* si = nullptr;
    //通过inode查找
    if (find_loaded_library_by_inode(ns, file_stat, file_offset, search_linked_namespaces, &si)) {
      LD_LOG(kLogDlopen,
             "load_library(ns=%s, task=%s): Already loaded under different name/path \"%s\" - "
             "will return existing soinfo",
             ns->get_name(), name, si->get_realpath());
      task->set_soinfo(si);
      return true;
    }
  }

  if ((rtld_flags & RTLD_NOLOAD) != 0) {
    DL_OPEN_ERR("library \"%s\" wasn't loaded and RTLD_NOLOAD prevented it", name);
    return false;
  }

  struct statfs fs_stat;
  if (TEMP_FAILURE_RETRY(fstatfs(task->get_fd(), &fs_stat)) != 0) {
    DL_OPEN_ERR("unable to fstatfs file for the library \"%s\": %s", name, strerror(errno));
    return false;
  }

  // do not check accessibility using realpath if fd is located on tmpfs
  // this enables use of memfd_create() for apps
  //如果当前namespace不能访问这个so，除非是grey list的库，否则都直接加载失败
  if ((fs_stat.f_type != TMPFS_MAGIC) && (!ns->is_accessible(realpath))) {
    // TODO(dimitry): workaround for http://b/26394120 - the grey-list

    // TODO(dimitry) before O release: add a namespace attribute to have this enabled
    // only for classloader-namespaces
    const soinfo* needed_by = task->is_dt_needed() ? task->get_needed_by() : nullptr;
    if (is_greylisted(ns, name, needed_by)) {
      // print warning only if needed by non-system library
      if (needed_by == nullptr || !is_system_library(needed_by->get_realpath())) {
        const soinfo* needed_or_dlopened_by = task->get_needed_by();
        const char* sopath = needed_or_dlopened_by == nullptr ? "(unknown)" :
                                                      needed_or_dlopened_by->get_realpath();
        DL_WARN_documented_change(24,
                                  "private-api-enforced-for-api-level-24",
                                  "library \"%s\" (\"%s\") needed or dlopened by \"%s\" "
                                  "is not accessible by namespace \"%s\"",
                                  name, realpath.c_str(), sopath, ns->get_name());
        add_dlwarning(sopath, "unauthorized access to",  name);
      }
    } else {
      // do not load libraries if they are not accessible for the specified namespace.
      const char* needed_or_dlopened_by = task->get_needed_by() == nullptr ?
                                          "(unknown)" :
                                          task->get_needed_by()->get_realpath();

      DL_OPEN_ERR("library \"%s\" needed or dlopened by \"%s\" is not accessible for the namespace \"%s\"",
             name, needed_or_dlopened_by, ns->get_name());

      // do not print this if a library is in the list of shared libraries for linked namespaces
      if (!maybe_accessible_via_namespace_links(ns, name)) {
        PRINT("library \"%s\" (\"%s\") needed or dlopened by \"%s\" is not accessible for the"
              " namespace: [name=\"%s\", ld_library_paths=\"%s\", default_library_paths=\"%s\","
              " permitted_paths=\"%s\"]",
              name, realpath.c_str(),
              needed_or_dlopened_by,
              ns->get_name(),
              android::base::Join(ns->get_ld_library_paths(), ':').c_str(),
              android::base::Join(ns->get_default_library_paths(), ':').c_str(),
              android::base::Join(ns->get_permitted_paths(), ':').c_str());
      }
      return false;
    }
  }

  soinfo* si = soinfo_alloc(ns, realpath.c_str(), &file_stat, file_offset, rtld_flags);
  if (si == nullptr) {
    return false;
  }

  task->set_soinfo(si);

  // Read the ELF header and some of the segments.
  if (!task->read(realpath.c_str(), file_stat.st_size)) {
    soinfo_free(si);
    task->set_soinfo(nullptr);
    return false;
  }

  // find and set DT_RUNPATH and dt_soname
  // Note that these field values are temporary and are
  // going to be overwritten on soinfo::prelink_image
  // with values from PT_LOAD segments.
  const ElfReader& elf_reader = task->get_elf_reader();
  for (const ElfW(Dyn)* d = elf_reader.dynamic(); d->d_tag != DT_NULL; ++d) {
    if (d->d_tag == DT_RUNPATH) {
      si->set_dt_runpath(elf_reader.get_string(d->d_un.d_val));
    }
    if (d->d_tag == DT_SONAME) {
      si->set_soname(elf_reader.get_string(d->d_un.d_val));
    }
  }

#if !defined(__ANDROID__)
  // Bionic on the host currently uses some Android prebuilts, which don't set
  // DT_RUNPATH with any relative paths, so they can't find their dependencies.
  // b/118058804
  if (si->get_dt_runpath().empty()) {
    si->set_dt_runpath("$ORIGIN/../lib64:$ORIGIN/lib64");
  }
#endif

  //根据so的DT_NEEDED信息，添加所有的依赖到load tasks中
  for_each_dt_needed(task->get_elf_reader(), [&](const char* name) {
    LD_LOG(kLogDlopen, "load_library(ns=%s, task=%s): Adding DT_NEEDED task: %s",
           ns->get_name(), task->get_name(), name);
    load_tasks->push_back(LoadTask::create(name, si, ns, task->get_readers_map()));
  });

  return true;
}
```

``load_library``函数如果不能在已加载库中查找到当前``so``，最终通过``elf reader``来读取``elf``的信息。

## ELF read : 读取和映射elf基础信息

```cpp
  bool read(const char* realpath, off64_t file_size) {
    ElfReader& elf_reader = get_elf_reader();
    return elf_reader.Read(realpath, fd_, file_offset_, file_size);
  }
```

```cpp
bool ElfReader::Read(const char* name, int fd, off64_t file_offset, off64_t file_size) {
  if (did_read_) {
    return true;
  }
  name_ = name;
  fd_ = fd;
  file_offset_ = file_offset;
  file_size_ = file_size;

  if (ReadElfHeader() && //读取elf header
      VerifyElfHeader() && //校验 elf header合法性
      ReadProgramHeaders() && //map program headers
      ReadSectionHeaders() && //map section headers
      ReadDynamicSection()) {//map .dynamic section和.strtab section
    did_read_ = true;
  }

  return did_read_;
}
```

经过``elf read``后，``so``文件的基础信息已经读取或者映射到内存中，往后装载和链接中，这些基础信息都相当重要，这些基础信息包括（``ElfReader``）：

```cpp
  std::string name_;//elf 文件路径
  int fd_;//elf文件描述符
  off64_t file_offset_;//基础offset（一般不存在，用于在文件中包含elf文件的情况，例如zip文件中包含没有压缩的elf）
  off64_t file_size_;//文件大小

  ElfW(Ehdr) header_;//elf header
  size_t phdr_num_;//program header数目

  MappedFileFragment phdr_fragment_;//program headers table的内存映射
  const ElfW(Phdr)* phdr_table_;//program headers table

  MappedFileFragment shdr_fragment_;//section headers table的内存映射
  const ElfW(Shdr)* shdr_table_;//section headers table
  size_t shdr_num_;//sections数目

  MappedFileFragment dynamic_fragment_;//.dynamic section的内存映射
  const ElfW(Dyn)* dynamic_;//.dynamic section

  MappedFileFragment strtab_fragment_;//.strtab section的内存映射
  const char* strtab_;//.strtab section
  size_t strtab_size_;//.strtab字节占用大小
```

## 结论

``dlopen``的第一部曲，完成了``elf``文件的关键信息读取，包括：读取并校验``elf header``，映射``program header``,，映射``section header``，映射``.dynamic section``和``.strtab section``。




