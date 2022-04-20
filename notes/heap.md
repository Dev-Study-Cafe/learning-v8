### Heap
This document will try to describe the Heap in V8 and the related classes and
functions.

When a V8 Isolate is created it is usually done by something like this:
```c++
  isolate_ = v8::Isolate::New(create_params_);
```
The first thing that happens in v8::Isolate::New is that the Isolate is
allocated:
```c++
Isolate* Isolate::New(const Isolate::CreateParams& params) {
  Isolate* isolate = Allocate()
  ...
}
```
And allocate calls v8::internal::Isolate::New and casts it back to v8::Isolate:
```c++
Isolate* Isolate::Allocate() {
  return reinterpret_cast<Isolate*>(i::Isolate::New());
}
```

### IsolateAllocator
Each Isolate has an IsolateAllocator instance associated with it which is passes
into the constructor:
```c++
Isolate* Isolate::New(IsolateAllocationMode mode) {
  std::unique_ptr<IsolateAllocator> isolate_allocator =
      std::make_unique<IsolateAllocator>(mode);
  void* isolate_ptr = isolate_allocator->isolate_memory();
  Isolate* isolate = new (isolate_ptr) Isolate(std::move(isolate_allocator));
  ...
  return isolate;
}
```
Notice the above code is using placement new and specifying the memory address
where the new Isolate should be constructed (isolate_ptr).

The definition of IsolateAllocator can be found in `src/init/isolate-allocator.h`
and below are a few of the methods and fields that I found interesting:
```c++
class V8_EXPORT_PRIVATE IsolateAllocator final {
 public:
  void* isolate_memory() const { return isolate_memory_; }
  v8::PageAllocator* page_allocator() const { return page_allocator_; }

 private:
  Address InitReservation();
  void CommitPagesForIsolate(Address heap_reservation_address);

  // The allocated memory for Isolate instance.
  void* isolate_memory_ = nullptr;
  v8::PageAllocator* page_allocator_ = nullptr;
  std::unique_ptr<base::BoundedPageAllocator> page_allocator_instance_;
  VirtualMemory reservation_;
};
```

An Isolate has a `page_allocator()` method that returns the v8::PageAllocator
from IsolateAllocator:
```
v8::PageAllocator* Isolate::page_allocator() {
  return isolate_allocator_->page_allocator();
}
```
So, we are in `Isolate::New(IsolationAllocationMode)` and the next line is:
```c++
  std::unique_ptr<IsolateAllocator> isolate_allocator =
      std::make_unique<IsolateAllocator>(mode);
}
```

The default allocation mode will be `IsolateAllocationMode::kInV8Heap`:
```c++
  static Isolate* New(
      IsolateAllocationMode mode = IsolateAllocationMode::kDefault);
```
And the possible options for mode are:
```c++
  // Allocate Isolate in C++ heap using default new/delete operators.
  kInCppHeap,

  // Allocate Isolate in a committed region inside V8 heap reservation.
  kInV8Heap,

#ifdef V8_COMPRESS_POINTERS
  kDefault = kInV8Heap,
#else
  kDefault = kInCppHeap,
#endif
};
```

The constructor takes an AllocationMode as mentioned above.
```c++
IsolateAllocator::IsolateAllocator(IsolateAllocationMode mode) {
#if V8_TARGET_ARCH_64_BIT
  if (mode == IsolateAllocationMode::kInV8Heap) {
    Address heap_reservation_address = InitReservation();
    CommitPagesForIsolate(heap_reservation_address);
    return;
  }
#endif  // V8_TARGET_ARCH_64_BIT
```
Lets take a closer look at InitReservation (`src/init/isolate-alloctor.cc`).
```c++
Address IsolateAllocator::InitReservation() {
  v8::PageAllocator* platform_page_allocator = GetPlatformPageAllocator();
  ...
   VirtualMemory padded_reservation(platform_page_allocator,
                                    reservation_size * 2,
                                    reinterpret_cast<void*>(hint));

  ...
  reservation_ = std::move(reservation);
```
`GetPlatformPageAllocator()` can be found in `src/utils/allocation.cc`:
```c++
v8::PageAllocator* GetPlatformPageAllocator() {
  DCHECK_NOT_NULL(GetPageTableInitializer()->page_allocator());
  return GetPageTableInitializer()->page_allocator();
}
```
Now, GetPageTableInitializer is also defined in allocation.cc but is defined
using a macro:
```c++
DEFINE_LAZY_LEAKY_OBJECT_GETTER(PageAllocatorInitializer,
                                GetPageTableInitializer)
```
This macro can be found in `src/base/lazy-instance.h`:
```c++
#define DEFINE_LAZY_LEAKY_OBJECT_GETTER(T, FunctionName, ...) \
  T* FunctionName() {                                         \
    static ::v8::base::LeakyObject<T> object{__VA_ARGS__};    \
    return object.get();                                      \
  }
```
So the preprocess would expand the usage above to:
```c++
  PageAllocatorInitializer* GetPageTableInitializer() {
    static ::v8::base::LeakyObject<PageAllocatorInitializer> object{};
    return object.get();
  }
```
```c++
PageAllocatorInitializer() {
    page_allocator_ = V8::GetCurrentPlatform()->GetPageAllocator();
    if (page_allocator_ == nullptr) {
      static base::LeakyObject<base::PageAllocator> default_page_allocator;
      page_allocator_ = default_page_allocator.get();
    }
  }
```
Notice that this is calling V8::GetCurrentPlatform() and recall that a
platform is created and initialized before an Isolate can be created:
```c++
    platform_ = v8::platform::NewDefaultPlatform();
    v8::V8::InitializePlatform(platform_.get());
    v8::V8::Initialize();
```
Looking again at PageAllocator's constructor we see that it retreives the
PageAllocator from the platform. How/when is this PageAllocator created?

It is created in DefaultPlatform's constructor(src/libplatform/default-platform.cc)
which is called by `NewDefaultPlatform`:
```c++
```
DefaultPlatform::DefaultPlatform(
    int thread_pool_size, IdleTaskSupport idle_task_support,
    std::unique_ptr<v8::TracingController> tracing_controller)
    : thread_pool_size_(GetActualThreadPoolSize(thread_pool_size)),
      idle_task_support_(idle_task_support),
      tracing_controller_(std::move(tracing_controller)),
      page_allocator_(std::make_unique<v8::base::PageAllocator>()) {
  ...
}
```
PageAllocators constructor can be found in `src/base/page-allocator.cc`:
```c++
PageAllocator::PageAllocator()
    : allocate_page_size_(base::OS::AllocatePageSize()),
      commit_page_size_(base::OS::CommitPageSize()) {}
```
The allocate_page_size_ is what would be retrieved by calling (on linux):
```c++
size_t OS::AllocatePageSize() {
  return static_cast<size_t>(sysconf(_SC_PAGESIZE));
}
```
This is the number of fixed block of bytes that is the unit of the page frames
in physical memory. (_SC is a prefix for System Config parameters for the sysconf function).
A page is the unit that is stored in a page frame:
```
                             Physical Memory                                          
                         +---------------------------+                                  
      physical address 1 | PageFrame 1: page content |                                  
                         +---------------------------+                                  
      physical address 2 | PageFrame 2: page content |                                  
                         +---------------------------+                                  
             ...		     ...
``` 

commit_page_size_ is implemented like this:
```c++
size_t OS::CommitPageSize() {
  static size_t page_size = getpagesize();
  return page_size;
}
```
According to the man page of getpagesize, portable solutions should use
sysconf(_SC_PAGESIZE) instead (which is what AllocatePageSize does). I'm a little
confused about this. TODO: take a closer look at this.

The internal PageAllocator extends the public one that is declared in
`include/v8-platform.h`.
```c++
class V8_BASE_EXPORT PageAllocator
    : public NON_EXPORTED_BASE(::v8::PageAllocator) {
 private:
   const size_t allocate_page_size_;
   const size_t commit_page_size_;
```
Om my machine these are:
```console
$ ./test/heap_test --gtest_filter=HeapTest.PageAllocator
AllocatePageSize: 4096
CommitPageSize: 4096
```

PageAllocator is the interface for interacting with memory pages (memory
units of 4096 bytes). The PageAllocator implemention provided by V8 is what
interacts with the underlying operating system via the base::OS class
(src/base/platform/platform.h).

And this PageAllocator instance was created when the DefaultPlatform was
created allowing it to be accessed using GetPlatformPageAllocator:
```c++
Address IsolateAllocator::InitReservation() {
  v8::PageAllocator* platform_page_allocator = GetPlatformPageAllocator();
  ...
   VirtualMemory padded_reservation(platform_page_allocator,
                                    reservation_size * 2,
                                    reinterpret_cast<void*>(hint));

  ...
  reservation_ = std::move(reservation);
```

Notice that a VirtualMemory instance is created and the arguments passed. This
is actually part of a retry loop but I've excluded that above.
VirtualMemory's constructor can be found in `src/utils/allocation.cc`:
```c++
VirtualMemory::VirtualMemory(v8::PageAllocator* page_allocator, size_t size,
                             void* hint, size_t alignment)
    : page_allocator_(page_allocator) {
  size_t page_size = page_allocator_->AllocatePageSize();
  alignment = RoundUp(alignment, page_size);
  Address address = reinterpret_cast<Address>(
      AllocatePages(page_allocator_, hint, RoundUp(size, page_size), alignment,
                    PageAllocator::kNoAccess));
  ...
}
```
And `AllocatePages` can be found in `src/base/page-allocator.cc`:
```c++
void* PageAllocator::AllocatePages(void* hint, size_t size, size_t alignment,
                                   PageAllocator::Permission access) {
  return base::OS::Allocate(hint, size, alignment,
                            static_cast<base::OS::MemoryPermission>(access));
}
```
Which will lead us to `base::OS::Allocate` which is declared in `src/base/platform/platform.h`
and the concrete version on my system (linux) can be found in platform-posix.cc
which can be found in `src/base/platform/platform-posix.cc`:
```c++
void* Allocate(void* hint, size_t size, OS::MemoryPermission access, PageType page_type) {
  int prot = GetProtectionFromMemoryPermission(access);
  int flags = GetFlagsForMemoryPermission(access, page_type);
  void* result = mmap(hint, size, prot, flags, kMmapFd, kMmapFdOffset);
  if (result == MAP_FAILED) return nullptr;
  return result;
}
```
Notice the call to `mmap` which is a system call. 
I've written a small example of using
[mmap](https://github.com/danbev/learning-linux-kernel/blob/master/mmap.c) and
[notes](https://github.com/danbev/learning-linux-kernel#mmap-sysmmanh) which
might help to look at mmap in isolation and better understand what `Allocate`
is doing.

`prot` is the memory protection wanted for this mapping and is retrived by
calling `GetProtectionFromMemoryPermission`:
```c++
int GetProtectionFromMemoryPermission(OS::MemoryPermission access) {
  switch (access) {
    case OS::MemoryPermission::kNoAccess:
      return PROT_NONE;
    case OS::MemoryPermission::kRead:
      return PROT_READ;
    case OS::MemoryPermission::kReadWrite:
      return PROT_READ | PROT_WRITE;
    case OS::MemoryPermission::kReadWriteExecute:
      return PROT_READ | PROT_WRITE | PROT_EXEC;
    case OS::MemoryPermission::kReadExecute:
      return PROT_READ | PROT_EXEC;
  }
  UNREACHABLE();
}
```
In our case this will return `PROT_NONE` which means no access. I believe this
is so that the memory mapping can be further subdivied and different permissions
can be given later to a region using mprotect.

Next we get the `flags` to be used in the mmap call:
```c++
enum class PageType { kShared, kPrivate };

int GetFlagsForMemoryPermission(OS::MemoryPermission access,
                                PageType page_type) {
  int flags = MAP_ANONYMOUS;
  flags |= (page_type == PageType::kShared) ? MAP_SHARED : MAP_PRIVATE;
  ...
  return flags;
```
PageType in our case is `kPrivate` so this will add the `MAP_PRIVATE` flag to
the flags, in addition to `MAP_ANONYMOUS`.

So the following values are used in the call to mmap:
```c++
  void* result = mmap(hint, size, prot, flags, kMmapFd, kMmapFdOffset);
```
```console
(lldb) expr hint
(void *) $20 = 0x00001b9100000000
(lldb) expr request_size/1073741824
(unsigned long) $24 = 8
(lldb) expr kMmapFd
(const int) $17 = -1
```
`kMmapFdOffset` seems to be optimized out but we can see that it is `0x0` and
passed in register `r9d` (as it is passed as the sixth argument):
```console
(lldb) disassemble
->  0x7ffff39ba4ca <+53>:  mov    ecx, dword ptr [rbp - 0xc]
    0x7ffff39ba4cd <+56>:  mov    edx, dword ptr [rbp - 0x10]
    0x7ffff39ba4d0 <+59>:  mov    rsi, qword ptr [rbp - 0x20]
    0x7ffff39ba4d4 <+63>:  mov    rax, qword ptr [rbp - 0x18]
    0x7ffff39ba4d8 <+67>:  mov    r9d, 0x0
    0x7ffff39ba4de <+73>:  mov    r8d, 0xffffffff
    0x7ffff39ba4e4 <+79>:  mov    rdi, rax
    0x7ffff39ba4e7 <+82>:  call   0x7ffff399f000            ; symbol stub for: mmap64
```
So if I've understood this correctly this is requesting a mapping of 8GB, with
a memory protection of `PROT_NONE` which can be used to protect that mapping space
and also allow it to be subdivied by calling mmap later with specific addresses
in this mapping. These could then be called with other memory protections depending
on the type of contents that should be stored there, for example for code objects
it would be executable.

So that will return the address to the new mapping, this will return us to:
```c++
void* OS::Allocate(void* hint, size_t size, size_t alignment,
                   MemoryPermission access) {
  ...
  void* result = base::Allocate(hint, request_size, access, PageType::kPrivate);
  if (result == nullptr) return nullptr;

  uint8_t* base = static_cast<uint8_t*>(result);
  uint8_t* aligned_base = reinterpret_cast<uint8_t*>(
      RoundUp(reinterpret_cast<uintptr_t>(base), alignment));
  if (aligned_base != base) {
    DCHECK_LT(base, aligned_base);
    size_t prefix_size = static_cast<size_t>(aligned_base - base);
    CHECK(Free(base, prefix_size));
    request_size -= prefix_size;
  }
  // Unmap memory allocated after the potentially unaligned end.
  if (size != request_size) {
    DCHECK_LT(size, request_size);
    size_t suffix_size = request_size - size;
    CHECK(Free(aligned_base + size, suffix_size));
    request_size -= suffix_size;
  }

  DCHECK_EQ(size, request_size);
  return static_cast<void*>(aligned_base);
}
```
TODO: figure out what why this unmapping is done.
This return will return to `AllocatePages` which will just return to
VirtualMemory's constructor where the address will be casted to an Address.
```c++
 Address address = reinterpret_cast<Address>(
      AllocatePages(page_allocator_, hint, RoundUp(size, page_size), alignment,
                    PageAllocator::kNoAccess));
  if (address != kNullAddress) {
    DCHECK(IsAligned(address, alignment));
    region_ = base::AddressRegion(address, size);
  }
}
```
`AddressRegion` can be found in `src/base/address-region.h` and is a helper
class to manage region with a start `address` and a `size`.

After this call the VirtualMemory instance will look like this:
```console
(lldb) expr *this
(v8::internal::VirtualMemory) $35 = {
  page_allocator_ = 0x00000000004936b0
  region_ = (address_ = 30309584207872, size_ = 8589934592)
}
```

After `InitReservation` returns, we will be in `IsolateAllocator::IsolateAllocator`:
```c++
  if (mode == IsolateAllocationMode::kInV8Heap) {
    Address heap_reservation_address = InitReservation();
    CommitPagesForIsolate(heap_reservation_address);
    return;
  }
```

In CommitPagesForIsolate we have :
```c++
void IsolateAllocator::CommitPagesForIsolate(Address heap_reservation_address) {
  v8::PageAllocator* platform_page_allocator = GetPlatformPageAllocator();

  Address isolate_root = heap_reservation_address + kIsolateRootBiasPageSize;


  size_t page_size = RoundUp(size_t{1} << kPageSizeBits,
                             platform_page_allocator->AllocatePageSize());

  page_allocator_instance_ = std::make_unique<base::BoundedPageAllocator>(
      platform_page_allocator, isolate_root, kPtrComprHeapReservationSize,
      page_size);
  page_allocator_ = page_allocator_instance_.get();
```
```
(lldb) expr isolate_root
(v8::internal::Address) $54 = 30309584207872
```
So the `platform_page_allocator` is just the page allocator returned by the
platform. The `page_allocator_` for the IsolateAllocator will be a
BoundedPageAllocator which is passed the platform_page_allocator.
To recap, `InitReservation` returns the address of the memory map and this
address is passed into `CommitPagesForIsolate`.
```console
(lldb) expr isolate_root
(v8::internal::Address) $3 = 43748536877056
(lldb) expr page_size/1024
(unsigned long) $5 = 256
(lldb) expr size_t{1} << 32
(size_t) $6 = 4294967296
```
BoundedPageAllocator's constructor will create a new RegionAllocator with
the platform page allocator, the address of the memory map, 4 GB pointer compression
heap reservation size, and a page size of 256. When memory is requested for an
object it will call BoundedPageAllocator's AllocatePages implementation which
will delegate to RegionAllocator and provide memory for the object. 

Next in `CommitPagesForIsolate` we have:
```c++
  Address isolate_address = isolate_root - Isolate::isolate_root_bias();
  Address isolate_end = isolate_address + sizeof(Isolate);
  {
    Address reserved_region_address = isolate_root;
    size_t reserved_region_size =
        RoundUp(isolate_end, page_size) - reserved_region_address;

    CHECK(page_allocator_instance_->AllocatePagesAt(
        reserved_region_address, reserved_region_size,
        PageAllocator::Permission::kNoAccess));
  }
```
The above will reserve the address space of the Isolate so that is not used
for other object.

`AllocatePagesAt` will call `BoundedPageAllocator::AllocatePagesAt`:
```c++
bool BoundedPageAllocator::AllocatePagesAt(Address address, size_t size,
                                           PageAllocator::Permission access) {
  if (!region_allocator_.AllocateRegionAt(address, size)) {
    return false;
  }
  CHECK(page_allocator_->SetPermissions(reinterpret_cast<void*>(address), size,
                                        access));
  return true;
}
```

`SetPermissions` can be found in `platform-posix.cc` and looks like this:
```c++
bool OS::SetPermissions(void* address, size_t size, MemoryPermission access) {
  ...
  int prot = GetProtectionFromMemoryPermission(access);
  int ret = mprotect(address, size, prot);
```
`mprotect` is a system call that can change a memory mappings protection.


Later in `Isolate::Init` the heap will be set up:
```c++
heap_.SetUp();
```
```c++
  memory_allocator_.reset(
      new MemoryAllocator(isolate_, MaxReserved(), code_range_size_));
```
In MemoryAllocator's constructor we have a the following function call:
```c++
MemoryAllocator::MemoryAllocator(Isolate* isolate, size_t capacity,
                                 size_t code_range_size)
    : isolate_(isolate),
      data_page_allocator_(isolate->page_allocator()),
      code_page_allocator_(nullptr),
      capacity_(RoundUp(capacity, Page::kPageSize)),
      size_(0),
      size_executable_(0),
      lowest_ever_allocated_(static_cast<Address>(-1ll)),
      highest_ever_allocated_(kNullAddress),
      unmapper_(isolate->heap(), this) {
  InitializeCodePageAllocator(data_page_allocator_, code_range_size);
```
Notice that the `data_page_allocator_` is set from `isolate->page_allocator`.
Now, a MemoryAllocator will use the data_page_allocator_ which will be either
PageAllocator or a BoundedPageAllocator if pointer compression is enabled.
A MemoryAllocator instance will also have a code_page_allocator.

When an object is to be created it is done so using one of the Space implementations
that are available in the Heap class. These spaces are create by 
`Heap::SetupSpaces`, which is called from `Isolate::Init`:
```c++
void Heap::SetUpSpaces() {
  space_[NEW_SPACE] = new_space_ =
      new NewSpace(this, memory_allocator_->data_page_allocator(),
                   initial_semispace_size_, max_semi_space_size_);
  space_[OLD_SPACE] = old_space_ = new OldSpace(this);
  space_[CODE_SPACE] = code_space_ = new CodeSpace(this);
  space_[MAP_SPACE] = map_space_ = new MapSpace(this);
  space_[LO_SPACE] = lo_space_ = new OldLargeObjectSpace(this);
  space_[NEW_LO_SPACE] = new_lo_space_ =
      new NewLargeObjectSpace(this, new_space_->Capacity());
  space_[CODE_LO_SPACE] = code_lo_space_ = new CodeLargeObjectSpace(this);
  ...
```

### Spaces
A space refers to parts of the heap that are handled in different ways with
regards to garbage collection:

```console
+----------------------- -----------------------------------------------------------+
|   Young Generation                  Old Generation          Large Object space    |
|  +-------------+--------------+  +-----------+-------------+ +------------------+ |
|  |        NEW_SPACE           |  | MAP_SPACE | OLD_SPACE   | | LO_SPACE         | |
|  +-------------+--------------+  +-----------+-------------+ +------------------+ |
|  |  from_Space   | to_Space   |                                                   |
|  +-------------+--------------+                                                   |
|  +-------------+                 +-----------+               +------------------+ |
|  | NEW_LO_SPACE|                 | CODE_SPACE|               | CODE_LO_SPACE    | |
|  +-------------+                 +-----------+               +------------------+ |
|                                                                                   |
|   Read-only                                                                       |
|  +--------------+                                                                 |
|  | RO_SPACE     |                                                                 |
|  +--------------+                                                                 |
+-----------------------------------------------------------------------------------+
```
The NEW_SPACE has a space called the nursery, and one called intermediate space.
All newly allocated object go into the nursery, and if they are alive after a
marking phase they are moved into the intermediate. If the object survives multiple
gc rounds it will then be moved into the OLD_SPACE.

The `Map space` contains only map objects and they are non-movable.

The base class for all spaces is BaseSpace.


#### BaseSpace
BaseSpace is a super class of all Space classes in V8 which are spaces that are
not read-only (sealed).

What are the spaces that are defined?  
* NewSpace
* PagedSpace
* SemiSpace
* LargeObjectSpace
* others ?

BaseSpace extends Malloced:
```c++
class V8_EXPORT_PRIVATE BaseSpace : public Malloced {
```
`Malloced` which can be found in in `src/utils/allocation.h`:
```c++
// Superclass for classes managed with new & delete.
class V8_EXPORT_PRIVATE Malloced {
 public:
  static void* operator new(size_t size);
  static void operator delete(void* p);
};
```
And we can find the implemention in `src/utils/allocation.cc`:
```c++
void* Malloced::operator new(size_t size) {
  void* result = AllocWithRetry(size);
  if (result == nullptr) {
    V8::FatalProcessOutOfMemory(nullptr, "Malloced operator new");
  }
  return result;
}

void* AllocWithRetry(size_t size) {
  void* result = nullptr;
  for (int i = 0; i < kAllocationTries; ++i) {
    result = malloc(size);
    if (result != nullptr) break;
    if (!OnCriticalMemoryPressure(size)) break;
  }
  return result;
}

void Malloced::operator delete(void* p) { free(p); }
```
So for all classes of type BaseSpace when those are created using the `new`
operator, the `Malloced::operator new` version will be called. This is like
other types of operator overloading in c++. Here is a standalone example
[new-override.cc](https://github.com/danbev/learning-cpp/blob/master/src/fundamentals/new-override.cc). 


The spaces are create by `Heap::SetupSpaces`, which is called from
`Isolate::Init`:
```c++
void Heap::SetUpSpaces() {
  space_[NEW_SPACE] = new_space_ =
      new NewSpace(this, memory_allocator_->data_page_allocator(),
                   initial_semispace_size_, max_semi_space_size_);
  space_[OLD_SPACE] = old_space_ = new OldSpace(this);
  space_[CODE_SPACE] = code_space_ = new CodeSpace(this);
  space_[MAP_SPACE] = map_space_ = new MapSpace(this);
  space_[LO_SPACE] = lo_space_ = new OldLargeObjectSpace(this);
  space_[NEW_LO_SPACE] = new_lo_space_ =
      new NewLargeObjectSpace(this, new_space_->Capacity());
  space_[CODE_LO_SPACE] = code_lo_space_ = new CodeLargeObjectSpace(this);
  ...
```
Notice that this use the `new` operator and that they all extend Malloced so
these instance will be allocated by Malloced.

An instance of BaseSpace has a Heap associated with it, an AllocationSpace, and
committed and uncommited counters.

`AllocationSpace` is a enum found in `src/common/globals.h`:
```c++
enum AllocationSpace {
  RO_SPACE,      // Immortal, immovable and immutable objects,
  NEW_SPACE,     // Young generation semispaces for regular objects collected with
                 // Scavenger.
  OLD_SPACE,     // Old generation regular object space.
  CODE_SPACE,    // Old generation code object space, marked executable.
  MAP_SPACE,     // Old generation map object space, non-movable.
  LO_SPACE,      // Old generation large object space.
  CODE_LO_SPACE, // Old generation large code object space.
  NEW_LO_SPACE,  // Young generation large object space.

  FIRST_SPACE = RO_SPACE,
  LAST_SPACE = NEW_LO_SPACE,
  FIRST_MUTABLE_SPACE = NEW_SPACE,
  LAST_MUTABLE_SPACE = NEW_LO_SPACE,
  FIRST_GROWABLE_PAGED_SPACE = OLD_SPACE,
  LAST_GROWABLE_PAGED_SPACE = MAP_SPACE
};
```
There is an abstract superclass that extends BaseSpace named `Space`.

### NewSpace
A NewSpace contains two [SemiSpace](https://www.memorymanagement.org/glossary/s.html#semi.space)'s,
one name `to_space_` and the other `from_space`. These are a subdivision and are
sometimes called the nursery and intermediate spaces. 

To understand the motivation for these semi spaces we need to understand a little
about the garbage collection that V8 does.

### Scavenger
Garbage collection involves visiting all the known roots in the program, like
globals and pointers on the stack, and marking all the objects that can be
reached. When this stage is done every thing else is garbage. After this
all the marked objects are moved into the to_space.
```
ptrs   from_space (evacuation)   to_space
      +----------+            +------------+
----->|marked: * | ---------->|marked: s   |       (s=survived)
      +----------+            +------------+
      |marked:   |      ----->|marked: s   |
      +----------+     /      +------------+
----->|marked: * | ----       |            |
      +----------+            +------------+
      |marked:   |            |            |
      +----------+            +------------+
      |marked:   |            |            |
      +----------+            +------------+
```
Notice that the pointers from globals/stack are now invalid and pointing and
need to be updated to point to the `to_space`. After that is done the spaces
are switched and new object can be added to the nursery again, below the two
objects that survived the gc. 
If those object that were marked with an `s` above (that is just something I added
as I'm not sure how the marking works. I can see that every HeapObject has a
MemoryChunk has a `marking_bitmap which I should look closer at), they will
be moved to the old generation.
This type of GC is called scavenger.

### Full Mark-Compact GC
Deals with the whole heap including old generation.

The first stage is a sweep, in which the heap is searched for non-marked object
and those are pointed to by a FreeList
```
 Page 1              FreeList                        Page 1
+----------+        +--------------+		+------------+
|marked: * |---\    |    Size 1    |	--------|marked: s   |
+----------+    \   | +----------+ |   /	+------------+
|marked:   |     ---|>|__________| |  /	       -|marked: s   |
+----------+        | |__________|<|--        /	+------------+
|marked: * |--\     | |__________| |         /	|            |
+----------+   \    |    Size 2    |        /	+------------+
|marked:   |    \   | +----------+ |       /	|            |
+----------+     ---|>|__________|<|-------	+------------+
|marked:   |        | |__________| |		|            |
+----------+        | |__________| |            +------------+
                    +--------------+
```
So the free list separated by size of available object and can be reused
for object that fit into that chunk of memory. So when placing something into
the old space the free list can be searched for a slot.
The pages (page 1 and page 2 above) can be compacted sometimes. TODO: when/how?


Class NewSpace extends SpaceWithLinearArea (can be found in `src/heap/spaces.h`).

When a NewSpace instance is created it is passed a page allocator:
```c++
  space_[NEW_SPACE] = new_space_ =
      new NewSpace(this, memory_allocator_->data_page_allocator(),
                   initial_semispace_size_, max_semi_space_size_);
```


### BasicMemoryChunk
A BasicMemoryChunk has a size, a start address, and an end address. It also has
flags that describe the memory chunk, like if it is executable, pinned (cannot be
moved etc). 

A BasicMemoryChunk is created by specifying the size and the start and end
address:
```c++
BasicMemoryChunk::BasicMemoryChunk(size_t size, Address area_start,
                                   Address area_end) {
  size_ = size;
  area_start_ = area_start;
  area_end_ = area_end;
}
```
So there is nothing else to creating an instance of a BasicMemoryChunk, but
it has other protected fields like:
```c++
  Heap* heap;
  size_t allocated_bytes_;
  size_t wasted_memory_;
  std::atomic<intptr_t> high_water_mark_;

  // The space owning this memory chunk.
  std::atomic<BaseSpace*> owner_;

  // If the chunk needs to remember its memory reservation, it is stored here.
  VirtualMemory reservation_;


```


There is a static function named `Initialize` that does take a pointer to a
Heap though:
```c++
 static BasicMemoryChunk* Initialize(Heap* heap, Address base, size_t size,
                                      Address area_start, Address area_end,
                                      BaseSpace* owner,
                                      VirtualMemory reservation);
```
This can be used to create a BasicMemory chun


### MemoryChunk
Represents a memory area that is owned by a Space.

Is a struct defined in `src/heap/memory-chunk.h` (there is another MemoryChunk
struct in the namespace `v8::heap_internal` just in case you find it instead).
```c++
class MemoryChunk : public BasicMemoryChunk {

};
```

### Page
Is a class that extends MemoryChunk and has a size of 256K.

The `Page` class can be found in `src/heap/spaces.h`
```c++
class Page : public MemoryChunk {
}
```
### Heap
A heap instance has the following members:
```c++
  NewSpace* new_space_ = nullptr;
  OldSpace* old_space_ = nullptr;
  CodeSpace* code_space_ = nullptr;
  MapSpace* map_space_ = nullptr;
  OldLargeObjectSpace* lo_space_ = nullptr;
  CodeLargeObjectSpace* code_lo_space_ = nullptr;
  NewLargeObjectSpace* new_lo_space_ = nullptr;
  ReadOnlySpace* read_only_space_ = nullptr;
```

### AllocateRaw
This is a method declared in `src/heap/heap.h` and will allocate an uninitialized
object:
```c++
 V8_WARN_UNUSED_RESULT inline AllocationResult AllocateRaw(
      int size_in_bytes, AllocationType allocation,
      AllocationOrigin origin = AllocationOrigin::kRuntime,
      AllocationAlignment alignment = kWordAligned);
```
So we have the size of the object in bytes. And the other types are described
in separate sections below.

The implemementation for this method can be found in `src/heap/heap-inl.h`.
```c++
  ...
  HeapObject object;
  AllocationResult allocation;
```
This is then followed by checks or the AllocationType 
```c++
  if (AllocationType::kYoung == type) {
    ...
    } else if (AllocationType::kCode == type) {
      if (size_in_bytes <= code_space()->AreaSize() && !large_object) {
        allocation = code_space_->AllocateRawUnaligned(size_in_bytes);
      } else {
        allocation = code_lo_space_->AllocateRaw(size_in_bytes);
      }

  }
```
Lets take a closer look at `code_lo_space` which is a field in the Heap class.
```c++
 private:
  CodeLargeObjectSpace* code_lo_space_ = nullptr;
```
And it's `AllocateRaw` function looks like this:
```c++
AllocationResult CodeLargeObjectSpace::AllocateRaw(int object_size) {
  return OldLargeObjectSpace::AllocateRaw(object_size, EXECUTABLE);
}
```
OldLargeObjectSpace's AllocateRaw can be found in `src/heap/large-spaces.cc`.

There is an example of AllocateRaw in [heap_test.cc](./test/heap_test.cc).


### AllocationType
```c++
enum class AllocationType : uint8_t {
  kYoung,    // Regular object allocated in NEW_SPACE or NEW_LO_SPACE
  kOld,      // Regular object allocated in OLD_SPACE or LO_SPACE
  kCode,     // Code object allocated in CODE_SPACE or CODE_LO_SPACE
  kMap,      // Map object allocated in MAP_SPACE
  kReadOnly  // Object allocated in RO_SPACE
};
```

### AllocationOrigin
```c++
enum class AllocationOrigin {
  kGeneratedCode = 0,
  kRuntime = 1,
  kGC = 2,
  kFirstAllocationOrigin = kGeneratedCode,
  kLastAllocationOrigin = kGC,
  kNumberOfAllocationOrigins = kLastAllocationOrigin + 1
};
```

### AllocationAlignement
```c++
enum AllocationAlignment {
  kWordAligned,
  kDoubleAligned,
  kDoubleUnaligned,
  kCodeAligned
};
```

### MemoryAllocator
Is defined in `src/heap/memory-allocator.h` and is created in Heap::SetUp.

When a MemoryAllocator is created it is passes an internal isolate, a capacity,
and a code_range_size.
Each instance has a PageAllocator for non-executable pages which is called
data_page_allocator, and another that is for executable pages called
code_page_allocator.

The constructor can be found in `src/heap/memory-allocator.cc`:
```c++
MemoryAllocator::MemoryAllocator(Isolate* isolate, size_t capacity,
                                 size_t code_range_size)
    : isolate_(isolate),
      data_page_allocator_(isolate->page_allocator()),
      code_page_allocator_(nullptr),
      capacity_(RoundUp(capacity, Page::kPageSize)),
      size_(0),
      size_executable_(0),
      lowest_ever_allocated_(static_cast<Address>(-1ll)),
      highest_ever_allocated_(kNullAddress),
      unmapper_(isolate->heap(), this) {
  InitializeCodePageAllocator(data_page_allocator_, code_range_size);
}
```
We can see that the data_page_allocator_ is set to the isolates page_allocator.

Notice that the above constructor calls `InitializeCodePageAllocator` which
will set the `code_page_allocator`:
```c++
void MemoryAllocator::InitializeCodePageAllocator(
    v8::PageAllocator* page_allocator, size_t requested) {

  code_page_allocator_ = page_allocator;
  ...
  code_page_allocator_instance_ = std::make_unique<base::BoundedPageAllocator>(
      page_allocator, aligned_base, size,
      static_cast<size_t>(MemoryChunk::kAlignment));
  code_page_allocator_ = code_page_allocator_instance_.get();
```

### PageAlloctor
Is responsible for allocating pages from the underlying OS. This class is a
abstract class (has virtual members that subclasses must implement).
Can be found in `src/base/page-alloctor.h` and notice that it is in the
`v8::base` namespace and it extends `v8::PageAllocator` which can be found
in `include/v8-platform.h`.


BoundedPageAllocator is 

A Platform as a PageAllocator as a member. For the DefaultPlatform an instance
of v8::base::PageAllocator is created in the constructor.

src/base/page-allocator.h

+-----------------+
|  DefaltPlatform |
+-----------------+         +-----------------------+
| GetPageAllocator|-------> |v8::base::PageAllocator|
+-----------------+         +-----------------------+      +------------------+
			    |AllocatePages          |----->|base::OS::Allocate|
			    +-----------------------+      +------------------+



Now, lets see what goes on when we create a new Object, say of type String.
```console
(lldb) bt
* thread #1, name = 'string_test', stop reason = step in
  * frame #0: 0x00007ffff633d55a libv8.so`v8::internal::FactoryBase<v8::internal::Factory>::AllocateRawWithImmortalMap(this=0x00001d1000000000, size=20, allocation=kYoung, map=Map @ 0x00007fffffffcab8, alignment=kWordAligned) at factory-base.cc:817:3
    frame #1: 0x00007ffff633c2f1 libv8.so`v8::internal::FactoryBase<v8::internal::Factory>::NewRawOneByteString(this=0x00001d1000000000, length=6, allocation=kYoung) at factory-base.cc:533:14
    frame #2: 0x00007ffff634771e libv8.so`v8::internal::Factory::NewStringFromOneByte(this=0x00001d1000000000, string=0x00007fffffffcbd0, allocation=kYoung) at factory.cc:607:3
    frame #3: 0x00007ffff5fc1c54 libv8.so`v8::(anonymous namespace)::NewString(factory=0x00001d1000000000, type=kNormal, string=(start_ = "bajja", length_ = 6)) at api.cc:6381:46
    frame #4: 0x00007ffff5fc2284 libv8.so`v8::String::NewFromOneByte(isolate=0x00001d1000000000, data="bajja", type=kNormal, length=6) at api.cc:6439:3
    frame #5: 0x0000000000413fa4 string_test`StringTest_create_Test::TestBody(this=0x000000000048dff0) at string_test.cc:18:8
```
If we look at the call `AllocateRawWithImmortalMap` we find:
```c++
  HeapObject result = AllocateRaw(size, allocation, alignment);
```
AllocateRaw will end up in FactoryBase<Impl>::AllocateRaw:
```c++
HeapObject FactoryBase<Impl>::AllocateRaw(int size, AllocationType allocation,
                                          AllocationAlignment alignment) {
  return impl()->AllocateRaw(size, allocation, alignment);
}
```
Which will end up in `Factory::AllocateRaw`:
```c++
HeapObject Factory::AllocateRaw(int size, AllocationType allocation,
                                AllocationAlignment alignment) {
  return isolate()->heap()->AllocateRawWith<Heap::kRetryOrFail>(
      size, allocation, AllocationOrigin::kRuntime, alignment);
  
}
```

```c++
   case kRetryOrFail:
-> 297 	      return AllocateRawWithRetryOrFailSlowPath(size, allocation, origin,
   298 	                                                alignment);
```
```c++
 5165	  AllocationResult alloc;
   5166	  HeapObject result =
-> 5167	      AllocateRawWithLightRetrySlowPath(size, allocation, origin, alignment);
```

```c++
-> 5143	  HeapObject result;
   5144	  AllocationResult alloc = AllocateRaw(size, allocation, origin, alignment);
```

```c++
-> 210 	        allocation = new_space_->AllocateRaw(size_in_bytes, alignment, origin);
```
```c++
-> 95  	    result = AllocateFastUnaligned(size_in_bytes, origin);
```

```c++
AllocationResult NewSpace::AllocateFastUnaligned(int size_in_bytes,
                                                 AllocationOrigin origin) {
  Address top = allocation_info_.top();

  HeapObject obj = HeapObject::FromAddress(top);
  allocation_info_.set_top(top + size_in_bytes);
```
Where `allocation_info_` is of type `LinearAllocationArea`. Notice that a
HeapObject is created using the value of top. This is a pointer to the top
of the address of mapped memory that this space manages. And after this top
is increased by the size of the object. 

`AllocateRawWithImmortalMap`:
```c++
   818 	  HeapObject result = AllocateRaw(size, allocation, alignment);
-> 819 	  result.set_map_after_allocation(map, SKIP_WRITE_BARRIER);
   820 	  return result;
```



Now, every Space will have a region of mapped memory that it can manage. For
example the new space would have it and it would manage it with a pointer
to the top of allocated memory. To create an object it could return the top address
and increase the top with the size of the object in question. Functions that
set values on the object would have to write to the field specified by the
layout of the object. TODO: verify if that is actually what is happening.


###
Pages in the heap are of size 256K.
Large pages are any objects that are of size greater than
kMaxRegularHeapObjectSize, which is half the of the page size of the OS?

Page size:
The allocation page size is retrieved from the OS using sysconf(_SC_PAGESIZE).
So this would be the size physical memory page frames. 
This could be 4K or 64K on some systems.

AreaSize:
A PagedSpace has a field named area_size_ which is set in the constructor:
```c++
PagedSpace::PagedSpace(Heap* heap, AllocationSpace space,
                       Executability executable, FreeList* free_list,
                       LocalSpaceKind local_space_kind)
    : SpaceWithLinearArea(heap, space, free_list),
      executable_(executable),
      local_space_kind_(local_space_kind) {
  area_size_ = MemoryChunkLayout::AllocatableMemoryInMemoryChunk(space);
  accounting_stats_.Clear();
}
```
```c++
size_t MemoryChunkLayout::AllocatableMemoryInMemoryChunk(
    AllocationSpace space) {
  if (space == CODE_SPACE) {
    return AllocatableMemoryInCodePage();
  }
  return AllocatableMemoryInDataPage();
}
```
```c++
size_t MemoryChunkLayout::AllocatableMemoryInCodePage() {
  size_t memory = ObjectEndOffsetInCodePage() - ObjectStartOffsetInCodePage();
  DCHECK_LE(kMaxRegularHeapObjectSize, memory);
  return memory;
}
```

```
intptr_t MemoryChunkLayout::ObjectEndOffsetInCodePage() {
  // We are guarding code pages: the last OS page will be protected as
  // non-writable.
  return MemoryChunk::kPageSize -
         static_cast<int>(MemoryAllocator::GetCommitPageSize());
}
```
```
intptr_t MemoryAllocator::GetCommitPageSize() {
  if (FLAG_v8_os_page_size != 0) {
    DCHECK(base::bits::IsPowerOfTwo(FLAG_v8_os_page_size));
    return FLAG_v8_os_page_size * KB;
  } else {
    return CommitPageSize();
  }
}
```
```c++
size_t OS::CommitPageSize() {
#if defined(__APPLE__) && V8_TARGET_ARCH_ARM64
  static size_t page_size = kAppleArmPageSize;
#else
  static size_t page_size = getpagesize();
#endif
  return page_size;
}
```
MemoryChunk::kPageSize:
```c++
// Page size in bytes.  This must be a multiple of the OS page size.
  static const int kPageSize = 1 << kPageSizeBits;
```
```c++
// Number of bits to represent the page size for paged spaces.
#if defined(V8_TARGET_ARCH_PPC) || defined(V8_TARGET_ARCH_PPC64)
// PPC has large (64KB) physical pages.
const int kPageSizeBits = 19;
#else
const int kPageSizeBits = 18;
#endif
```
$ ./size
Size (page_size_bits: 19):524288=512
Size (page_size_bits: 18):262144=256
_SC_PAGESIZE: 4096

On my machine area_size_ is 236Kb



CodeSpace extends PagedSpace:
```c++
class CodeSpace : public PagedSpace {
 public:
  // Creates an old space object. The constructor does not allocate pages
  // from OS.
  explicit CodeSpace(Heap* heap)
      : PagedSpace(heap, CODE_SPACE, EXECUTABLE, FreeList::CreateFreeList()) {}
};
```

I think that it is possible for the OS to be configured with 4Kb pages or
64Kb pages.

```c++
  bool large_object = size_in_bytes > kMaxRegularHeapObjectSize;
  ...
  } else if (AllocationType::kCode == type) {
      if (size_in_bytes <= code_space()->AreaSize() && !large_object) {
        allocation = code_space_->AllocateRawUnaligned(size_in_bytes);
      } else {
        allocation = code_lo_space_->AllocateRaw(size_in_bytes);
      }

 if (allocation.To(&object)) {                                                 
    if (AllocationType::kCode == type && !V8_ENABLE_THIRD_PARTY_HEAP_BOOL) {    
      ...
      if (!large_object) {                                                      
        MemoryChunk::FromHeapObject(object)                                     
            ->GetCodeObjectRegistry()                                           
            ->RegisterNewlyAllocatedCodeObject(object.address());               
      }                                                                         
    }      
```
Notice that the check is `size_in_bytes` <= `code_space()->AreaSize()`
We could be in a situation where the size_in_bytes less than 128 and hence
not considered a large object, but it is larger than the available area_size
which will cause it to be created in the CodeLargeObjectSpace. And later the
!large_object will be entered causing a segment fault.

This solution is to add a check when createing the bool large_object to take
into consideration the area_size.
```c++
  size_t large_object_threshold = kMaxRegularHeapObjectSize;
  if (AllocationType::kCode == type)
    large_object_threshold = std::min(kMaxRegularHeapObjectSize,
                                      code_space()->AreaSize());
  bool large_object =
      static_cast<size_t>(size_in_bytes) > large_object_threshold;
```
This code will take the min value of the kMaxRegularHeapObjectSize and the code_space
area_size and use use that value to determine if the size of the requested object
should be considered a large object or not.

```c++
size_t area_size() const {
    return static_cast<size_t>(area_end() - area_start());
  }
```

### Pagesize in V8
V8 heap pages are `256kb`. For an executable object 3 OS pages are allocated,
which normally means `3*4kb` if the page size is `4kb`, but could also be
`3*64kb` if the system is `aarch64` and I think this is also true for some
PowerPC systems.

```
4 kb page size                        64 kb page size
3*4096=12288=12kb                     3*65536=192kb
256-12=244                            256-192=64

244kb available for alloction         64kb available for allocation
```

Now, an object is considered large if it is larger than
`kMaxRegularHeapObjectSize` which is half of the V8 page size, `256/2=128`.
So, when 64kb pages sizes are used an object me be considered to be able to
be stored in a normal sized page even though it might be between 64-128, but
there is only 64kb available for allocation.

