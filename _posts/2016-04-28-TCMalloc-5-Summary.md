---
layout: post
title: TCMalloc源码学习-5-总结
---

前面几篇分别介绍了TCMalloc的整体，PageHeap，CentralFreeList和ThreadCache，这一篇介绍一下最上层的用户接口，其实只要理解了前面的所有结构，用户接口的实现就非常好理解了

## do_malloc
用户调用malloc后，主要做的事情都在do_malloc里，直接上代码：

```cpp

inline void* do_malloc(size_t size) {
  void* ret = NULL;

  // The following call forces module initialization
  //获取本线程的ThreadCache对象
  ThreadCache* heap = ThreadCache::GetCache();
  //如果是小内存申请
  if (size <= kMaxSize) {
    //通过查策略找到对应的size和cl
    size_t cl = Static::sizemap()->SizeClass(size);
    size = Static::sizemap()->class_to_size(cl);

    if ((FLAGS_tcmalloc_sample_parameter > 0) && heap->SampleAllocation(size)) {
      ret = DoSampledAllocation(size);
    } else {
      // The common case, and also the simplest.  This just pops the
      // size-appropriate freelist, after replenishing it if it's empty.
      //调用本线程ThreadCache的Allocate接口从缓存中申请内存，具体实现上一篇有讲
      ret = CheckedMallocResult(heap->Allocate(size, cl));
    }
  } else {
    //如果是大内存申请，则直接调用do_malloc_pages从PageHeap来申请
    ret = do_malloc_pages(heap, size);
  }
  if (ret == NULL) errno = ENOMEM;
  return ret;
}

// Helper for do_malloc().
inline void* do_malloc_pages(ThreadCache* heap, size_t size) {
  void* result;
  bool report_large;
  //通过size算出需要多少页
  Length num_pages = tcmalloc::pages(size);
  //得到实际要实际分配的大小
  size = num_pages << kPageShift;

  if ((FLAGS_tcmalloc_sample_parameter > 0) && heap->SampleAllocation(size)) {
    result = DoSampledAllocation(size);

    SpinLockHolder h(Static::pageheap_lock());
    report_large = should_report_large(num_pages);
  } else {
    //锁住PageHeap
    SpinLockHolder h(Static::pageheap_lock());
    //从PageHeap中申请需要的页数
    Span* span = Static::pageheap()->New(num_pages);
    //将返回的大内存的首页号缓存到PageHeap的pagemap_cache，对应的cl是0，这个0会
    //在将来还这块内存的时候表明这是个大内存
    result = (span == NULL ? NULL : SpanToMallocResult(span));
    report_large = should_report_large(num_pages);
  }

  if (report_large) {
    ReportLargeAlloc(num_pages, result);
  }
  return result;
}
```

还是十分简单的，再来看看释放内存的实现

## do_free

```cpp

// The default "do_free" that uses the default callback.
inline void do_free(void* ptr) {
  //实际调用
  return do_free_with_callback(ptr, &InvalidFree);
}

// This lets you call back to a given function pointer if ptr is invalid.
// It is used primarily by windows code which wants a specialized callback.
inline void do_free_with_callback(void* ptr, void (*invalid_free_fn)(void*)) {
  if (ptr == NULL) return;
  if (Static::pageheap() == NULL) {
    // We called free() before malloc().  This can occur if the
    // (system) malloc() is called before tcmalloc is loaded, and then
    // free() is called after tcmalloc is loaded (and tc_free has
    // replaced free), but before the global constructor has run that
    // sets up the tcmalloc data structures.
    (*invalid_free_fn)(ptr);  // Decide how to handle the bad free request
    return;
  }
  //通过ptr算出PageID
  const PageID p = reinterpret_cast<uintptr_t>(ptr) >> kPageShift;
  Span* span = NULL;
  //从PageHeap的pagemap_cache中找到这个page对应的cl，如果没找到就返回0
  size_t cl = Static::pageheap()->GetSizeClassIfCached(p);

  if (cl == 0) {
    //这里cl等于0有两种情况，一种是页p没有在pagemap_cache中找到默认返回0，
    //另一种是页p对应的是大内存的首页，所以当初存的cl就是0
    span = Static::pageheap()->GetDescriptor(p);
    if (!span) {
      // span can be NULL because the pointer passed in is invalid
      // (not something returned by malloc or friends), or because the
      // pointer was allocated with some other allocator besides
      // tcmalloc.  The latter can happen if tcmalloc is linked in via
      // a dynamic library, but is not listed last on the link line.
      // In that case, libraries after it on the link line will
      // allocate with libc malloc, but free with tcmalloc's free.
      (*invalid_free_fn)(ptr);  // Decide how to handle the bad free request
      return;
    }
    cl = span->sizeclass;
    Static::pageheap()->CacheSizeClass(p, cl);
  }
  if (cl != 0) {
    //不等于0说明归还的是86种之一的小内存
    ASSERT(!Static::pageheap()->GetDescriptor(p)->sample);
    //得到本线程的ThreadCache对象
    ThreadCache* heap = GetCacheIfPresent();
    if (heap != NULL) {
      //如果有就放进线程缓存
      heap->Deallocate(ptr, cl);
    } else {
      //如果没有就返回CentralFreeList
      // Delete directly into central cache
      tcmalloc::SLL_SetNext(ptr, NULL);
      Static::central_cache()[cl].InsertRange(ptr, ptr, 1);
    }
  } else {
    //等于0，大内存，直接还给PageHeap
    SpinLockHolder h(Static::pageheap_lock());
    ASSERT(reinterpret_cast<uintptr_t>(ptr) % kPageSize == 0);
    ASSERT(span != NULL && span->start == p);
    if (span->sample) {
      StackTrace* st = reinterpret_cast<StackTrace*>(span->objects);
      tcmalloc::DLL_Remove(span);
      Static::stacktrace_allocator()->Delete(st);
      span->objects = NULL;
    }
    //还给PageHeap
    Static::pageheap()->Delete(span);
  }
}
```

## 总结
TCMalloc的源码学习收获还是不少的，不过里面还是有一些疑惑的，过阵子再回头来看看没准会有新的理解
