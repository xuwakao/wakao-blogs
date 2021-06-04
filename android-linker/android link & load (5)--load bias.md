# android linker load bias就是基址？？

``load_bias_``这个值很重要，涉及到``so``各部分加载地址的计算。

``load_bias_``的计算方法如下：

``load_bias_ = reinterpret_cast<uint8_t*>(start) - min_vaddr;``

``start``很容易理解，就是``mmap``内存的的起始地址。（由``ReserveAligned``负责计算，里面对于对齐的计算还是比较巧妙的，主要是保证``align up``的部分依旧是该``mapping``的部分）

``min_vaddr``的计算方式如下：

```min_vaddr = PAGE_START(phdr0->p_vaddr)```

``phdr0->p_vaddr``即是首个（地址最低的）可加载段的虚拟地址``p_vaddr``。``p_vaddr``不是真正加载到内存后的地址，只是编译器为``ELF segment``分配的虚拟地址。
但是，``p_vaddr``的分配和``loader``的实现，需要满足一个条件就是，任意一个段的``load address``和``p_vaddr``的差值恒定。

> 例子：
>  
> 考虑下面两个``segment``的情况：
>
>    [ offset:0,      filesz:0x4000, memsz:0x4000, vaddr:0x30000 ],
>    [ offset:0x4000, filesz:0x2000, memsz:0x8000, vaddr:0x40000 ],
>
>  两个``segment``覆盖的地址空间是：
>
>       0x30000...0x34000
>       0x40000...0x48000
>  
>  如果``loader``加载首个``segment``的内存地址为``0xa0000000``，那么，它们两个的真正加载内存区域为：
>
>       0xa0030000...0xa0034000
>       0xa0040000...0xa0048000

也就是说，任意段``x``，需要满足：

```phdrX_load_address - phdrX->p_vaddr = phdr0_load_address - phdr0->p_vaddr```

右值``phdr0_load_address - phdr0->p_vaddr``即为``load_bias``。

但上面公式还是有一个问题，``p_vaddr``的值可能不是页对齐的，计算出来``phdrX_load_address``很有可能也不是页对齐的。但是加载每个段的起始地址是页对齐的。所以，需要对该计算方法进行调整。

将``load_bias``调整为：
```phdr0_load_address - PAGE_START(phdr0->p_vaddr)```

也就是说，不要求任意一个段的``load address``和``p_vaddr``的差值恒定。而是，任意一个段的``load address``和``p_vaddr``**所在页**的差值恒定。这个时候，``phdrX_load_address``不再是``segment``内容的真正起始位置，真正的起始位置为：
``phdrX_load_address + PAGE_OFFSET(phdr0->p_vaddr)``。但是，从概念上，依然认为``phdrX_load_address``是段的基址。

## 结论

``load_bias``有些人认为它是所谓``so``的基址，这个说法是不准确的。从常规认知角度来说，``mmap``的起始地址才是段基址。但是，由于``offset``的存在，段基址又不一定是``segment``的内容起始位置。

其实，``load_bias``只是一个巧妙的计算方法，以满足**页对齐**和**任意一个段的``load address``和``p_vaddr``的差值恒定**的要求，并没有实际含义。
