# RELR relative relocations 
从[Android linker changes for NDK developers](https://android.googlesource.com/platform/bionic/+/master/android-changes-for-ndk-developers.md#relative-relocations-relr)可知，``api level >= 28``中，实验性质地引入了``RELR relative relocations``。

# Why & what

为什么会出现``relr section``，详细看：[Reducing code size of Position Independent Executables (PIE) by shrinking the size of dynamic relocations section](https://sourceware.org/legacy-ml/gnu-gabi/2017-q2/msg00000.html) 和 [Re: Reducing code size of Position Independent Executables (PIE) by shrinking the size of dynamic relocations section](https://sourceware.org/legacy-ml/gnu-gabi/2017-q2/msg00003.html)。。

主要就是，``PIE/PIC``引入动态链接向相关的表项，导致库文件增大。所以，需要想办法``shrinking``重定位表。

# How & proposal

``R_*_RELATIVE``类型的重定位占了大多数，所以可以专门针对这个类型的重定位进行优化。``R_*_RELATIVE``重定位表项中，实际有用的字段就是``offset``，其他字段用处不大。

接近``android relr``的方案，可以参考：(Proposal for a new section type SHT_RELR)[https://groups.google.com/g/generic-abi/c/bX460iggiKg]

# Proposal实现
上面文档提出方案中，对于如何重定位，代码大致如下：

```cpp
#define ELF64_R_JUMP(val) ((val) >> 56)
#define ELF64_R_BITS(val) ((val) & 0xffffffffffffff)

#ifdef DO_RELR
{
    ElfW(Addr) offset = 0;
    for (; relative < end; ++relative)
    {
        ElfW(Addr) jump = ELFW(R_JUMP) (*relative);
        ElfW(Addr) bits = ELFW(R_BITS) (*relative);
        offset += jump * sizeof(ElfW(Addr));

        if (jump == 0)
        {
            ++relative;
            offset = *relative;
        }
        
        ElfW(Addr) r_offset = offset;
        for (; bits != 0; bits >>= 1)
        {
            if ((bits&1) != 0) elf_machine_relr_relative (l_addr, (void *) (l_addr+r_offset));
            r_offset += sizeof(ElfW(Addr));
        }
    }
}
#endif
```

该方案是，每一项的值由``delta-encoding``和``bitmap offset``组成。例如，对于64位``entry``，高8位为``delta-encoding``(也可以叫``jump``)，低56位为``bitmap offset``。

``delta-encoding``（``jump``）代表，相对于上一次的``offset``，需要偏移多少个``ElfW(Addr)``长度的量。如果该值为0，那么下一个表项的值，则为``offset``。

``bitmap offset``，每一个非``0 bit``位，则按顺序偏移``ElfW(Addr)``长度的量。

# Android relr实现

上面的方案和在``android linker``中的实现还是一定差距：

```cpp
void soinfo::apply_relr_reloc(ElfW(Addr) offset) {
  ElfW(Addr) address = offset + load_bias;
  *reinterpret_cast<ElfW(Addr)*>(address) += load_bias;
}

// Process relocations in SHT_RELR section (experimental).
// Details of the encoding are described in this post:
//   https://groups.google.com/d/msg/generic-abi/bX460iggiKg/Pi9aSwwABgAJ
bool soinfo::relocate_relr() {
  ElfW(Relr)* begin = relr_;
  ElfW(Relr)* end = relr_ + relr_count_;
  constexpr size_t wordsize = sizeof(ElfW(Addr));

  ElfW(Addr) base = 0;
  for (ElfW(Relr)* current = begin; current < end; ++current) {
    ElfW(Relr) entry = *current;
    ElfW(Addr) offset;

    //值为偶数的表项，该值就是relocation的offset
    if ((entry&1) == 0) {
      // Even entry: encodes the offset for next relocation.
      offset = static_cast<ElfW(Addr)>(entry);
      apply_relr_reloc(offset);
      // Set base offset for subsequent bitmap entries.
      base = offset + wordsize;
      continue;
    }

    //对于值为奇数的表项，该值是bitmap，每一个非0 bit位，代表一次相对于base的便宜
    // Odd entry: encodes bitmap for relocations starting at base.
    offset = base;
    while (entry != 0) {
      entry >>= 1;
      if ((entry&1) != 0) {
        apply_relr_reloc(offset);
      }
      offset += wordsize;
    }

    // Advance base offset by 63 words for 64-bit platforms,
    // or 31 words for 32-bit platforms.
    base += (8*wordsize - 1) * wordsize;
  }
  return true;
}
```



