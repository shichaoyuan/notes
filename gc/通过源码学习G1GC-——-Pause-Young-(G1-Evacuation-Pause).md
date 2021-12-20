介绍 G1GC 的文章很多，但是读完之后都感觉写得不够透彻，所以笔者决定直接读 G1 的源代码，这一系列文章就是 G1 代码的阅读笔记。

由于 gdb/lldb 的功力不足，所以选择了日志打印比较详细的 [openjdk-jdk10](https://github.com/AdoptOpenJDK/openjdk-jdk10)。

先从最常见的 Pause Young (G1 Evacuation Pause) 开始吧。

## 0. 封装类

Evacuation Pause 的逻辑封装在 `VM_G1IncCollectionPause` 类中，其中的处理逻辑包含了 Young GC，Mixed GC 和 Initial Mark。

[VM_G1IncCollectionPause](https://github.com/AdoptOpenJDK/openjdk-jdk10/blob/0f7bcfb753bcb6d8057562da74466140737f4a0d/src/hotspot/share/gc/g1/vm_operations_g1.hpp#L80)
```cpp
// share/gc/g1/vm_operations_g1.hpp#L80
class VM_G1IncCollectionPause: public VM_G1OperationWithAllocRequest {
private:
  bool         _should_initiate_conc_mark;
  bool         _should_retry_gc;
  double       _target_pause_time_ms;
  uint         _old_marking_cycles_completed_before;
public:
  VM_G1IncCollectionPause(uint           gc_count_before,
                          size_t         word_size,
                          bool           should_initiate_conc_mark,
                          double         target_pause_time_ms,
                          GCCause::Cause gc_cause);
  virtual VMOp_Type type() const { return VMOp_G1IncCollectionPause; }
  virtual bool doit_prologue();
  virtual void doit();
  virtual void doit_epilogue();
  virtual const char* name() const {
    return "garbage-first incremental collection pause";
  }
  bool should_retry_gc() const { return _should_retry_gc; }
};
```

## 1. Pause Young (G1 Evacuation Pause) 触发流程

我们暂且忽略其它情况，只是按照 Pause Young (G1 Evacuation Pause) 追踪代码执行。

触发 Pause Young (G1 Evacuation Pause) 的大致流程如下：
```
Universe::heap()->mem_allocate(size, &gc_overhead_limit_was_exceeded);
    attempt_allocation(word_size, &gc_count_before, &gclocker_retry_count);
        attempt_allocation_slow(word_size, context, gc_count_before_ret, gclocker_retry_count_ret);
            do_collection_pause(word_size, gc_count_before, &succeeded, GCCause::_g1_inc_collection_pause);
```

如果两次尝试分配内存失败，并且满足 `should_try_gc` 的条件，那么就会执行 `VM_G1IncCollectionPause`。

[G1CollectedHeap::do_collection_pause](https://github.com/AdoptOpenJDK/openjdk-jdk10/blob/0f7bcfb753bcb6d8057562da74466140737f4a0d/src/hotspot/share/gc/g1/g1CollectedHeap.cpp#L2617)
```cpp
// share/gc/g1/g1CollectedHeap.cpp#L2617
HeapWord* G1CollectedHeap::do_collection_pause(size_t word_size,
                                               uint gc_count_before,
                                               bool* succeeded,
                                               GCCause::Cause gc_cause) {
  assert_heap_not_locked_and_not_at_safepoint();
  VM_G1IncCollectionPause op(gc_count_before,
                             word_size,
                             false, /* should_initiate_conc_mark */
                             g1_policy()->max_pause_time_ms(),
                             gc_cause);

  op.set_allocation_context(AllocationContext::current());
  VMThread::execute(&op);

  HeapWord* result = op.result();
  bool ret_succeeded = op.prologue_succeeded() && op.pause_succeeded();
  assert(result == NULL || ret_succeeded,
         "the result should be NULL if the VM did not succeed");
  *succeeded = ret_succeeded;

  assert_heap_not_locked();
  return result;
}
```

具体逻辑在 `doit` 方法中：
[VM_G1IncCollectionPause::doit](https://github.com/AdoptOpenJDK/openjdk-jdk10/blob/0f7bcfb753bcb6d8057562da74466140737f4a0d/src/hotspot/share/gc/g1/vm_operations_g1.cpp#L90)
```cpp
// share/gc/g1/vm_operations_g1.cpp#L90
void VM_G1IncCollectionPause::doit() {
  G1CollectedHeap* g1h = G1CollectedHeap::heap();
  assert(!_should_initiate_conc_mark || g1h->should_do_concurrent_full_gc(_gc_cause),
      "only a GC locker, a System.gc(), stats update, whitebox, or a hum allocation induced GC should start a cycle");

  if (_word_size > 0) {
    // An allocation has been requested. So, try to do that first.
    _result = g1h->attempt_allocation_at_safepoint(_word_size,
                                                   allocation_context(),
                                                   false /* expect_null_cur_alloc_region */);
    if (_result != NULL) {
      // If we can successfully allocate before we actually do the
      // pause then we will consider this pause successful.
      _pause_succeeded = true;
      return;
    }
  }

  GCCauseSetter x(g1h, _gc_cause);
  if (_should_initiate_conc_mark) {
    // It's safer to read old_marking_cycles_completed() here, given
    // that noone else will be updating it concurrently. Since we'll
    // only need it if we're initiating a marking cycle, no point in
    // setting it earlier.
    _old_marking_cycles_completed_before = g1h->old_marking_cycles_completed();

    // At this point we are supposed to start a concurrent cycle. We
    // will do so if one is not already in progress.
    bool res = g1h->g1_policy()->force_initial_mark_if_outside_cycle(_gc_cause);

    // The above routine returns true if we were able to force the
    // next GC pause to be an initial mark; it returns false if a
    // marking cycle is already in progress.
    //
    // If a marking cycle is already in progress just return and skip the
    // pause below - if the reason for requesting this initial mark pause
    // was due to a System.gc() then the requesting thread should block in
    // doit_epilogue() until the marking cycle is complete.
    //
    // If this initial mark pause was requested as part of a humongous
    // allocation then we know that the marking cycle must just have
    // been started by another thread (possibly also allocating a humongous
    // object) as there was no active marking cycle when the requesting
    // thread checked before calling collect() in
    // attempt_allocation_humongous(). Retrying the GC, in this case,
    // will cause the requesting thread to spin inside collect() until the
    // just started marking cycle is complete - which may be a while. So
    // we do NOT retry the GC.
    if (!res) {
      assert(_word_size == 0, "Concurrent Full GC/Humongous Object IM shouldn't be allocating");
      if (_gc_cause != GCCause::_g1_humongous_allocation) {
        _should_retry_gc = true;
      }
      return;
    }
  }

  _pause_succeeded =
    g1h->do_collection_pause_at_safepoint(_target_pause_time_ms);
  if (_pause_succeeded && _word_size > 0) {
    // An allocation had been requested.
    _result = g1h->attempt_allocation_at_safepoint(_word_size,
                                                   allocation_context(),
                                                   true /* expect_null_cur_alloc_region */);
  } else {
    assert(_result == NULL, "invariant");
    if (!_pause_succeeded) {
      // Another possible reason reason for the pause to not be successful
      // is that, again, the GC locker is active (and has become active
      // since the prologue was executed). In this case we should retry
      // the pause after waiting for the GC locker to become inactive.
      _should_retry_gc = true;
    }
  }
}
```

首先又尝试分配了一次，如果还申请不到内存，那么就开始进行垃圾回收了。

[G1CollectedHeap::do_collection_pause_at_safepoint(double target_pause_time_ms)](https://github.com/AdoptOpenJDK/openjdk-jdk10/blob/0f7bcfb753bcb6d8057562da74466140737f4a0d/src/hotspot/share/gc/g1/g1CollectedHeap.cpp#L2898)

下面我们对照 GC 日志一步一步跟踪 Young GC 的逻辑。

## 3. Phase 1 -- Pre Evacuate Collection Set

```
GC(6)   Pre Evacuate Collection Set: 0.2ms
GC(6)     Prepare TLABs: 0.3ms
GC(6)     Choose Collection Set: 0.0ms
GC(6)     Humongous Register: 0.2ms
GC(6)       Humongous Total: 128
GC(6)       Humongous Candidate: 128
```

### a. Prepare TLAB

```cpp
// share/gc/g1/g1CollectedHeap.cpp#L2581
  double start = os::elapsedTime();
  accumulate_statistics_all_tlabs();
  ensure_parsability(true);
  g1_policy()->phase_times()->record_prepare_tlab_time_ms((os::elapsedTime() - start) * 1000.0);
```

将 TLAB 处理为可 parse 的

### b. 释放 mutator 分配的 region

```
// share/gc/g1/g1CollectedHeap.cpp#L5207
void G1CollectedHeap::retire_mutator_alloc_region(HeapRegion* alloc_region,
                                                  size_t allocated_bytes) {
  assert_heap_locked_or_at_safepoint(true /* should_be_vm_thread */);
  assert(alloc_region->is_eden(), "all mutator alloc regions should be eden");

  collection_set()->add_eden_region(alloc_region);
  increase_used(allocated_bytes);
  _hr_printer.retire(alloc_region);
  // We update the eden sizes here, when the region is retired,
  // instead of when it's allocated, since this is the point that its
  // used space has been recored in _summary_bytes_used.
  g1mm()->update_eden_size();
}
```

核心代码是将 region 放到 CSet 中

### c. 选择 CSet

```cpp
// share/gc/g1/g1CollectionSet.cpp#L354
double G1CollectionSet::finalize_young_part(double target_pause_time_ms, G1SurvivorRegions* survivors) {
  double young_start_time_sec = os::elapsedTime();

  finalize_incremental_building();

  guarantee(target_pause_time_ms > 0.0,
            "target_pause_time_ms = %1.6lf should be positive", target_pause_time_ms);

  size_t pending_cards = _policy->pending_cards();
  double base_time_ms = _policy->predict_base_elapsed_time_ms(pending_cards);
  double time_remaining_ms = MAX2(target_pause_time_ms - base_time_ms, 0.0);

  log_trace(gc, ergo, cset)("Start choosing CSet. pending cards: " SIZE_FORMAT " predicted base time: %1.2fms remaining time: %1.2fms target pause time: %1.2fms",
                            pending_cards, base_time_ms, time_remaining_ms, target_pause_time_ms);

  collector_state()->set_last_gc_was_young(collector_state()->gcs_are_young());

  // The young list is laid with the survivor regions from the previous
  // pause are appended to the RHS of the young list, i.e.
  //   [Newly Young Regions ++ Survivors from last pause].

  uint survivor_region_length = survivors->length();
  uint eden_region_length = _g1->eden_regions_count();
  init_region_lengths(eden_region_length, survivor_region_length);

  verify_young_cset_indices();

  // Clear the fields that point to the survivor list - they are all young now.
  survivors->convert_to_eden();

  _bytes_used_before = _inc_bytes_used_before;
  time_remaining_ms = MAX2(time_remaining_ms - _inc_predicted_elapsed_time_ms, 0.0);

  log_trace(gc, ergo, cset)("Add young regions to CSet. eden: %u regions, survivors: %u regions, predicted young region time: %1.2fms, target pause time: %1.2fms",
                            eden_region_length, survivor_region_length, _inc_predicted_elapsed_time_ms, target_pause_time_ms);

  // The number of recorded young regions is the incremental
  // collection set's current size
  set_recorded_rs_lengths(_inc_recorded_rs_lengths);

  double young_end_time_sec = os::elapsedTime();
  phase_times()->record_young_cset_choice_time_ms((young_end_time_sec - young_start_time_sec) * 1000.0);

  return time_remaining_ms;
}
```

Young GC 的 CSet 是 Eden 和 Survivor 区，实际已经在 CSet 中了，所以这部分基本不耗时。

这部分还有一块核心的代码，基于 dirtyCard 和 RememberSet 预估 pause time。

### d. 筛选满足条件的 Humongous Region

```cpp
// share/gc/g1/g1CollectedHeap.cpp#L2663

bool humongous_region_is_candidate(G1CollectedHeap* heap, HeapRegion* region) const {
    assert(region->is_starts_humongous(), "Must start a humongous object");

    oop obj = oop(region->bottom());

    // Dead objects cannot be eager reclaim candidates. Due to class
    // unloading it is unsafe to query their classes so we return early.
    if (heap->is_obj_dead(obj, region)) {
      return false;
    }

    // Candidate selection must satisfy the following constraints
    // while concurrent marking is in progress:
    //
    // * In order to maintain SATB invariants, an object must not be
    // reclaimed if it was allocated before the start of marking and
    // has not had its references scanned.  Such an object must have
    // its references (including type metadata) scanned to ensure no
    // live objects are missed by the marking process.  Objects
    // allocated after the start of concurrent marking don't need to
    // be scanned.
    //
    // * An object must not be reclaimed if it is on the concurrent
    // mark stack.  Objects allocated after the start of concurrent
    // marking are never pushed on the mark stack.
    //
    // Nominating only objects allocated after the start of concurrent
    // marking is sufficient to meet both constraints.  This may miss
    // some objects that satisfy the constraints, but the marking data
    // structures don't support efficiently performing the needed
    // additional tests or scrubbing of the mark stack.
    //
    // However, we presently only nominate is_typeArray() objects.
    // A humongous object containing references induces remembered
    // set entries on other regions.  In order to reclaim such an
    // object, those remembered sets would need to be cleaned up.
    //
    // We also treat is_typeArray() objects specially, allowing them
    // to be reclaimed even if allocated before the start of
    // concurrent mark.  For this we rely on mark stack insertion to
    // exclude is_typeArray() objects, preventing reclaiming an object
    // that is in the mark stack.  We also rely on the metadata for
    // such objects to be built-in and so ensured to be kept live.
    // Frequent allocation and drop of large binary blobs is an
    // important use case for eager reclaim, and this special handling
    // may reduce needed headroom.

    return obj->is_typeArray() && is_remset_small(region);
  }
```

也就是选择 array 类型并且 RemSet 小于等于 G1RSetSparseRegionEntries 的 region，加入 CSet (_humongous_reclaim_candidates) 中

## 4. Phase 2 -- Evacuate Collection Set

```
GC(6)   Evacuate Collection Set: 17.2ms
GC(6)       GC Worker Start (ms):     Min: 73651.3, Avg: 73651.3, Max: 73651.3, Diff:  0.0, Workers: 2
GC(6)                                 73651.3 73651.3
GC(6)     Ext Root Scanning (ms):   Min:  0.9, Avg:  0.9, Max:  0.9, Diff:  0.0, Sum:  1.9, Workers: 2
GC(6)                                0.9  0.9
GC(6)       Thread Roots (ms):        Min:  0.1, Avg:  0.3, Max:  0.4, Diff:  0.3, Sum:  0.5, Workers: 2
GC(6)                                  0.1  0.4
GC(6)       StringTable Roots (ms):   Min:  0.6, Avg:  0.6, Max:  0.6, Diff:  0.1, Sum:  1.2, Workers: 2
GC(6)                                  0.6  0.6
GC(6)       Universe Roots (ms):      Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       JNI Handles Roots (ms):   Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       ObjectSynchronizer Roots (ms): Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       Management Roots (ms):    Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       SystemDictionary Roots (ms): Min:  0.0, Avg:  0.0, Max:  0.1, Diff:  0.1, Sum:  0.1, Workers: 2
GC(6)                                  0.1  0.0
GC(6)       CLDG Roots (ms):          Min:  0.0, Avg:  0.1, Max:  0.2, Diff:  0.2, Sum:  0.2, Workers: 2
GC(6)                                  0.2  0.0
GC(6)       JVMTI Roots (ms):         Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       CM RefProcessor Roots (ms): Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       Wait For Strong CLD (ms): Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       Weak CLD Roots (ms):      Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)       SATB Filtering (ms):      Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)     Update RS (ms):           Min:  2.1, Avg:  2.4, Max:  2.7, Diff:  0.6, Sum:  4.9, Workers: 2
GC(6)                                2.1  2.7
GC(6)       Processed Buffers:        Min: 9, Avg: 10.0, Max: 11, Diff: 2, Sum: 20, Workers: 2
GC(6)                                  9  11
GC(6)       Scanned Cards:            Min: 1684, Avg: 1809.5, Max: 1935, Diff: 251, Sum: 3619, Workers: 2
GC(6)                                  1935  1684
GC(6)       Skipped Cards:            Min: 0, Avg:  0.0, Max: 0, Diff: 0, Sum: 0, Workers: 2
GC(6)                                  0  0
GC(6)       Scan HCC (ms):            Min:  0.7, Avg:  0.7, Max:  0.7, Diff:  0.0, Sum:  1.4, Workers: 2
GC(6)                                  0.7  0.7
GC(6)     Scan RS (ms):             Min:  5.1, Avg:  5.1, Max:  5.2, Diff:  0.1, Sum: 10.3, Workers: 2
GC(6)                                5.2  5.1
GC(6)       Scanned Cards:            Min: 4019, Avg: 4362.0, Max: 4705, Diff: 686, Sum: 8724, Workers: 2
GC(6)                                  4705  4019
GC(6)       Claimed Cards:            Min: 19702, Avg: 23614.5, Max: 27527, Diff: 7825, Sum: 47229, Workers: 2
GC(6)                                  19702  27527
GC(6)       Skipped Cards:            Min: 5000, Avg: 5125.0, Max: 5250, Diff: 250, Sum: 10250, Workers: 2
GC(6)                                  5000  5250
GC(6)     Code Root Scanning (ms):  Min:  0.1, Avg:  0.4, Max:  0.7, Diff:  0.6, Sum:  0.8, Workers: 2
GC(6)                                0.7  0.1
GC(6)     AOT Root Scanning (ms):   skipped
GC(6)                               - -
GC(6)     Object Copy (ms):         Min:  7.9, Avg:  7.9, Max:  7.9, Diff:  0.0, Sum: 15.7, Workers: 2
GC(6)                                7.9  7.9
GC(6)     Termination (ms):         Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                0.0  0.0
GC(6)       Termination Attempts:     Min: 1, Avg:  1.0, Max: 1, Diff: 0, Sum: 2, Workers: 2
GC(6)                                  1  1
GC(6)     GC Worker Other (ms):     Min:  0.1, Avg:  0.1, Max:  0.2, Diff:  0.1, Sum:  0.3, Workers: 2
GC(6)                                0.1  0.2
GC(6)     GC Worker Total (ms):     Min: 16.9, Avg: 16.9, Max: 17.0, Diff:  0.1, Sum: 33.8, Workers: 2
GC(6)                               16.9 17.0
GC(6)       GC Worker End (ms):       Min: 73668.2, Avg: 73668.2, Max: 73668.3, Diff:  0.1, Workers: 2
GC(6)                                 73668.2 73668.3
```

下面开始执行垃圾收集了，这部分逻辑封装在 `G1ParTask` 和 `G1RootProcessor` 中，由 ParallelGCThreads 个 "GC Thread #" 线程并行执行。(或者通过 UseDynamicNumberOfGCThreads 设置动态线程数)

### a. Ext Root Scanning

```cpp
// share/gc/g1/g1RootProcessor.cpp#L75

void G1RootProcessor::evacuate_roots(G1EvacuationRootClosures* closures, uint worker_i) {
  double ext_roots_start = os::elapsedTime();
  G1GCPhaseTimes* phase_times = _g1h->g1_policy()->phase_times();

  process_java_roots(closures, phase_times, worker_i);

  // This is the point where this worker thread will not find more strong CLDs/nmethods.
  // Report this so G1 can synchronize the strong and weak CLDs/nmethods processing.
  if (closures->trace_metadata()) {
    worker_has_discovered_all_strong_classes();
  }

  process_vm_roots(closures, phase_times, worker_i);
  process_string_table_roots(closures, phase_times, worker_i);

  {
    // Now the CM ref_processor roots.
    G1GCParPhaseTimesTracker x(phase_times, G1GCPhaseTimes::CMRefRoots, worker_i);
    if (!_process_strong_tasks.is_task_claimed(G1RP_PS_refProcessor_oops_do)) {
      // We need to treat the discovered reference lists of the
      // concurrent mark ref processor as roots and keep entries
      // (which are added by the marking threads) on them live
      // until they can be processed at the end of marking.
      _g1h->ref_processor_cm()->weak_oops_do(closures->strong_oops());
    }
  }

  if (closures->trace_metadata()) {
    {
      G1GCParPhaseTimesTracker x(phase_times, G1GCPhaseTimes::WaitForStrongCLD, worker_i);
      // Barrier to make sure all workers passed
      // the strong CLD and strong nmethods phases.
      wait_until_all_strong_classes_discovered();
    }

    // Now take the complement of the strong CLDs.
    G1GCParPhaseTimesTracker x(phase_times, G1GCPhaseTimes::WeakCLDRoots, worker_i);
    assert(closures->second_pass_weak_clds() != NULL, "Should be non-null if we are tracing metadata.");
    ClassLoaderDataGraph::roots_cld_do(NULL, closures->second_pass_weak_clds());
  } else {
    phase_times->record_time_secs(G1GCPhaseTimes::WaitForStrongCLD, worker_i, 0.0);
    phase_times->record_time_secs(G1GCPhaseTimes::WeakCLDRoots, worker_i, 0.0);
    assert(closures->second_pass_weak_clds() == NULL, "Should be null if not tracing metadata.");
  }

  // Finish up any enqueued closure apps (attributed as object copy time).
  closures->flush();

  double obj_copy_time_sec = closures->closure_app_seconds();

  phase_times->record_time_secs(G1GCPhaseTimes::ObjCopy, worker_i, obj_copy_time_sec);

  double ext_root_time_sec = os::elapsedTime() - ext_roots_start - obj_copy_time_sec;

  phase_times->record_time_secs(G1GCPhaseTimes::ExtRootScan, worker_i, ext_root_time_sec);

  // During conc marking we have to filter the per-thread SATB buffers
  // to make sure we remove any oops into the CSet (which will show up
  // as implicitly live).
  {
    G1GCParPhaseTimesTracker x(phase_times, G1GCPhaseTimes::SATBFiltering, worker_i);
    if (!_process_strong_tasks.is_task_claimed(G1RP_PS_filter_satb_buffers) && _g1h->collector_state()->mark_in_progress()) {
      JavaThread::satb_mark_queue_set().filter_thread_buffers();
    }
  }

  _process_strong_tasks.all_tasks_completed(n_workers());
}
```

```
// share/gc/g1/g1OopClosures.inline.hpp#L220

template <G1Barrier barrier, G1Mark do_mark_object, bool use_ext>
template <class T>
void G1ParCopyClosure<barrier, do_mark_object, use_ext>::do_oop_work(T* p) {
  T heap_oop = oopDesc::load_heap_oop(p);

  if (oopDesc::is_null(heap_oop)) {
    return;
  }

  oop obj = oopDesc::decode_heap_oop_not_null(heap_oop);

  assert(_worker_id == _par_scan_state->worker_id(), "sanity");

  const InCSetState state = _g1->in_cset_state(obj);
  if (state.is_in_cset()) {
    oop forwardee;
    markOop m = obj->mark();
    if (m->is_marked()) {
      forwardee = (oop) m->decode_pointer();
    } else {
      forwardee = _par_scan_state->copy_to_survivor_space(state, obj, m);
    }
    assert(forwardee != NULL, "forwardee should not be NULL");
    oopDesc::encode_store_heap_oop(p, forwardee);
    if (do_mark_object != G1MarkNone && forwardee != obj) {
      // If the object is self-forwarded we don't need to explicitly
      // mark it, the evacuation failure protocol will do so.
      mark_forwarded_object(obj, forwardee);
    }

    if (barrier == G1BarrierCLD) {
      do_cld_barrier(forwardee);
    }
  } else {
    if (state.is_humongous()) {
      _g1->set_humongous_is_live(obj);
    }

    if (use_ext && state.is_ext()) {
      _par_scan_state->do_oop_ext(p);
    }
    // The object is not in collection set. If we're a root scanning
    // closure during an initial mark pause then attempt to mark the object.
    if (do_mark_object == G1MarkFromRoot) {
      mark_object(obj);
    }
  }
}
```

首先从 root 进行扫描、复制 obj

### b. Update RS 和 Scan RS

```cpp
// share/gc/g1/g1RemSet.cpp#L494

void G1RemSet::oops_into_collection_set_do(G1ParScanThreadState* pss,
                                           CodeBlobClosure* heap_region_codeblobs,
                                           uint worker_i) {
  update_rem_set(pss, worker_i);
  scan_rem_set(pss, heap_region_codeblobs, worker_i);;
}
```

更新扫描 Remember Set，将待处理的引用加入队列。

### c. Object Copy

```cpp
// share/gc/g1/g1ParScanThreadState.inline.hpp#L117

template <class T> inline void G1ParScanThreadState::deal_with_reference(T* ref_to_scan) {
  if (!has_partial_array_mask(ref_to_scan)) {
    HeapRegion* r = _g1h->heap_region_containing(ref_to_scan);
    do_oop_evac(ref_to_scan, r);
  } else {
    do_oop_partial_array((oop*)ref_to_scan);
  }
}
```

从 RefToScanQueue 中取任务来处理

```cpp
// share/gc/g1/g1ParScanThreadState.inline.hpp#L32

template <class T> void G1ParScanThreadState::do_oop_evac(T* p, HeapRegion* from) {
  assert(!oopDesc::is_null(oopDesc::load_decode_heap_oop(p)),
         "Reference should not be NULL here as such are never pushed to the task queue.");
  oop obj = oopDesc::load_decode_heap_oop_not_null(p);

  // Although we never intentionally push references outside of the collection
  // set, due to (benign) races in the claim mechanism during RSet scanning more
  // than one thread might claim the same card. So the same card may be
  // processed multiple times. So redo this check.
  const InCSetState in_cset_state = _g1h->in_cset_state(obj);
  if (in_cset_state.is_in_cset()) {
    markOop m = obj->mark();
    if (m->is_marked()) {
      obj = (oop) m->decode_pointer();
    } else {
      obj = copy_to_survivor_space(in_cset_state, obj, m);
    }
    oopDesc::encode_store_heap_oop(p, obj);
  } else if (in_cset_state.is_humongous()) {
    _g1h->set_humongous_is_live(obj);
  } else {
    assert(in_cset_state.is_default() || in_cset_state.is_ext(),
         "In_cset_state must be NotInCSet or Ext here, but is " CSETSTATE_FORMAT, in_cset_state.value())
  }

  assert(obj != NULL, "Must be");
  if (!HeapRegion::is_in_same_region(p, obj)) {
    update_rs(from, p, obj);
  }
}

```

如果在 CSet 中并且没有被 mark，那么就复制到 survivor 区，或者 promote 到 old 区。

## 5. Phase 3 -- Post Evacuate Collection Set

```
GC(6)   Post Evacuate Collection Set: 3.7ms
GC(6)     Code Roots Fixup: 0.0ms
GC(6)     Preserve CM Refs: 0.0ms
GC(6)       Parallel Preserve CM Refs (ms): skipped
GC(6)                                 - -
GC(6)     Reference Processing: 0.0ms
GC(6)       SoftReference: 0.5ms
GC(6)         Balance queues: 0.0ms
GC(6)         Phase1: 0.2ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Phase2: 0.1ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Phase3: 0.1ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Discovered: 0
GC(6)         Cleared: 0
GC(6)       WeakReference: 0.3ms
GC(6)         Balance queues: 0.0ms
GC(6)         Phase2: 0.1ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Phase3: 0.1ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Discovered: 23
GC(6)         Cleared: 11
GC(6)       FinalReference: 0.3ms
GC(6)         Balance queues: 0.0ms
GC(6)         Phase2: 0.2ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Phase3: 0.1ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Discovered: 0
GC(6)         Cleared: 0
GC(6)       PhantomReference: 0.3ms
GC(6)         Balance queues: 0.0ms
GC(6)         Phase2: 0.2ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Phase3: 0.1ms
GC(6)           Process lists (ms)        Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)         Discovered: 24
GC(6)         Cleared: 15
GC(6)     Clear Card Table: 0.2ms
GC(6)     Reference Enqueuing: 0.1ms
GC(6)       Reference Counts:  Soft: 0  Weak: 12  Final: 0  Phantom: 9
GC(6)     Merge Per-Thread State: 0.0ms
GC(6)     Code Roots Purge: 0.0ms
GC(6)     Redirty Cards: 0.2ms
GC(6)       Parallel Redirty (ms):    Min:  0.0, Avg:  0.0, Max:  0.0, Diff:  0.0, Sum:  0.0, Workers: 2
GC(6)                                  0.0  0.0
GC(6)         Redirtied Cards:          Min: 0, Avg: 1843.0, Max: 3686, Diff: 3686, Sum: 3686, Workers: 2
GC(6)                                    3686  0
GC(6)     DerivedPointerTable Update: 0.0ms
GC(6)     Free Collection Set: 1.1ms
GC(6)       Free Collection Set Serial: 0.1ms
GC(6)       Young Free Collection Set (ms): Min:  0.2, Avg:  0.2, Max:  0.3, Diff:  0.1, Sum:  0.5, Workers: 2
GC(6)                                  0.2  0.3
GC(6)       Non-Young Free Collection Set (ms): skipped
GC(6)                                 - -
GC(6)     Humongous Reclaim: 2.0ms
GC(6)       Humongous Reclaimed: 0
GC(6)     Start New Collection Set: 0.0ms
GC(6)     Resize TLABs: 0.3ms
GC(6)     Expand Heap After Collection: 0.0ms
```

### a. Reference Processing

```
// share/gc/g1/g1CollectedHeap.cpp#L4371

void G1CollectedHeap::post_evacuate_collection_set(EvacuationInfo& evacuation_info, G1ParScanThreadStateSet* per_thread_states) {
  // Process any discovered reference objects - we have
  // to do this _before_ we retire the GC alloc regions
  // as we may have to copy some 'reachable' referent
  // objects (and their reachable sub-graphs) that were
  // not copied during the pause.
  if (g1_policy()->should_process_references()) {
    preserve_cm_referents(per_thread_states);
    process_discovered_references(per_thread_states);
  } else {
    ref_processor_stw()->verify_no_references_recorded();
  }

  G1STWIsAliveClosure is_alive(this);
  G1KeepAliveClosure keep_alive(this);

  {
    double start = os::elapsedTime();

    WeakProcessor::weak_oops_do(&is_alive, &keep_alive);

    double time_ms = (os::elapsedTime() - start) * 1000.0;
    g1_policy()->phase_times()->record_ref_proc_time(time_ms);
  }

  if (G1StringDedup::is_enabled()) {
    double fixup_start = os::elapsedTime();

    G1StringDedup::unlink_or_oops_do(&is_alive, &keep_alive, true, g1_policy()->phase_times());

    double fixup_time_ms = (os::elapsedTime() - fixup_start) * 1000.0;
    g1_policy()->phase_times()->record_string_dedup_fixup_time(fixup_time_ms);
  }

  g1_rem_set()->cleanup_after_oops_into_collection_set_do();

  if (evacuation_failed()) {
    restore_after_evac_failure();

    // Reset the G1EvacuationFailureALot counters and flags
    // Note: the values are reset only when an actual
    // evacuation failure occurs.
    NOT_PRODUCT(reset_evacuation_should_fail();)
  }

  _preserved_marks_set.assert_empty();

  // Enqueue any remaining references remaining on the STW
  // reference processor's discovered lists. We need to do
  // this after the card table is cleaned (and verified) as
  // the act of enqueueing entries on to the pending list
  // will log these updates (and dirty their associated
  // cards). We need these updates logged to update any
  // RSets.
  if (g1_policy()->should_process_references()) {
    enqueue_discovered_references(per_thread_states);
  } else {
    g1_policy()->phase_times()->record_ref_enq_time(0);
  }

  _allocator->release_gc_alloc_regions(evacuation_info);

  merge_per_thread_state_info(per_thread_states);

  // Reset and re-enable the hot card cache.
  // Note the counts for the cards in the regions in the
  // collection set are reset when the collection set is freed.
  _hot_card_cache->reset_hot_cache();
  _hot_card_cache->set_use_cache(true);

  purge_code_root_memory();

  redirty_logged_cards();
#if COMPILER2_OR_JVMCI
  double start = os::elapsedTime();
  DerivedPointerTable::update_pointers();
  g1_policy()->phase_times()->record_derived_pointer_table_update_time((os::elapsedTime() - start) * 1000.0);
#endif
  g1_policy()->print_age_table();
}
```

处理弱引用

### b. Free Collection Set


```
// share/gc/g1/g1CollectedHeap.cpp#L4794

void G1CollectedHeap::free_collection_set(G1CollectionSet* collection_set, EvacuationInfo& evacuation_info, const size_t* surviving_young_words) {
  _eden.clear();

  double free_cset_start_time = os::elapsedTime();

  {
    uint const num_chunks = MAX2(_collection_set.region_length() / G1FreeCollectionSetTask::chunk_size(), 1U);
    uint const num_workers = MIN2(workers()->active_workers(), num_chunks);

    G1FreeCollectionSetTask cl(collection_set, &evacuation_info, surviving_young_words);

    log_debug(gc, ergo)("Running %s using %u workers for collection set length %u",
                        cl.name(),
                        num_workers,
                        _collection_set.region_length());
    workers()->run_task(&cl, num_workers);
  }
  g1_policy()->phase_times()->record_total_free_cset_time_ms((os::elapsedTime() - free_cset_start_time) * 1000.0);

  collection_set->clear();
}
```

### c. Humongous Reclaim

```
// share/gc/g1/g1CollectedHeap.cpp#L4928

void G1CollectedHeap::eagerly_reclaim_humongous_regions() {
  assert_at_safepoint(true);

  if (!G1EagerReclaimHumongousObjects ||
      (!_has_humongous_reclaim_candidates && !log_is_enabled(Debug, gc, humongous))) {
    g1_policy()->phase_times()->record_fast_reclaim_humongous_time_ms(0.0, 0);
    return;
  }

  double start_time = os::elapsedTime();

  FreeRegionList local_cleanup_list("Local Humongous Cleanup List");

  G1FreeHumongousRegionClosure cl(&local_cleanup_list);
  heap_region_iterate(&cl);

  remove_from_old_sets(0, cl.humongous_regions_reclaimed());

  G1HRPrinter* hrp = hr_printer();
  if (hrp->is_active()) {
    FreeRegionListIterator iter(&local_cleanup_list);
    while (iter.more_available()) {
      HeapRegion* hr = iter.get_next();
      hrp->cleanup(hr);
    }
  }

  prepend_to_freelist(&local_cleanup_list);
  decrement_summary_bytes(cl.bytes_freed());

  g1_policy()->phase_times()->record_fast_reclaim_humongous_time_ms((os::elapsedTime() - start_time) * 1000.0,
                                                                    cl.humongous_objects_reclaimed());
}
```

### d. Start New Collection Set


```
// share/gc/g1/g1CollectedHeap.cpp#L2888

void G1CollectedHeap::start_new_collection_set() {
  collection_set()->start_incremental_building();

  clear_cset_fast_test();

  guarantee(_eden.length() == 0, "eden should have been cleared");
  g1_policy()->transfer_survivors_to_cset(survivor());
}
```

## 异常情况 -- To-space exhausted


```
// share/gc/g1/g1ParScanThreadState.cpp#L220

oop G1ParScanThreadState::copy_to_survivor_space(InCSetState const state,
                                                 oop const old,
                                                 markOop const old_mark) {
  const size_t word_sz = old->size();
  HeapRegion* const from_region = _g1h->heap_region_containing(old);
  // +1 to make the -1 indexes valid...
  const int young_index = from_region->young_index_in_cset()+1;
  assert( (from_region->is_young() && young_index >  0) ||
         (!from_region->is_young() && young_index == 0), "invariant" );
  const AllocationContext_t context = from_region->allocation_context();

  uint age = 0;
  InCSetState dest_state = next_state(state, old_mark, age);
  // The second clause is to prevent premature evacuation failure in case there
  // is still space in survivor, but old gen is full.
  if (_old_gen_is_full && dest_state.is_old()) {
    return handle_evacuation_failure_par(old, old_mark);
  }
  HeapWord* obj_ptr = _plab_allocator->plab_allocate(dest_state, word_sz, context);

  // PLAB allocations should succeed most of the time, so we'll
  // normally check against NULL once and that's it.
  if (obj_ptr == NULL) {
    bool plab_refill_failed = false;
    obj_ptr = _plab_allocator->allocate_direct_or_new_plab(dest_state, word_sz, context, &plab_refill_failed);
    if (obj_ptr == NULL) {
      obj_ptr = allocate_in_next_plab(state, &dest_state, word_sz, context, plab_refill_failed);
      if (obj_ptr == NULL) {
        // This will either forward-to-self, or detect that someone else has
        // installed a forwarding pointer.
        return handle_evacuation_failure_par(old, old_mark);
      }
    }
    if (_g1h->_gc_tracer_stw->should_report_promotion_events()) {
      // The events are checked individually as part of the actual commit
      report_promotion_event(dest_state, old, word_sz, age, obj_ptr, context);
    }
  }

  assert(obj_ptr != NULL, "when we get here, allocation should have succeeded");
  assert(_g1h->is_in_reserved(obj_ptr), "Allocated memory should be in the heap");

#ifndef PRODUCT
  // Should this evacuation fail?
  if (_g1h->evacuation_should_fail()) {
    // Doing this after all the allocation attempts also tests the
    // undo_allocation() method too.
    _plab_allocator->undo_allocation(dest_state, obj_ptr, word_sz, context);
    return handle_evacuation_failure_par(old, old_mark);
  }
#endif // !PRODUCT

  // We're going to allocate linearly, so might as well prefetch ahead.
  Prefetch::write(obj_ptr, PrefetchCopyIntervalInBytes);

  const oop obj = oop(obj_ptr);
  const oop forward_ptr = old->forward_to_atomic(obj);
  if (forward_ptr == NULL) {
    Copy::aligned_disjoint_words((HeapWord*) old, obj_ptr, word_sz);

    if (dest_state.is_young()) {
      if (age < markOopDesc::max_age) {
        age++;
      }
      if (old_mark->has_displaced_mark_helper()) {
        // In this case, we have to install the mark word first,
        // otherwise obj looks to be forwarded (the old mark word,
        // which contains the forward pointer, was copied)
        obj->set_mark(old_mark);
        markOop new_mark = old_mark->displaced_mark_helper()->set_age(age);
        old_mark->set_displaced_mark_helper(new_mark);
      } else {
        obj->set_mark(old_mark->set_age(age));
      }
      _age_table.add(age, word_sz);
    } else {
      obj->set_mark(old_mark);
    }

    if (G1StringDedup::is_enabled()) {
      const bool is_from_young = state.is_young();
      const bool is_to_young = dest_state.is_young();
      assert(is_from_young == _g1h->heap_region_containing(old)->is_young(),
             "sanity");
      assert(is_to_young == _g1h->heap_region_containing(obj)->is_young(),
             "sanity");
      G1StringDedup::enqueue_from_evacuation(is_from_young,
                                             is_to_young,
                                             _worker_id,
                                             obj);
    }

    _surviving_young_words[young_index] += word_sz;

    if (obj->is_objArray() && arrayOop(obj)->length() >= ParGCArrayScanChunk) {
      // We keep track of the next start index in the length field of
      // the to-space object. The actual length can be found in the
      // length field of the from-space object.
      arrayOop(obj)->set_length(0);
      oop* old_p = set_partial_array_mask(old);
      push_on_queue(old_p);
    } else {
      HeapRegion* const to_region = _g1h->heap_region_containing(obj_ptr);
      _scanner.set_region(to_region);
      obj->oop_iterate_backwards(&_scanner);
    }
    return obj;
  } else {
    _plab_allocator->undo_allocation(dest_state, obj_ptr, word_sz, context);
    return forward_ptr;
  }
}
```

To-space exhausted 产生的原因只有一个：无法为复制对象申请空间。
