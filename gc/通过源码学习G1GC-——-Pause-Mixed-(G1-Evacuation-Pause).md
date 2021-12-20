Pause Mixed 的逻辑也是封装在`VM_G1IncCollectionPause` 中，所以本文的重点是分析触发 Pause Mixed 的条件，以及 Pause Mixed 相对于 Young 的不同点。

##1. 触发条件

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

对于 Initial Mark 这里就不重复了，所以区别 Young 和 Mixed 的参数就是 `gcs_are_young`，该参数赋值的逻辑在上次 Pause 结束前调用的 `record_collection_pause_end` 方法中。

首先我们通过注释了解一下几个关键参数：

```cpp
// hotspot/share/gc/g1/g1CollectorState.hpp#L35

  // Indicates whether we are in "full young" or "mixed" GC mode.
  bool _gcs_are_young;
  // Was the last GC "young"?
  bool _last_gc_was_young;
  // Is this the "last young GC" before we start doing mixed GCs?
  // Set after a concurrent mark has completed.
  bool _last_young_gc;
```

在`record_collection_pause_end`方法中：

```cpp
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L622

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
```

1. 如果这是执行 mixed 之前最后一次 yong，那么调用 `next_gc_should_be_mixed` 判断下次是否应该开始 mixed；
2. 如果这次是 mixed ，那么调用 `next_gc_should_be_mixed` 判断下次是否应该继续 mixed。


继续分析 `next_gc_should_be_mixed` 的判断逻辑：

```cpp
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L1073

bool G1DefaultPolicy::next_gc_should_be_mixed(const char* true_action_str,
                                       const char* false_action_str) const {
  if (cset_chooser()->is_empty()) {
    log_debug(gc, ergo)("%s (candidate old regions not available)", false_action_str);
    return false;
  }

  // Is the amount of uncollected reclaimable space above G1HeapWastePercent?
  size_t reclaimable_bytes = cset_chooser()->remaining_reclaimable_bytes();
  double reclaimable_percent = reclaimable_bytes_percent(reclaimable_bytes);
  double threshold = (double) G1HeapWastePercent;
  if (reclaimable_percent <= threshold) {
    log_debug(gc, ergo)("%s (reclaimable percentage not over threshold). candidate old regions: %u reclaimable: " SIZE_FORMAT " (%1.2f) threshold: " UINTX_FORMAT,
                        false_action_str, cset_chooser()->remaining_regions(), reclaimable_bytes, reclaimable_percent, G1HeapWastePercent);
    return false;
  }
  log_debug(gc, ergo)("%s (candidate old regions available). candidate old regions: %u reclaimable: " SIZE_FORMAT " (%1.2f) threshold: " UINTX_FORMAT,
                      true_action_str, cset_chooser()->remaining_regions(), reclaimable_bytes, reclaimable_percent, G1HeapWastePercent);
  return true;
}
```

1. 首先候选的老年代 region 回收集合不为空；
2. 未收集的可回收空间的比例（注：相对于 committed 的大小）大于 G1HeapWastePercent。

`last_young_gc` 这个参数的设置在 Concurrent Mark 完成之后。

```
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L503

void G1DefaultPolicy::record_concurrent_mark_cleanup_completed() {
  bool should_continue_with_reclaim = next_gc_should_be_mixed("request last young-only gc",
                                                              "skip last young-only gc");
  collector_state()->set_last_young_gc(should_continue_with_reclaim);
  // We skip the marking phase.
  if (!should_continue_with_reclaim) {
    abort_time_to_mixed_tracking();
  }
  collector_state()->set_in_marking_window(false);
}
```

调用 `next_gc_should_be_mixed` 判断是否应该执行 mixed，如果满足条件那么设置 last_young_gc 为 true。


##2. 操作

Pause Mixed 相对于 Young GC 的不同点主要是 cset 的选择。

```cpp
// hotspot/share/gc/g1/g1CollectionSet.cpp#L410

void G1CollectionSet::finalize_old_part(double time_remaining_ms) {
  double non_young_start_time_sec = os::elapsedTime();
  double predicted_old_time_ms = 0.0;

  if (!collector_state()->gcs_are_young()) {
    cset_chooser()->verify();
    const uint min_old_cset_length = _policy->calc_min_old_cset_length();
    const uint max_old_cset_length = _policy->calc_max_old_cset_length();

    uint expensive_region_num = 0;
    bool check_time_remaining = _policy->adaptive_young_list_length();

    HeapRegion* hr = cset_chooser()->peek();
    while (hr != NULL) {
      if (old_region_length() >= max_old_cset_length) {
        // Added maximum number of old regions to the CSet.
        log_debug(gc, ergo, cset)("Finish adding old regions to CSet (old CSet region num reached max). old %u regions, max %u regions",
                                  old_region_length(), max_old_cset_length);
        break;
      }

      // Stop adding regions if the remaining reclaimable space is
      // not above G1HeapWastePercent.
      size_t reclaimable_bytes = cset_chooser()->remaining_reclaimable_bytes();
      double reclaimable_percent = _policy->reclaimable_bytes_percent(reclaimable_bytes);
      double threshold = (double) G1HeapWastePercent;
      if (reclaimable_percent <= threshold) {
        // We've added enough old regions that the amount of uncollected
        // reclaimable space is at or below the waste threshold. Stop
        // adding old regions to the CSet.
        log_debug(gc, ergo, cset)("Finish adding old regions to CSet (reclaimable percentage not over threshold). "
                                  "old %u regions, max %u regions, reclaimable: " SIZE_FORMAT "B (%1.2f%%) threshold: " UINTX_FORMAT "%%",
                                  old_region_length(), max_old_cset_length, reclaimable_bytes, reclaimable_percent, G1HeapWastePercent);
        break;
      }

      double predicted_time_ms = predict_region_elapsed_time_ms(hr);
      if (check_time_remaining) {
        if (predicted_time_ms > time_remaining_ms) {
          // Too expensive for the current CSet.

          if (old_region_length() >= min_old_cset_length) {
            // We have added the minimum number of old regions to the CSet,
            // we are done with this CSet.
            log_debug(gc, ergo, cset)("Finish adding old regions to CSet (predicted time is too high). "
                                      "predicted time: %1.2fms, remaining time: %1.2fms old %u regions, min %u regions",
                                      predicted_time_ms, time_remaining_ms, old_region_length(), min_old_cset_length);
            break;
          }

          // We'll add it anyway given that we haven't reached the
          // minimum number of old regions.
          expensive_region_num += 1;
        }
      } else {
        if (old_region_length() >= min_old_cset_length) {
          // In the non-auto-tuning case, we'll finish adding regions
          // to the CSet if we reach the minimum.

          log_debug(gc, ergo, cset)("Finish adding old regions to CSet (old CSet region num reached min). old %u regions, min %u regions",
                                    old_region_length(), min_old_cset_length);
          break;
        }
      }

      // We will add this region to the CSet.
      time_remaining_ms = MAX2(time_remaining_ms - predicted_time_ms, 0.0);
      predicted_old_time_ms += predicted_time_ms;
      cset_chooser()->pop(); // already have region via peek()
      _g1->old_set_remove(hr);
      add_old_region(hr);

      hr = cset_chooser()->peek();
    }
    if (hr == NULL) {
      log_debug(gc, ergo, cset)("Finish adding old regions to CSet (candidate old regions not available)");
    }

    if (expensive_region_num > 0) {
      // We print the information once here at the end, predicated on
      // whether we added any apparently expensive regions or not, to
      // avoid generating output per region.
      log_debug(gc, ergo, cset)("Added expensive regions to CSet (old CSet region num not reached min)."
                                "old: %u regions, expensive: %u regions, min: %u regions, remaining time: %1.2fms",
                                old_region_length(), expensive_region_num, min_old_cset_length, time_remaining_ms);
    }

    cset_chooser()->verify();
  }

  stop_incremental_building();

  log_debug(gc, ergo, cset)("Finish choosing CSet. old: %u regions, predicted old region time: %1.2fms, time remaining: %1.2f",
                            old_region_length(), predicted_old_time_ms, time_remaining_ms);

  double non_young_end_time_sec = os::elapsedTime();
  phase_times()->record_non_young_cset_choice_time_ms((non_young_end_time_sec - non_young_start_time_sec) * 1000.0);

  QuickSort::sort(_collection_set_regions, _collection_set_cur_length, compare_region_idx, true);
}
```

从 `cset_chooser()` 中获取老年代的 region 加入 cset，并且限制了每次加入 cset 的 region 个数。

而 `cset_chooser` 的构建发生在 Cleanup 这个 STW 操作中。

```cpp
// hotspot/share/gc/g1/g1DefaultPolicy.cpp#L996

void G1DefaultPolicy::record_concurrent_mark_cleanup_end() {
  cset_chooser()->rebuild(_g1->workers(), _g1->num_regions());

  double end_sec = os::elapsedTime();
  double elapsed_time_ms = (end_sec - _mark_cleanup_start_sec) * 1000.0;
  _analytics->report_concurrent_mark_cleanup_times_ms(elapsed_time_ms);
  _analytics->append_prev_collection_pause_end_ms(elapsed_time_ms);

  record_pause(Cleanup, _mark_cleanup_start_sec, end_sec);
}
```

加入 cset 的条件是 region 中存活空间小于 `HeapRegion::GrainBytes * (size_t) G1MixedGCLiveThresholdPercent / 100`
