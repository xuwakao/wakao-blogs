# Android linker & loader (2) -- Segment Load

``dlopen``函数负责对``elf``文件进行装载，整个装载过程，可概括成总共四个步骤：
1. [Android linker & loader (1) -- Read ELF file](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(1)--read%20ELF.md) : 读取并映射``elf``基础信息，包括读取并校验``elf header``，映射``program header``，映射``section header``，映射``.dynamic section``和``.strtab section``；
2. [Android linker & loader (2) -- Segment Load](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(2)--segment%20load.md) : 分配内存空间，装载``loadable segment``；
3. [Android linker & loader (3) -- prelink](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(3)--image%20prelink.md) : 从``.dynamic section``进一步获取各个段的详细信息；
4. [Android linker & loader (4) -- Relocate](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(4)--image%20link%20%26%20relocate.md) : ``link image``，进行``relocate``的工作；


本文分析四部曲的第二部。

## ELF load

加载``segment``的相关函数如下：
```cpp (bionic/linker/linker.cpp)
  bool load(address_space_params* address_space) {
    ElfReader& elf_reader = get_elf_reader();
    if (!elf_reader.Load(address_space)) {
      return false;
    }

    si_->base = elf_reader.load_start();
    si_->size = elf_reader.load_size();
    si_->set_mapped_by_caller(elf_reader.is_mapped_by_caller());
    si_->load_bias = elf_reader.load_bias();
    si_->phnum = elf_reader.phdr_count();
    si_->phdr = elf_reader.loaded_phdr();

    return true;
  }
```
通过``ElfReader``的``load``函数完成段加载任务。

```cpp (bionic/linker/linker_phdr.cpp)
bool ElfReader::Load(address_space_params* address_space) {
  CHECK(did_read_);
  if (did_load_) {
    return true;
  }
  if (ReserveAddressSpace(address_space) && LoadSegments() && FindPhdr()) {
    did_load_ = true;
  }

  return did_load_;
}
```


``load``函数完成下面三件事情：
1. 分配空间；（``ReserveAddressSpace``）
2. 映射可加载segment；（``LoadSegments``）
3. 计算``program header table``的地址；(``FindPhdr``)

### ReserveAddressSpace ： 分配内存空间
``ReserveAddressSpace``为将要加载的``segments``，分配一段足够大的内存空间。

```cpp
// Reserve a virtual address range big enough to hold all loadable
// segments of a program header table. This is done by creating a
// private anonymous mmap() with PROT_NONE.
bool ElfReader::ReserveAddressSpace(address_space_params* address_space) {
  ElfW(Addr) min_vaddr;
  load_size_ = phdr_table_get_load_size(phdr_table_, phdr_num_, &min_vaddr);
  if (load_size_ == 0) {
    DL_ERR("\"%s\" has no loadable segments", name_.c_str());
    return false;
  }

  uint8_t* addr = reinterpret_cast<uint8_t*>(min_vaddr);
  void* start;

  if (load_size_ > address_space->reserved_size) {
    if (address_space->must_use_address) {
      DL_ERR("reserved address space %zd smaller than %zd bytes needed for \"%s\"",
             load_size_ - address_space->reserved_size, load_size_, name_.c_str());
      return false;
    }
    start = ReserveAligned(load_size_, kLibraryAlignment);
    if (start == nullptr) {
      DL_ERR("couldn't reserve %zd bytes of address space for \"%s\"", load_size_, name_.c_str());
      return false;
    }
  } else {
    start = address_space->start_addr;
    mapped_by_caller_ = true;

    // Update the reserved address space to subtract the space used by this library.
    address_space->start_addr = reinterpret_cast<uint8_t*>(address_space->start_addr) + load_size_;
    address_space->reserved_size -= load_size_;
  }

  load_start_ = start;
  load_bias_ = reinterpret_cast<uint8_t*>(start) - addr;
  return true;
}
```

``phdr_table_get_load_size``计算加载段大小的同时，也计算了``min_vaddr``，``min_vaddr``实际就是首个可加载``segment``的``p_vaddr``。
从上看出，``ReserveAddressSpace``目的，实际就是分配空间，并计算出``load_bias_``。（``load bias``值很重要，详细看：[android linker load bias就是基址？？](https://github.com/xuwakao/wakao-blogs/blob/master/android-linker/android%20link%20%26%20load%20(5)--load%20bias.md)）

### 加载Segment

分配内存空间后，开始加载``loadable segment``（不是所有的``segment``都是需要加载到内存的，只有类型为``PT_LOAD``的需要）。

```cpp
bool ElfReader::LoadSegments() {
  for (size_t i = 0; i < phdr_num_; ++i) {
    const ElfW(Phdr)* phdr = &phdr_table_[i];

    if (phdr->p_type != PT_LOAD) {
      continue;
    }

    // Segment addresses in memory.
    ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
    ElfW(Addr) seg_end   = seg_start + phdr->p_memsz;

    ElfW(Addr) seg_page_start = PAGE_START(seg_start);
    ElfW(Addr) seg_page_end   = PAGE_END(seg_end);

    ElfW(Addr) seg_file_end   = seg_start + phdr->p_filesz;

    // File offsets.
    ElfW(Addr) file_start = phdr->p_offset;
    ElfW(Addr) file_end   = file_start + phdr->p_filesz;

    ElfW(Addr) file_page_start = PAGE_START(file_start);
    ElfW(Addr) file_length = file_end - file_page_start;

    if (file_size_ <= 0) {
      DL_ERR("\"%s\" invalid file size: %" PRId64, name_.c_str(), file_size_);
      return false;
    }

    if (file_end > static_cast<size_t>(file_size_)) {
      DL_ERR("invalid ELF file \"%s\" load segment[%zd]:"
          " p_offset (%p) + p_filesz (%p) ( = %p) past end of file (0x%" PRIx64 ")",
          name_.c_str(), i, reinterpret_cast<void*>(phdr->p_offset),
          reinterpret_cast<void*>(phdr->p_filesz),
          reinterpret_cast<void*>(file_end), file_size_);
      return false;
    }

    if (file_length != 0) {
      int prot = PFLAGS_TO_PROT(phdr->p_flags);
      if ((prot & (PROT_EXEC | PROT_WRITE)) == (PROT_EXEC | PROT_WRITE)) {
        // W + E PT_LOAD segments are not allowed in O.
        if (get_application_target_sdk_version() >= 26) {
          DL_ERR_AND_LOG("\"%s\": W+E load segments are not allowed", name_.c_str());
          return false;
        }
        DL_WARN_documented_change(26,
                                  "writable-and-executable-segments-enforced-for-api-level-26",
                                  "\"%s\" has load segments that are both writable and executable",
                                  name_.c_str());
        add_dlwarning(name_.c_str(), "W+E load segments");
      }

      void* seg_addr = mmap64(reinterpret_cast<void*>(seg_page_start),
                            file_length,
                            prot,
                            MAP_FIXED|MAP_PRIVATE,
                            fd_,
                            file_offset_ + file_page_start);
      if (seg_addr == MAP_FAILED) {
        DL_ERR("couldn't map \"%s\" segment %zd: %s", name_.c_str(), i, strerror(errno));
        return false;
      }
    }

    // if the segment is writable, and does not end on a page boundary,
    // zero-fill it until the page limit.
    if ((phdr->p_flags & PF_W) != 0 && PAGE_OFFSET(seg_file_end) > 0) {
      memset(reinterpret_cast<void*>(seg_file_end), 0, PAGE_SIZE - PAGE_OFFSET(seg_file_end));
    }

    seg_file_end = PAGE_END(seg_file_end);

    // seg_file_end is now the first page address after the file
    // content. If seg_end is larger, we need to zero anything
    // between them. This is done by using a private anonymous
    // map for all extra pages.
    if (seg_page_end > seg_file_end) {
      size_t zeromap_size = seg_page_end - seg_file_end;
      void* zeromap = mmap(reinterpret_cast<void*>(seg_file_end),
                           zeromap_size,
                           PFLAGS_TO_PROT(phdr->p_flags),
                           MAP_FIXED|MAP_ANONYMOUS|MAP_PRIVATE,
                           -1,
                           0);
      if (zeromap == MAP_FAILED) {
        DL_ERR("couldn't zero fill \"%s\" gap: %s", name_.c_str(), strerror(errno));
        return false;
      }

      prctl(PR_SET_VMA, PR_SET_VMA_ANON_NAME, zeromap, zeromap_size, ".bss");
    }
  }
  return true;
}
```

``LoadSegments``很简单，就是计算``loadable segment``的所在页表范围，并映射到内存中去。对于``p_memsz``大于``p_filesz``的情况，把多余的内存区域，作为额外的``.bss``段。

### 计算``program header table``的地址

虽然前面在``ReadProgramHeaders``的时候，映射过一次``program header table``，但是，``phdr_table_``只是临时使用的，在之后会被销毁，所以这里``FindPhdr``要重新计算一遍。

```cpp
// Sets loaded_phdr_ to the address of the program header table as it appears
// in the loaded segments in memory. This is in contrast with phdr_table_,
// which is temporary and will be released before the library is relocated.
bool ElfReader::FindPhdr() {
  const ElfW(Phdr)* phdr_limit = phdr_table_ + phdr_num_;

  //如果程序头表包含PT_PHDR的项，直接使用该项计算
  // If there is a PT_PHDR, use it directly.
  for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
    if (phdr->p_type == PT_PHDR) {
      return CheckPhdr(load_bias_ + phdr->p_vaddr);
    }
  }

  //如果程序头表不包含PT_PHDR的项，找到offset位0和类型为PT_LOAD的项，这个segment
  //肯定映射了elf header和program header，根据elf header的信息计算
  // Otherwise, check the first loadable segment. If its file offset
  // is 0, it starts with the ELF header, and we can trivially find the
  // loaded program header from it.
  for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
    if (phdr->p_type == PT_LOAD) {
      if (phdr->p_offset == 0) {
        ElfW(Addr)  elf_addr = load_bias_ + phdr->p_vaddr;
        const ElfW(Ehdr)* ehdr = reinterpret_cast<const ElfW(Ehdr)*>(elf_addr);
        ElfW(Addr)  offset = ehdr->e_phoff;
        return CheckPhdr(reinterpret_cast<ElfW(Addr)>(ehdr) + offset);
      }
      break;
    }
  }

  DL_ERR("can't find loaded phdr for \"%s\"", name_.c_str());
  return false;
}

//检验计算出的地址，一定要在loadable segment的范围内
// Ensures that our program header is actually within a loadable
// segment. This should help catch badly-formed ELF files that
// would cause the linker to crash later when trying to access it.
bool ElfReader::CheckPhdr(ElfW(Addr) loaded) {
  const ElfW(Phdr)* phdr_limit = phdr_table_ + phdr_num_;
  ElfW(Addr) loaded_end = loaded + (phdr_num_ * sizeof(ElfW(Phdr)));
  for (const ElfW(Phdr)* phdr = phdr_table_; phdr < phdr_limit; ++phdr) {
    if (phdr->p_type != PT_LOAD) {
      continue;
    }
    ElfW(Addr) seg_start = phdr->p_vaddr + load_bias_;
    ElfW(Addr) seg_end = phdr->p_filesz + seg_start;
    if (seg_start <= loaded && loaded_end <= seg_end) {
      loaded_phdr_ = reinterpret_cast<const ElfW(Phdr)*>(loaded);
      return true;
    }
  }
  DL_ERR("\"%s\" loaded phdr %p not in loadable segment",
         name_.c_str(), reinterpret_cast<void*>(loaded));
  return false;
}
```

计算```phdr``（注意看注释), ``phdr``一定在某个``loadable segment``当中。

## 结论

``dlopen``的第二部曲，完成了内存空间的分配，并加载``segment``到内存中。