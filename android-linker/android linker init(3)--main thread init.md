# Linker和主线程初始化

**NOTE: 这里分析arm64的情况。**

## two-pass初始化

主线程初始化流程在``linker``要分为三部分（都在``__linker_init``的中），分别为``__libc_init_main_thread_early``、``__libc_init_main_thread_late``和``__libc_init_main_thread_final``。

### 1-pass : __libc_init_main_thread_early

```cpp

//-------------------------------------------------------------
//bionic/libc/platform/bionic/tls_defines.h
#if defined(__arm__) || defined(__aarch64__)

// The ARM ELF TLS ABI specifies[1] that the thread pointer points at a 2-word
// TCB followed by the executable's TLS segment. Both the TCB and the
// executable's segment are aligned according to the segment, so Bionic requires
// a minimum segment alignment, which effectively reserves an 8-word TCB. The
// ARM spec allocates the first TCB word to the DTV.
//
// [1] "Addenda to, and Errata in, the ABI for the ARM Architecture". Section 3.
// http://infocenter.arm.com/help/topic/com.arm.doc.ihi0045e/IHI0045E_ABI_addenda.pdf

#define MIN_TLS_SLOT            (-1) // update this value when reserving a slot
#define TLS_SLOT_BIONIC_TLS     (-1)
#define TLS_SLOT_DTV              0
#define TLS_SLOT_THREAD_ID        1
#define TLS_SLOT_APP              2 // was historically used for errno
#define TLS_SLOT_OPENGL           3
#define TLS_SLOT_OPENGL_API       4
#define TLS_SLOT_STACK_GUARD      5
#define TLS_SLOT_SANITIZER        6 // was historically used for dlerror
#define TLS_SLOT_ART_THREAD_SELF  7

// The maximum slot is fixed by the minimum TLS alignment in Bionic executables.
#define MAX_TLS_SLOT         

//-------------------------------------------------------------
//bionic/linker/linker_main.cpp

  // Initialize TLS early so system calls and errno work.
  KernelArgumentBlock args(raw_args);
  bionic_tcb temp_tcb __attribute__((uninitialized));
  linker_memclr(&temp_tcb, sizeof(temp_tcb));//初始化tcb空间
  __libc_init_main_thread_early(args, &temp_tcb);


//-------------------------------------------------------------
//bionic/libc/bionic/__libc_init_main_thread.cpp

  static pthread_internal_t main_thread;

// Setup for the main thread. For dynamic executables, this is called by the
// linker _before_ libc is mapped in memory. This means that all writes to
// globals from this function will apply to linker-private copies and will not
// be visible from libc later on.
//
// Note: this function creates a pthread_internal_t for the initial thread and
// stores the pointer in TLS, but does not add it to pthread's thread list. This
// has to be done later from libc itself (see __libc_init_common).
//
// This is in a file by itself because it needs to be built with
// -fno-stack-protector because it's responsible for setting up the main
// thread's TLS (which stack protector relies on). It's also built with
// -ffreestanding because the early init function runs in the linker before
// ifunc resolvers have run.

// Do enough setup to:
//  - Let the dynamic linker invoke system calls (and access errno)
//  - Ensure that TLS access functions (__get_{tls,thread}) never return NULL
//  - Allow the stack protector to work (with a zero cookie)
// Avoid doing much more because, when this code is called within the dynamic
// linker, the linker binary hasn't been relocated yet, so certain kinds of code
// are hazardous, such as accessing non-hidden global variables or calling
// string.h functions.
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
extern "C" void __libc_init_main_thread_early(const KernelArgumentBlock& args,
                                              bionic_tcb* temp_tcb) {
  __libc_shared_globals()->auxv = args.auxv;
#if defined(__i386__)
  __libc_init_sysinfo(); // uses AT_SYSINFO auxv entry
#endif
  __init_tcb(temp_tcb, &main_thread);//初始化tcb，并用slot记录下main_thread
  __init_tcb_dtv(temp_tcb);//初始化DTV，并用slot记录下dtv
  __set_tls(&temp_tcb->tls_slot(0));//把tls_slot[0]的指针存储到tpidr_el0寄存器中,在arm64中，tls_slot[0]实际就是TLS, tcb和tls实际是同一个地址
  main_thread.tid = __getpid();
  main_thread.set_cached_pid(main_thread.tid);
  main_thread.stack_top = reinterpret_cast<uintptr_t>(args.argv);
}

  // This code is used both by each new pthread and the code that initializes the main thread.
void __init_tcb(bionic_tcb* tcb, pthread_internal_t* thread) {
#ifdef TLS_SLOT_SELF
  // On x86, slot 0 must point to itself so code can read the thread pointer by
  // loading %fs:0 or %gs:0.
  tcb->tls_slot(TLS_SLOT_SELF) = &tcb->tls_slot(TLS_SLOT_SELF);//x86才执行，忽略
#endif
  tcb->tls_slot(TLS_SLOT_THREAD_ID) = thread;//tls_slot[TLS_SLOT_THREAD_ID]指向main thread
}

//-------------------------------------------------------------
//bionic/libc/arch-arm64/bionic/__set_tls.c
__LIBC_HIDDEN__ void __set_tls(void* tls) {
  asm("msr tpidr_el0, %0" : : "r" (tls));
}

```

### 2-pass : __libc_init_main_thread_late & 为什么要early init和late init

#### Q : 为什么``init``要分``early init``和``late init``?

> A: 原因是，``linker``在完成重定位前，其内部不能引用外部符号或者GOT表等（例如一些全局变量，``string.h``的引用等）。但是在``linker``初始化过程中，而``system call``可能会发生``errno``，则需要初始化``tcb``去获取持有``errno``的主线程``pthread_internal_t``对象。

以``errno``为例，当发生错误后，获取``errno``的方法是：

```cpp
//bionic/libc/bionic/__errno.cpp
int*  __errno() {
  return &__get_thread()->errno_value;
}

//bionic/libc/bionic/pthread_internal.h
// Make __get_thread() inlined for performance reason. See http://b/19825434.
static inline __always_inline pthread_internal_t* __get_thread() {
  return static_cast<pthread_internal_t*>(__get_tls()[TLS_SLOT_THREAD_ID]);
}
```

可见，获取``errno``，需要获得主线程（``pthread_internal_t``），而主线程由``tcb tls slot``引用，所以需要在``linker``重定位前就已经初始化完。

所以，在``__libc_init_main_thread_early``中，需要先初始化``tcb``，使得初始化``tls slot``（``dtv``, ``main thread``）。并在``linker``重定位完成后，再在``__libc_init_main_thread_late``中，进行实际的``bionic_tls``结构的真正初始化的相关操作。

#### __libc_init_main_thread_late

在``__libc_init_main_thread_late``中，继续进行``main thread``的初始化。

```cpp
//bionic/libc/bionic/__libc_init_main_thread.cpp
// Finish initializing the main thread.
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
extern "C" void __libc_init_main_thread_late() {
  __init_bionic_tls_ptrs(__get_bionic_tcb(), __allocate_temp_bionic_tls());//mmap真正的bionic_tls对象

  // Tell the kernel to clear our tid field when we exit, so we're like any other pthread.
  // For threads created by pthread_create, this setup happens during the clone syscall (i.e.
  // CLONE_CHILD_CLEARTID).
  __set_tid_address(&main_thread.tid);

  //pthread attr相关初始化
  pthread_attr_init(&main_thread.attr);
  // We don't want to explicitly set the main thread's scheduler attributes (http://b/68328561).
  pthread_attr_setinheritsched(&main_thread.attr, PTHREAD_INHERIT_SCHED);
  // The main thread has no guard page.
  pthread_attr_setguardsize(&main_thread.attr, 0);
  // User code should never see this; we'll compute it when asked.
  pthread_attr_setstacksize(&main_thread.attr, 0);

  //stack guard相关初始化
  // The TLS stack guard is set from the global, so ensure that we've initialized the global
  // before we initialize the TLS. Dynamic executables will initialize their copy of the global
  // stack protector from the one in the main thread's TLS.
  __libc_safe_arc4random_buf(&__stack_chk_guard, sizeof(__stack_chk_guard));
  __init_tcb_stack_guard(__get_bionic_tcb());

  //初始化main thread（线程状态，调度策略等）
  __init_thread(&main_thread);

  //额外初始化一些stack(signal stack和shadow call (https://source.android.com/devices/tech/debug/shadow-call-stack)))，不懂不展开
  __init_additional_stacks(&main_thread);
}
```

**NOTE : ``late init``的工作注意看代码解释。**

### 3-pass:__libc_init_main_thread_final & 为什么还有final init

#### Q : 为什么还有final init

> A : 因为在前面``late init``阶段，``exe``及其依赖的``DSO``依然没有加载到用户空间，这些``elf``文件中，可能包含``tls``相关的``section``。而``final init``阶段，这些``dso``已经加载完成，需要在这个阶段把每个``module``（即加载后的``dso``）的``tsl``内容拷贝过来，完成``tls``的初始化。

```cpp
// Once all ELF modules are loaded, allocate the final copy of the main thread's
// static TLS memory.
__BIONIC_WEAK_FOR_NATIVE_BRIDGE
extern "C" void __libc_init_main_thread_final() {
  bionic_tcb* temp_tcb = __get_bionic_tcb();
  bionic_tls* temp_tls = &__get_bionic_tls();

  // Allocate the main thread's static TLS. (This mapping doesn't include a
  // stack.)
  ThreadMapping mapping = __allocate_thread_mapping(0, PTHREAD_GUARD_SIZE);
  if (mapping.mmap_base == nullptr) {
    async_safe_fatal("failed to mmap main thread static TLS: %s", strerror(errno));
  }

  const StaticTlsLayout& layout = __libc_shared_globals()->static_tls_layout;
  auto new_tcb = reinterpret_cast<bionic_tcb*>(mapping.static_tls + layout.offset_bionic_tcb());
  auto new_tls = reinterpret_cast<bionic_tls*>(mapping.static_tls + layout.offset_bionic_tls());

  __init_static_tls(mapping.static_tls);
  new_tcb->copy_from_bootstrap(temp_tcb);
  new_tls->copy_from_bootstrap(temp_tls);
  __init_tcb(new_tcb, &main_thread);
  __init_bionic_tls_ptrs(new_tcb, new_tls);

  main_thread.mmap_base = mapping.mmap_base;
  main_thread.mmap_size = mapping.mmap_size;
  main_thread.mmap_base_unguarded = mapping.mmap_base_unguarded;
  main_thread.mmap_size_unguarded = mapping.mmap_size_unguarded;

  __set_tls(&new_tcb->tls_slot(0));

  __set_stack_and_tls_vma_name(true);
  __free_temp_bionic_tls(temp_tls);
}
```

这个阶段，主要是，拷贝各个已加载``module``的``tls``内容到``main thread``中的``tcb/tls``结构中去，真正完成整个``main thread``的初始化。

## Reference

关于``tls``的相关文档：

[ELF Handling For Thread-Local Storage](https://www.akkadia.org/drepper/tls.pdf)

[Addenda to the ABI for the ARM Architecture](https://developer.arm.com/documentation/ihi0045/e/)

[A Deep dive into (implicit) Thread Local Storage](https://chao-tic.github.io/blog/2018/12/25/tls)
