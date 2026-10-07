---
layout: page
title: Memory Pool Project Report
description: 
img: /assets/img/proj1_cover.jpg
importance: 1
category: work
related_publications: false
---

> A Lock-free Allocator for Small Objects optimized for C++17

👉 [1st Version GitHub Repo Link](https://github.com/Leakingpipe/mymempool/tree/main)

## Prerequisites:
To understand memory pool better, let's review <code>STL::allocator</code> first

### STL::allocator
It is used to manage memory allocation in C++ standard library.
In order to understand how an allocator is defined and how does it work, I wrote one myself:
```cpp
#include <iostream>
#include <memory>
#include <vector>

template <typename T>
class MyAllocator
{
public:
    using value_type = T;
    using pointer = T*;
    using const_pointer = const T*;
    using reference = T&;
    using const_reference = const T&;
    using size_type = std::size_t;
    using difference_type = std::ptrdiff_t;

    //constructor
    MyAllocator() noexcept {} 
    template <typename U> 
    MyAllocator(const MyAllocator<U>&) noexcept {}

    // allocate memory
    T* allocate(std::size_t n){
        std::cout << "[Allocate]" << n << "element\n";
        return static_cast<T*>(::operator new(n * sizeof(T)));
    }

    // release memory
    void deallocate(T* p, std::size_t n) {
        std::cout << "[Deallocate] " << n << " element(s)\n";
        ::operator delete(p);
    }

    // rebinding forming new type
    template <typename U>
    struct rebind {
        using other = MyAllocator<U>;
    };
};

template <typename T, typename U>
bool operator==(const MyAllocator<T>&, const MyAllocator<U>&) { return true; }
template <typename T, typename U>
bool operator!=(const MyAllocator<T>&, const MyAllocator<U>&) { return false; }
```
This allocator fulfills the standard allocator requirements. It declares different type aliases and provides type conversion and rebind template to support rebinding to different element types. 

![](/assets/img/MemPool_allocator.png)
It works! :D

---
### SGI STL allocator
Before we move on to creating the project by ourselves, let's look at more examples of allocators.

STL allocator are usually devided into two levels. If the chunk is bigger that 128 bytes, then first-level allocator will be called.

#### first-level allocator
The process usually gose like this: It first tries to allocate memory directly. If the allocation succeeds, it immediately returns the memory block. However, if the allocation fails, it triggers the out-of-memory handling mechanism. The allocator checks whether a custom handler has been registered to release memory in low-memory situations.
If no handler is set, the program will throw an exception and terminate. But if a handler is present, it will be executed in an attempt to release memory and reattempt the allocation. This process repeats until the allocation eventually succeeds.
Nevertheless, there is a potential issue: if the handler is not well-designed — in other words, it fails to actually release any usable memory and simply returns without taking meaningful action — it may lead to an infinite loop.
To avoid this risk, the SGI STL by default does not set this handler (i.e., it is null). As a result, if memory allocation fails, the program will immediately terminate.This approach ensures that the program fails fast and avoids unpredictable behavior caused by faulty recovery logic.
If allocation finally succeeds, the memory block is returned.
```cpp
typedef void(*MALLOCALLOC)();           //rename void (*)() as MALLOCALLOC
template<int inst>
class _MallocAllocTemplate //offers Allocate /De allocate 
{
private:
       static void* _OomMalloc(size_t);       //called when malloc failed
       static MALLOCALLOC _MallocAllocOomHandler;         // store OOM handler
public:
       static void* _Allocate(size_t n)                        // allocate n bytes of memory
       {
              void *result=0;
              result = malloc(n);                 //call malloc to allocate memory
              if (0 == result)                    // call OOM_malloc if malloc failed
                     _OomMalloc(n);
              return result;                      // return the address of the allocated memory
       }

       static void _DeAllocate(void *p)                // free the memory
       {
              free(p);
       }

       static MALLOCALLOC _SetMallocHandler(MALLOCALLOC f)    // to set OOM handler
       {
              MALLOCALLOC old = _MallocAllocOomHandler;
              _MallocAllocOomHandler = f;              // pass function f, as a emergency function if memory allocation failed
              return old;
       }
};

template<int inst>
void(* _MallocAllocTemplate<inst>::_MallocAllocOomHandler)()=0;    // set default as not cope with out of memory mechanism
template<int inst>
void* _MallocAllocTemplate<inst>::_OomMalloc(size_t n)
{
       MALLOCALLOC _MyMallocHandler;     // define a function pointer
       void *result;               
       while (1)
       {
              _MyMallocHandler = _MallocAllocOomHandler;
              if (0 == _MyMallocHandler)                  // handler is not set -> failed
                     throw std::bad_alloc();                  // throw an exception
              (*_MyMallocHandler)();                 // call handler to release memory
              if (result = malloc(n))                // try allocate again
                     break;
       }
       return result;                              // return if succeed
}
typedef _MallocAllocTemplate<0> malloc_alloc;
```

#### second-level allocator
The second-level allocator handles small memory allocations (≤128 bytes) using a memory pool and multiple free lists to improve speed and reduce fragmentation. When memory is requested, it first checks the appropriate free list for a block of the required size. If available, it returns the block directly. If not, it requests a large chunk of memory from the first-level allocator (which uses malloc), splits it into small blocks, stores them in the free list, and returns one to the user. This layered design combines efficient small object reuse with large block allocation from the system.

The second-level allocator places the large memory blocks allocated from first-level allocator into a memory pool (pointed to by _start_free and _end_free), and then slices out small blocks from this pool as needed and attaches them to the linked lists of multiple free_list[]. 

##### Logic Flow of the second-level allocator:
__Size check:__ When memory is requested, the allocator first checks if the requested size exceeds the maximum threshold (typically 128 bytes). If the size is greater than 128 bytes, the allocator delegates the request to the first-level allocator (<code>malloc_alloc</code>), which uses <code>malloc</code> directly.
__Free list lookup:__ The requested size is rounded up to the nearest multiple of 8 bytes to maintain alignment. Then, the allocator checks the corresponding free list for a block of that size. If the free list has available blocks, the allocator pops one from the list and returns it. If the free list is empty, the allocator calls <code>_Refill()</code>.
__Refill mechanism:__ <code>_Refill(n)</code> tries to allocate multiple blocks of size <code>n</code> at once (typically 20) to improve future allocation efficiency. It calls <code>_ChunkAlloc(size, nobjs)</code> to cut a large chunk of memory into small blocks. One block is returned to the user; the remaining are linked into the appropriate free list.
__Chunk allocation(<code>_ChunkAlloc</code>):__ This function attempts to cut memory from the internal memory pool defined by <code>_start_free</code> and <code>_end_free</code>. If enough memory exists, it returns a batch of blocks. If not enough memory remains, it attempts to allocate a new large chunk from the first-level allocator and updates the internal memory pool. If this also fails, it may search other free lists for recyclable blocks or fall back to the first-level allocator's OOM handler.
__Deallocation:__ When memory is returned, it is pushed back to the appropriate free list, using the block’s starting address as a next pointer in the linked list structure.

```cpp
enum { _ALIGN = 8 };              // Perform memory operations in multiples of the base value 8
enum { _MAXBYTES = 128 };        // the largest chunk in the free list is 128 byte
enum { _NFREELISTS = 16 };       // the length of the free list, == _MAXBYTES/_ALIGN
template <bool threads, int inst>
class _DefaultAllocTemplate
{
       union _Obj                      // free list node type
       {
              _Obj* _freeListLink;         // this pointer points to free list node
              char _clientData[1];          //this client sees
       };
private:
       static char* _startFree;             // head pointer to the memory pool
       static char* _endFree;               // tail pointer
       static size_t _heapSize;              // record the size of memory the pool has applied from the system
       static _Obj* volatile _freeList[_NFREELISTS];    //free list
private:
       static size_t _GetFreeListIndex(size_t bytes)   // get the position of the byte in the free list and transform the size into its index
       {
              return (bytes +(size_t) _ALIGN - 1) / (size_t)_ALIGN - 1;     
       }

       static size_t _GetRoundUp(size_t bytes)        // Round up any byte size to a multiple of 8 for alignment
       {
              return (bytes + (size_t)_ALIGN - 1)&(~(_ALIGN-1));
       }
       
       static void* _Refill(size_t n);          // apply about 20 chuncks with each size of n bytes, return the first one
       static char* _chunkAlloc(size_t size,int& nobjs);    //Allocate memory from the memory pool for nobjs objects, each of size size
public:
       static void* Allocate(size_t n);      //External allocation entry point, n larger than 0
       static void DeAllocate(void *p,size_t n);        // n != 0
};
template<bool threads,int inst>
char* _DefaultAllocTemplate<threads,inst>::_startFree = 0;        //head pointer of the pool
template<bool threads, int inst>
char* _DefaultAllocTemplate<threads, inst>::_endFree=0;           // end pointer of the pool
template<bool threads, int inst>
size_t _DefaultAllocTemplate<threads, inst>::_heapSize = 0;              // record how much memory that the pool has applied from the system
template<bool threads, int inst>
typename _DefaultAllocTemplate<threads, inst>::_Obj* volatile      //'typename' indicates that '_Obj*' is a type dependent on the template parameters.
_DefaultAllocTemplate<threads, inst>::_freeList[_NFREELISTS] = {0};    //free list
 
template<bool threads, int inst>
void* _DefaultAllocTemplate<threads, inst>::Allocate(size_t n)    // allocate memory
{
       void *ret;
       //First check whether the requested memory size is greater than 128 bytes
       if (n>_MAXBYTES)      // If greater than _MAXBYTES, treat it as a large memory block and use the first-level allocator directly
       {
              ret = malloc_alloc::_Allocate(n);
       }
       else       // Otherwise, look for available space in the free list
       {
              _Obj* volatile *myFreeList = _freeList+_GetFreeListIndex(n);  // Let myFreeList point to the free list corresponding to n rounded up to the nearest multiple of 8
              _Obj* result = *myFreeList;
              if (result == 0)  // If no memory is linked at this node, request memory from the memory pool
              {
                     ret = _Refill(_GetRoundUp(n));      //  Request memory from the memory pool
              }
              else            //Memory has been found in the free list
              {
                     *myFreeList= result->_freeListLink;      //Put the address of the second memory block back into the free list
                     ret = result;
              }
       }
       return ret;
}

template<bool threads, int inst>
void _DefaultAllocTemplate<threads, inst>::DeAllocate(void *p, size_t n)
{
       //check the size of the memory block
       if (n > _MAXBYTES)  //If n exceeds the maximum block size managed by the free list, call the first-level deallocator directly
       {
              malloc_alloc::_DeAllocate(p);
       }
       else        //Recycle the memory block back into the free list
       {
              _Obj* q = (_Obj*)p;
              _Obj* volatile *myFreeList = _freeList + _GetFreeListIndex(n);
              q->_freeListLink = *myFreeList;
              *myFreeList = q;
       }
}
 
template<bool threads,int inst>
void* _DefaultAllocTemplate<threads, inst>::_Refill(size_t n)     // 'n' indicates the number of bytes to allocate
{
       int nobjs = 20;           // When requesting memory from the pool, allocate 20 at once
       char* chunk = _chunkAlloc(n,nobjs);    // Since the free list is currently empty, request memory from the pool and add the remaining objects to the free list
       if (1 == nobjs)          // Only one object was allocated
       {
              return chunk;
       }
       _Obj* ret = (_Obj*)chunk;                  // Return the first allocated object as the return value
       _Obj* volatile *myFreeList = _freeList+ _GetFreeListIndex(n);
       *myFreeList =(_Obj*)(chunk+n);             // Add the second object's address to the free list
       _Obj* cur= *myFreeList;
       _Obj* next=0;
       cur->_freeListLink = 0;
       for (int i = 2; i < nobjs; ++i)             // Link the remaining blocks into the free list
       {
              next= (_Obj*)(chunk + n*i);
              cur->_freeListLink = next;
              cur = next;
       }
       cur->_freeListLink = 0;
       return ret;
}

template<bool threads, int inst>
char* _DefaultAllocTemplate<threads, inst>::_chunkAlloc(size_t size, int& nobjs)  //Request memory from the system
{
       char* result = 0;
       size_t totalBytes = size*nobjs;        //Total number of bytes requested
       size_t leftBytes = _endFree - _startFree;      //Remaining bytes in the memory pool
       if (leftBytes>=totalBytes)     //If the remaining pool memory is enough, allocate directly
       {
              result = _startFree;
              _startFree += totalBytes;
              return result;
       }
       else if (leftBytes>size)         //If only enough for fewer objects
       {
              nobjs=(int)(leftBytes/size);    // Adjust the number of objects accordingly
              result = _startFree;
              _startFree +=(nobjs*size);
              return result;
       }
       else            //Not even enough for one object
       {
              size_t NewBytes = 2 * totalBytes+_GetRoundUp(_heapSize>>4);       //Calculate new memory pool size
              if (leftBytes >0)  //Recycle remaining memory into the free list
              {
              {
                     _Obj* volatile *myFreeList = _freeList + _GetFreeListIndex(leftBytes);
                     ((_Obj*)_startFree)->_freeListLink = *myFreeList;
                     *myFreeList = (_Obj*)_startFree;
              }
              
              //Try to allocate new memory from the system
              _startFree = (char*)malloc(NewBytes);
              if (0 == _startFree)
              {
                     // Search the free list for a larger block to use as the new pool if allocate failed
                     for (size_t i = size; i <(size_t)_MAXBYTES;i+=(size_t)_ALIGN)
                     {
                           _Obj* volatile *myFreeList = _freeList + _GetFreeListIndex(i);
                           _Obj* p =*myFreeList;
                           if (NULL != p)       //Found a block in the free list
                           {
                                  _startFree =(char*)p;                  
                                  //Remove this block from the free list and assign it to the memory pool
                                  *myFreeList = p->_freeListLink;
                                  _endFree = _startFree + i;
                                  return _chunkAlloc(size, nobjs);  //Retry allocation with updated pool
                           }
                     }
                     // If still no memory found, fall back to the first-level allocator which has its own out-of-memory handling mechanism (may throw)
                     _endFree = NULL;
                     _startFree=(char*)malloc_alloc::_Allocate(NewBytes);
              }      
              //Allocation succeeded, update heapSize, and _endFree
              _heapSize += NewBytes;
              _endFree = _startFree + NewBytes;
              return _chunkAlloc(size, nobjs);             //Retry allocation with updated pool
       }
}
 
typedef _DefaultAllocTemplate<0,0>  default_alloc;
```

To put it in a simple way, <code>free_list[i]</code> works like a shelf in a store, <code>_Refill(n)</code> acts as a worker that restocks the shelf, and <code>_chunkAllocate()</code> is the way to acquire new supplies. <code>_startFree</code> and <code>_endFree</code> indicate the remaining space in the inventory (memory pool).

As for <code>_chunkAlloc()</code> 
- if there's enough space:
   - If `_endFree - _startFree` ≥ `nobjs * size`
   - Directly return the memory block and update `_startFree`

- if there's only part enough remaining space:
   - Can’t get all `nobjs`, but can get at least one block
   - Adjust `nobjs`, return what’s available

- not enough space：
   - Not even one block fits
   - Recycle leftover memory to free list
   - Request more memory from system via `malloc()`
   - If system fails, try scavenging other free lists
   - Finally fallback to first-level allocator (may throw)

### Spinlock
A spinlock is a lightweight synchronization primitive used in multithreaded programming to protect shared resources from concurrent access.
Unlike traditional mutexes, a spinlock does not put the thread to sleep when the lock is already held. Instead, the thread "spins" in a loop, repeatedly checking until the lock becomes available.
```cpp
while (lock.test_and_set(std::memory_order_acquire)) {
    // busy wait (spin)
}
```
<code>test_and_set()</code> is an atomic operation that attempts to set the lock to true and returns its previous value. If the lock was already true (held), the thread keeps spinning. Once it reads false, it means the lock is free, and the thread takes it.

#### When to use spinlocks?
1. Short critical sections where holding time is minimal;
2. High-frequency, low-contention scenarios;
3. Systems where thread context switching is expensive (e.g., real-time or low-latency systems). 

#### Advantages of spinlocks:
1. Fast and low-overhead when the lock is uncontended;
2. No need for system calls or thread suspension;
3. Ideal for situations where lock is expected to be released very quickly.

#### Disadvantages:
1. Wastes CPU cycles during spinning (especially under contention);
2. Not suitable for long blocking operations;
3. Can cause CPU starvation if not designed carefully.

#### Spinlock vs. Mutex
| Features | Spinlock | Mutex |
| :---:    | :---:    | :---: |
|Blocking  |No, busy waitiing|Yes, thread sleeps|
|CPU usage | High under contention| Lower |
|Overhead|Low(no context switch)|High(system-level)|
|Use case | Fast, shrot operations|Long or IO-bound operations|

## Code framework:(provided by instructor)
This code frame is provided by Carl at https://github.com/youngyangyang04/memory-pool

### Version 1
The hash bucket maps requested byte sizes to corresponding MemoryPools. Each MemoryPool batches fixed-size 4096-byte Blocks from the system, but only slices them into Slots of a single SlotSize.
__Allocation process:__
When <code>newElement</code> is called, it first checks the freeList — if there is an available Slot, it returns one in O(1) time. 
If the list is empty, a new Slot is carved from curSlot. 
Once the current Block is cut up, <code>_chunkAlloc</code> is triggered to request a new Block,
which is linked via a head pointer stored in each Block, and <code>firstBlock</code> records the chain head.
__Deallocation process:__
When <code>deleteElement</code> is called, the returned Slot is turned into a pointer to the previous head and pushed back onto the freeList for future reuse. To maintain page alignment, the end of a Block may contain a padding area.

#### Optimization: lock-free datastructure
This improvement tries to use a lock-free data structure to manage free slots. 

This optimization 
1. first introduces atomic operations by replacing regular pointers with <code>std::atomic<Slot*> </code>, and implements lock-free Compare-And-Swap (CAS) to ensure thread safety.
2. Memory ordering are applied to guarantee consistency and visibility across threads.
The free slots are managed using a stack-like structure, ensuring that each push and pop operation only manipulates the top element, enabling fast and contention-free access.
3. Finally, the mutex protecting free slot operations is removed to reduce lock contention (as these operations are frequent), while the mutex for block allocation is preserved, since it is triggered infrequently and does not significantly impact performance.

### Version 2
In the second version of the memory pool project, a three-tier caching architecture was introduced, similar to the design adopted by mainstream allocators such as tcmalloc and jemalloc.
The structure consists of __ThreadCache__, __CentralCache__, and __PageCache__, each responsible for thread-local storage, centralized coordination, and system memory management, respectively.
This architecture improves the efficiency of small-object memory allocation in multithreaded environments, while avoiding frequent allocation and deallocation requests to the operating system.

#### 1. ThreadCache
First, ThreadCache provides each thread with an independent cache for small objects, effectively avoiding inter-thread lock contention. It's like having a private memory drawer under each thread’s desk — no locking is needed, so allocation becomes much faster.

Internally, it maintains a <code>FreeList[]</code> array, where each slot holds a singly linked list that manages memory blocks of a specific size. When a slot runs out of memory, the thread will batch-fetch new blocks from the __CentralCache__. If there are too many unused blocks, they will be returned in bulk.

The <code>ThreadCache</code> class uses a thread-local singleton instance (<code>thread_local</code>)， so each thread owns its private cache that cannot be accessed by others — just like a personal drawer.
It defines four key methods:
1. <code>allocate()</code> and <code>deallocate()</code> for allocating and releasing small objects.
2. <code>fetchFromCentralCache()</code> and <code>returnToCentralCache()</code> for bulk acquiring and returning memory blocks. Additionally, it maintains an array <code>freeList_[]</code> to manage free memory of various sizes using linked lists.

#### 2. CentralCache
CentralCache acts as the middle layer between ThreadCache and PageCache.
Its main responsibility includes:
- Managing free lists for different object sizes;
- Allocating memory blocks from PageCache in batches;
- Providing allocation and recycling services to multiple ThreadCaches;
- Supporting thread-safe operations using lock-free atomic pointers and lightweight spinlocks.

The `centralFreeList_` is an array of atomic pointers to manage the free memory blocks,  
while the `locks_` array contains atomic flags for protecting each individual free list.

The main implementation includes four steps:
1. Try to acquire the lock (Spinlock):It uses `test_and_set(memory_order_acquire)` to acquire a lock on the corresponding free list. Just like flipping a "busy" sign before accessing a shared file folder, making sure only one person can open it at a time.
2. Attempt to load from the central free list: It calls `.load(memory_order_relaxed)` to check if there’s any memory available. It's like peeking in to the drawing without worrying if it's up-to-date, just a quick glance to see if something's there. 
3. If empty, fetch from PageCache: Calls `fetchFromPageCache()` to get a large chunk and splits it into small blocks. It's like when the drawer is empty, you go to the warehouse and bring back a whole box to restock.
4. Update the free list head and return: Uses `.store(memory_order_release)` to update the free list with the new head. Just like after restocking, you properly put the new box in place and lable it so others know it's ready.

__Design Feature: Batch Allocation__
In the memory pool design, `CentralCache` does not request memory from `PageCache` one page at a time. Instead, it batches the allocation using a constant value (e.g., `SPAN_PAGES = 8`), meaning it fetches 8 pages in one go.
This improves performance by:
- Reducing the number of calls to `PageCache`
- Increasing allocation efficiency
- Lowering lock contention during multi-threaded access

It's like imagining you are part of a writing club. Instead of asking the supply room for one sheet of paper every time you write, you now take 8 pages at once and keep them in your drawer. This way, you save time and avoid crowding around the supply desk. That’s exactly what batch allocation achieves in memory management.

__ALSO__ in this memory pool design, different memory sizes are categorized by fixed alignment (8 bytes). Instead of maintaining a separate list for every possible byte size, we use an index to represent ranges (e.g., 1–8B → index 0, 9–16B → index 1, and so on). The getIndex(size) function calculates the appropriate index for any memory size, enabling the CentralCache to quickly locate the corresponding free list. This approach simplifies memory management, improves lookup efficiency, and ensures consistent alignment for reuse.

#### 3. PageCache
PageCache is like a warehouse manager, they manage the big chunk of memory(4kb per page) that are requested from the os.
Its main responsibility includes:
- Requesting large memory chunks (via `mmap`) from the system
- Managing and organizing memory in units of 4KB pages
- Splitting large memory blocks into smaller spans and assigning them to `CentralCache`
- Handling span merging and memory recycling

It uses:
- `Span` structs to represent continuous page blocks
- `freeSpans_` to manage available page blocks grouped by size
- `spanMap_` to track which address belongs to which span

### Version 3
Comparing to version two, the improvement is that the ThreadCache uses batch allocation from the CentralCache.

## My Inplementation: WIP...
This part of the note explains the way I implement according to the basic principle that the instructor has given.
Including: 
1. Implementation of header files and main cpp files;
2. My comments;
3. Differences and similarities from the original project;
4. Test results.

### 📘 Disclaimer
This project is based on my understanding and reproduction of a memory pool design concept, originally learned from a paid course.
While the core allocation logic is standard in C++ memory management discussions, the implementation, comments, file structure and documentation here reflect my personal learning process and re-implementation.
This repository is intended for educational and portfolio purposes only, not for commercial distribution.

![](/assets/img/MemPool_v1_result.png)
