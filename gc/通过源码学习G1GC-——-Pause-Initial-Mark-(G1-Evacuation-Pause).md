上一篇文章分析了 Pause Young (G1 Evacuation Pause) 的流程，这一篇我们分析 Pause Initial Mark (G1 Evacuation Pause) 的流程。

Initial Mark 的逻辑是附加在 Young GC 中的，而且具体代码也封装在`VM_G1IncCollectionPause` 类中，所以本文的重点是分析触发 Initial Mark 的条件，以及 Initial Mark 相对于 Young GC 附加的逻辑。

##1. 触发条件

对比 Initial Mark 与 Young 的日志，主要的不同在于：

```
GC(3) Initiate concurrent cycle (concurrent cycle initiation requested)
GC(3) Pause Initial Mark (G1 Evacuation Pause)
```

"Initial Mark" 这个关键字段的打印在 `do_collection_pause_at_safepoint` 方法中，标识 GC 类型的字段在`collector_state`中。

```java
// hotspot/share/gc/g1/g1CollectedHeap.cpp#L2960
    G1HeapVerifier::G1VerifyType verify_type;
    FormatBuffer<> gc_string("Pause ");
    if (collector_state()->during_initial_mark_pause()) {
      gc_string.append("Initial Mark");
      verify_type = G1HeapVerifier::G1VerifyInitialMark;
    } else if (collector_state()->gcs_are_young()) {
      gc_string.append("Young");
      verify_type = G1HeapVerifier::G1VerifyYoungOnly;
    } else {
      gc_string.append("Mixed");
      verify_type = G1HeapVerifier::G1VerifyMixed;
    }
```

对 Initial Mark 的判定在前面的代码中。

```java
// hotspot/share/gc/g1/g1CollectedHeap.cpp#L2925

// We should not be doing initial mark unless the conc mark thread is running
if (!_cmThread->should_terminate()) {
  // This call will decide whether this pause is an initial-mark
  // pause. If it is, during_initial_mark_pause() will return true
  // for the duration of this pause.
  g1_policy()->decide_on_conc_mark_initiation();
}
```

首先 ConcurrentMarkThread 得处于 running 状态，然后调用`g1_policy`的 `decide_on_conc_mark_initiation` 方法判定。

```java
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L950

void G1DefaultPolicy::decide_on_conc_mark_initiation() {
  // We are about to decide on whether this pause will be an
  // initial-mark pause.

  // First, collector_state()->during_initial_mark_pause() should not be already set. We
  // will set it here if we have to. However, it should be cleared by
  // the end of the pause (it's only set for the duration of an
  // initial-mark pause).
  assert(!collector_state()->during_initial_mark_pause(), "pre-condition");

  if (collector_state()->initiate_conc_mark_if_possible()) {
    // We had noticed on a previous pause that the heap occupancy has
    // gone over the initiating threshold and we should start a
    // concurrent marking cycle. So we might initiate one.

    if (!about_to_start_mixed_phase() && collector_state()->gcs_are_young()) {
      // Initiate a new initial mark if there is no marking or reclamation going on.
      initiate_conc_mark();
      log_debug(gc, ergo)("Initiate concurrent cycle (concurrent cycle initiation requested)");
    } else if (_g1->is_user_requested_concurrent_full_gc(_g1->gc_cause())) {
      // Initiate a user requested initial mark. An initial mark must be young only
      // GC, so the collector state must be updated to reflect this.
      collector_state()->set_gcs_are_young(true);
      collector_state()->set_last_young_gc(false);

      abort_time_to_mixed_tracking();
      initiate_conc_mark();
      log_debug(gc, ergo)("Initiate concurrent cycle (user requested concurrent cycle)");
    } else {
      // The concurrent marking thread is still finishing up the
      // previous cycle. If we start one right now the two cycles
      // overlap. In particular, the concurrent marking thread might
      // be in the process of clearing the next marking bitmap (which
      // we will use for the next cycle if we start one). Starting a
      // cycle now will be bad given that parts of the marking
      // information might get cleared by the marking thread. And we
      // cannot wait for the marking thread to finish the cycle as it
      // periodically yields while clearing the next marking bitmap
      // and, if it's in a yield point, it's waiting for us to
      // finish. So, at this point we will not start a cycle and we'll
      // let the concurrent marking thread complete the last one.
      log_debug(gc, ergo)("Do not initiate concurrent cycle (concurrent cycle already in progress)");
    }
  }
}
```

首先 `collector_state` 的 initiate_conc_mark_if_possible 标识得为 true，然后满足 GC 状态的条件，那么就触发 `initiate_conc_mark`

```
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L945
void G1DefaultPolicy::initiate_conc_mark() {
  collector_state()->set_during_initial_mark_pause(true);
  collector_state()->set_initiate_conc_mark_if_possible(false);
}
```

接着追踪 initiate_conc_mark_if_possible，发现更新逻辑在 `record_collection_pause_end` 方法中，而该方法的调用位置在 `do_collection_pause_at_safepoint` 方法中，也就是上次 GC 执行完后调用。

```java
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L569
void G1DefaultPolicy::record_collection_pause_end(double pause_time_ms, size_t cards_scanned, size_t heap_used_bytes_before_gc) {
  double end_time_sec = os::elapsedTime();

  size_t cur_used_bytes = _g1->used();
  assert(cur_used_bytes == _g1->recalculate_used(), "It should!");
  bool last_pause_included_initial_mark = false;
  bool update_stats = !_g1->evacuation_failed();

  record_pause(young_gc_pause_kind(), end_time_sec - pause_time_ms / 1000.0, end_time_sec);

  _collection_pause_end_millis = os::javaTimeNanos() / NANOSECS_PER_MILLISEC;

  last_pause_included_initial_mark = collector_state()->during_initial_mark_pause();
  if (last_pause_included_initial_mark) {
    record_concurrent_mark_init_end(0.0);
  } else {
    maybe_start_marking();
  }

  double app_time_ms = (phase_times()->cur_collection_start_sec() * 1000.0 - _analytics->prev_collection_pause_end_ms());
  if (app_time_ms < MIN_TIMER_GRANULARITY) {
    // This usually happens due to the timer not having the required
    // granularity. Some Linuxes are the usual culprits.
    // We'll just set it to something (arbitrarily) small.
    app_time_ms = 1.0;
  }

  if (update_stats) {
    // We maintain the invariant that all objects allocated by mutator
    // threads will be allocated out of eden regions. So, we can use
    // the eden region number allocated since the previous GC to
    // calculate the application's allocate rate. The only exception
    // to that is humongous objects that are allocated separately. But
    // given that humongous object allocations do not really affect
    // either the pause's duration nor when the next pause will take
    // place we can safely ignore them here.
    uint regions_allocated = _collection_set->eden_region_length();
    double alloc_rate_ms = (double) regions_allocated / app_time_ms;
    _analytics->report_alloc_rate_ms(alloc_rate_ms);

    double interval_ms =
      (end_time_sec - _analytics->last_known_gc_end_time_sec()) * 1000.0;
    _analytics->update_recent_gc_times(end_time_sec, pause_time_ms);
    _analytics->compute_pause_time_ratio(interval_ms, pause_time_ms);
  }

  bool new_in_marking_window = collector_state()->in_marking_window();
  bool new_in_marking_window_im = false;
  if (last_pause_included_initial_mark) {
    new_in_marking_window = true;
    new_in_marking_window_im = true;
  }

  if (collector_state()->last_young_gc()) {
    // This is supposed to to be the "last young GC" before we start
    // doing mixed GCs. Here we decide whether to start mixed GCs or not.
    assert(!last_pause_included_initial_mark, "The last young GC is not allowed to be an initial mark GC");

    if (next_gc_should_be_mixed("start mixed GCs",
                                "do not start mixed GCs")) {
      collector_state()->set_gcs_are_young(false);
    } else {
      // We aborted the mixed GC phase early.
      abort_time_to_mixed_tracking();
    }

    collector_state()->set_last_young_gc(false);
  }

  if (!collector_state()->last_gc_was_young()) {
    // This is a mixed GC. Here we decide whether to continue doing
    // mixed GCs or not.
    if (!next_gc_should_be_mixed("continue mixed GCs",
                                 "do not continue mixed GCs")) {
      collector_state()->set_gcs_are_young(true);

      maybe_start_marking();
    }
  }

  _short_lived_surv_rate_group->start_adding_regions();
  // Do that for any other surv rate groups

  double scan_hcc_time_ms = G1HotCardCache::default_use_cache() ? average_time_ms(G1GCPhaseTimes::ScanHCC) : 0.0;

  if (update_stats) {
    double cost_per_card_ms = 0.0;
    if (_pending_cards > 0) {
      cost_per_card_ms = (average_time_ms(G1GCPhaseTimes::UpdateRS) - scan_hcc_time_ms) / (double) _pending_cards;
      _analytics->report_cost_per_card_ms(cost_per_card_ms);
    }
    _analytics->report_cost_scan_hcc(scan_hcc_time_ms);

    double cost_per_entry_ms = 0.0;
    if (cards_scanned > 10) {
      cost_per_entry_ms = average_time_ms(G1GCPhaseTimes::ScanRS) / (double) cards_scanned;
      _analytics->report_cost_per_entry_ms(cost_per_entry_ms, collector_state()->last_gc_was_young());
    }

    if (_max_rs_lengths > 0) {
      double cards_per_entry_ratio =
        (double) cards_scanned / (double) _max_rs_lengths;
      _analytics->report_cards_per_entry_ratio(cards_per_entry_ratio, collector_state()->last_gc_was_young());
    }

    // This is defensive. For a while _max_rs_lengths could get
    // smaller than _recorded_rs_lengths which was causing
    // rs_length_diff to get very large and mess up the RSet length
    // predictions. The reason was unsafe concurrent updates to the
    // _inc_cset_recorded_rs_lengths field which the code below guards
    // against (see CR 7118202). This bug has now been fixed (see CR
    // 7119027). However, I'm still worried that
    // _inc_cset_recorded_rs_lengths might still end up somewhat
    // inaccurate. The concurrent refinement thread calculates an
    // RSet's length concurrently with other CR threads updating it
    // which might cause it to calculate the length incorrectly (if,
    // say, it's in mid-coarsening). So I'll leave in the defensive
    // conditional below just in case.
    size_t rs_length_diff = 0;
    size_t recorded_rs_lengths = _collection_set->recorded_rs_lengths();
    if (_max_rs_lengths > recorded_rs_lengths) {
      rs_length_diff = _max_rs_lengths - recorded_rs_lengths;
    }
    _analytics->report_rs_length_diff((double) rs_length_diff);

    size_t freed_bytes = heap_used_bytes_before_gc - cur_used_bytes;
    size_t copied_bytes = _collection_set->bytes_used_before() - freed_bytes;
    double cost_per_byte_ms = 0.0;

    if (copied_bytes > 0) {
      cost_per_byte_ms = average_time_ms(G1GCPhaseTimes::ObjCopy) / (double) copied_bytes;
      _analytics->report_cost_per_byte_ms(cost_per_byte_ms, collector_state()->in_marking_window());
    }

    if (_collection_set->young_region_length() > 0) {
      _analytics->report_young_other_cost_per_region_ms(young_other_time_ms() /
                                                        _collection_set->young_region_length());
    }

    if (_collection_set->old_region_length() > 0) {
      _analytics->report_non_young_other_cost_per_region_ms(non_young_other_time_ms() /
                                                            _collection_set->old_region_length());
    }

    _analytics->report_constant_other_time_ms(constant_other_time_ms(pause_time_ms));

    _analytics->report_pending_cards((double) _pending_cards);
    _analytics->report_rs_lengths((double) _max_rs_lengths);
  }

  collector_state()->set_in_marking_window(new_in_marking_window);
  collector_state()->set_in_marking_window_im(new_in_marking_window_im);
  _free_regions_at_end_of_collection = _g1->num_free_regions();
  // IHOP control wants to know the expected young gen length if it were not
  // restrained by the heap reserve. Using the actual length would make the
  // prediction too small and the limit the young gen every time we get to the
  // predicted target occupancy.
  size_t last_unrestrained_young_length = update_young_list_max_and_target_length();
  update_rs_lengths_prediction();

  update_ihop_prediction(app_time_ms / 1000.0,
                         _bytes_allocated_in_old_since_last_gc,
                         last_unrestrained_young_length * HeapRegion::GrainBytes);
  _bytes_allocated_in_old_since_last_gc = 0;

  _ihop_control->send_trace_event(_g1->gc_tracer_stw());

  // Note that _mmu_tracker->max_gc_time() returns the time in seconds.
  double update_rs_time_goal_ms = _mmu_tracker->max_gc_time() * MILLIUNITS * G1RSetUpdatingPauseTimePercent / 100.0;

  if (update_rs_time_goal_ms < scan_hcc_time_ms) {
    log_debug(gc, ergo, refine)("Adjust concurrent refinement thresholds (scanning the HCC expected to take longer than Update RS time goal)."
                                "Update RS time goal: %1.2fms Scan HCC time: %1.2fms",
                                update_rs_time_goal_ms, scan_hcc_time_ms);

    update_rs_time_goal_ms = 0;
  } else {
    update_rs_time_goal_ms -= scan_hcc_time_ms;
  }
  _g1->concurrent_refine()->adjust(average_time_ms(G1GCPhaseTimes::UpdateRS) - scan_hcc_time_ms,
                                      phase_times()->sum_thread_work_items(G1GCPhaseTimes::UpdateRS),
                                      update_rs_time_goal_ms);

  cset_chooser()->verify();
}
```

1. 如果这次不是“Initial Mark”，那么调用 maybe_start_marking 方法检查一下;
2. 如果这次是“Mixed”，并且下次不是“Mixed”，那么调用 maybe_start_marking 方法检查一下。

```java
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L1011
void G1DefaultPolicy::maybe_start_marking() {
  if (need_to_start_conc_mark("end of GC")) {
    // Note: this might have already been set, if during the last
    // pause we decided to start a cycle but at the beginning of
    // this pause we decided to postpone it. That's OK.
    collector_state()->set_initiate_conc_mark_if_possible(true);
  }
}
```

```java
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L544
bool G1DefaultPolicy::need_to_start_conc_mark(const char* source, size_t alloc_word_size) {
  if (about_to_start_mixed_phase()) {
    return false;
  }

  size_t marking_initiating_used_threshold = _ihop_control->get_conc_mark_start_threshold();

  size_t cur_used_bytes = _g1->non_young_capacity_bytes();
  size_t alloc_byte_size = alloc_word_size * HeapWordSize;
  size_t marking_request_bytes = cur_used_bytes + alloc_byte_size;

  bool result = false;
  if (marking_request_bytes > marking_initiating_used_threshold) {
    result = collector_state()->gcs_are_young() && !collector_state()->last_young_gc();
    log_debug(gc, ergo, ihop)("%s occupancy: " SIZE_FORMAT "B allocation request: " SIZE_FORMAT "B threshold: " SIZE_FORMAT "B (%1.2f) source: %s",
                              result ? "Request concurrent cycle initiation (occupancy higher than threshold)" : "Do not request concurrent cycle initiation (still doing mixed collections)",
                              cur_used_bytes, alloc_byte_size, marking_initiating_used_threshold, (double) marking_initiating_used_threshold / _g1->capacity() * 100, source);
  }

  return result;
}
```

如果 (_old_set + _humongous_set) 占用的内存大于 IHOP 控制的阈值，那么设置 initiate_conc_mark_if_possible 为 true。

###a. IHOP

IHOP 有两种实现，通过 G1UseAdaptiveIHOP 参数控制。

```java
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L755
G1IHOPControl* G1DefaultPolicy::create_ihop_control(const G1Predictions* predictor){
  if (G1UseAdaptiveIHOP) {
    return new G1AdaptiveIHOPControl(InitiatingHeapOccupancyPercent,
                                     predictor,
                                     G1ReservePercent,
                                     G1HeapWastePercent);
  } else {
    return new G1StaticIHOPControl(InitiatingHeapOccupancyPercent);
  }
}
```

默认情况下 G1UseAdaptiveIHOP 为 true，那么实例化 G1AdaptiveIHOPControl，该实现结合 G1ReservePercent 和 G1HeapWastePercent 动态评估该值。

如果设置了 -XX:-G1UseAdaptiveIHOP，那么实例化 G1StaticIHOPControl，那么该值等于 InitiatingHeapOccupancyPercent * _target_occupancy / 100.0，而 _target_occupancy 是当前占用的 heap 大小，不一定是 Xmx。

###b. Humongous

还有一个触发 "Initial Mark" 的情况是 humongous 分配。

```
HeapWord* G1CollectedHeap::attempt_allocation_at_safepoint(size_t word_size,
                                                           AllocationContext_t context,
                                                           bool expect_null_mutator_alloc_region) {
  assert_at_safepoint(true /* should_be_vm_thread */);
  assert(!_allocator->has_mutator_alloc_region(context) || !expect_null_mutator_alloc_region,
         "the current alloc region was unexpectedly found to be non-NULL");

  if (!is_humongous(word_size)) {
    return _allocator->attempt_allocation_locked(word_size, context);
  } else {
    HeapWord* result = humongous_obj_allocate(word_size, context);
    if (result != NULL && g1_policy()->need_to_start_conc_mark("STW humongous allocation")) {
      collector_state()->set_initiate_conc_mark_if_possible(true);
    }
    return result;
  }

  ShouldNotReachHere();
}
```

如果在安全点调用 humongous 分配成功，并且满足 need_to_start_conc_mark 的条件，那么设置 initiate_conc_mark_if_possible 为true。


##2. 操作

下面看一下  Initial Mark 相对于 Young GC 附加的操作。

### a. checkpoint_roots_initial_pre

```java
// hotspot/share/gc/g1/g1CollectedHeap.cpp#L3053
        if (collector_state()->during_initial_mark_pause()) {
          concurrent_mark()->checkpoint_roots_initial_pre();
        }
```

1. reset G1ConcurrentMark
2. 遍历所有 region，标记 start_of_marking


### b. Clear Claimed Marks

在 `pre_evacuate_collection_set()` 方法中清除 Claimed Marks

```java
// hotspot/share/gc/g1/g1CollectedHeap.cpp#4324
  // InitialMark needs claim bits to keep track of the marked-through CLDs.
  if (collector_state()->during_initial_mark_pause()) {
    double start_clear_claimed_marks = os::elapsedTime();

    ClassLoaderDataGraph::clear_claimed_marks();

    double recorded_clear_claimed_marks_time_ms = (os::elapsedTime() - start_clear_claimed_marks) * 1000.0;
    phase_times->record_clear_claimed_marks_time_ms(recorded_clear_claimed_marks_time_ms);
  }
```

### c.  mark from root

```java
// hotspot/share/gc/g1/g1RootClosures.cpp#L113
G1EvacuationRootClosures* G1EvacuationRootClosures::create_root_closures(G1ParScanThreadState* pss, G1CollectedHeap* g1h) {
  G1EvacuationRootClosures* res = create_root_closures_ext(pss, g1h);
  if (res != NULL) {
    return res;
  }

  if (g1h->collector_state()->during_initial_mark_pause()) {
    if (ClassUnloadingWithConcurrentMark) {
      res = new G1InitialMarkClosures<G1MarkPromotedFromRoot>(g1h, pss);
    } else {
      res = new G1InitialMarkClosures<G1MarkFromRoot>(g1h, pss);
    }
  } else {
    res = new G1EvacuationClosures(g1h, pss, g1h->collector_state()->gcs_are_young());
  }
  return res;
}
```

根扫描的逻辑封装在 G1InitialMarkClosures 中，而 Young GC 的逻辑封装在 G1EvacuationClosures 中。

G1InitialMarkClosures 中 process_only_dirty = false 并且 must_claim_cld = true，

G1EvacuationClosures 中 process_only_dirty = true 并且 must_claim_cld = false。

也就是说在 InitialMark 过程中将会扫描所有的 cld，并且必须是 claim 的才会尝试清除修改的 oops。

### d. doConcurrentMark

```
// hotspot/share/gc/g1/g1CollectedHeap.cpp#L3225
  if (should_start_conc_mark) {
    // CAUTION: after the doConcurrentMark() call below,
    // the concurrent marking thread(s) could be running
    // concurrently with us. Make sure that anything after
    // this point does not assume that we are the only GC thread
    // running. Note: of course, the actual marking work will
    // not start until the safepoint itself is released in
    // SuspendibleThreadSet::desynchronize().
    doConcurrentMark();
  }
```

通知 concurrent marking thread 启动
