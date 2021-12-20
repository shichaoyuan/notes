并行执行阶段的逻辑封装在 `ConcurrentMarkThread` 中，该线程对应 G1 Main Marker 。

具体的执行逻辑在 `run_service()` 方法中，我们对应着日志分块阅读代码。

## 1. Concurrent Clear Claimed Marks

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L280

        G1ConcPhase p(G1ConcurrentPhase::CLEAR_CLAIMED_MARKS, this);
        ClassLoaderDataGraph::clear_claimed_marks();
```

清理声明的标记。

```cpp
// hotspot/share/classfile/classLoaderData.cpp#L434

void ClassLoaderDataGraph::clear_claimed_marks() {
  for (ClassLoaderData* cld = _head; cld != NULL; cld = cld->next()) {
    cld->clear_claimed();
  }
}
```

## 2. Concurrent Scan Root Regions

并行扫描根 region。

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L293

        G1ConcPhase p(G1ConcurrentPhase::SCAN_ROOT_REGIONS, this);
        _cm->scan_root_regions();
```

```cpp
// hotspot/share/gc/g1/g1ConcurrentMark.cpp#L928

void G1ConcurrentMark::scan_root_regions() {
  // scan_in_progress() will have been set to true only if there was
  // at least one root region to scan. So, if it's false, we
  // should not attempt to do any further work.
  if (root_regions()->scan_in_progress()) {
    assert(!has_aborted(), "Aborting before root region scanning is finished not supported.");

    _num_concurrent_workers = MIN2(calc_active_marking_workers(),
                                   // We distribute work on a per-region basis, so starting
                                   // more threads than that is useless.
                                   root_regions()->num_root_regions());
    assert(_num_concurrent_workers <= _max_concurrent_workers,
           "Maximum number of marking threads exceeded");

    G1CMRootRegionScanTask task(this);
    log_debug(gc, ergo)("Running %s using %u workers for %u work units.",
                        task.name(), _num_concurrent_workers, root_regions()->num_root_regions());
    _concurrent_workers->run_task(&task, _num_concurrent_workers);

    // It's possible that has_aborted() is true here without actually
    // aborting the survivor scan earlier. This is OK as it's
    // mainly used for sanity checking.
    root_regions()->scan_finished();
  }
}
```

如果 survivor 区非空的话才进行根扫描。

扫描的就是 survivor 区。


## 3. Concurrent Mark

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L310
          // Concurrent marking.
          {
            G1ConcPhase p(G1ConcurrentPhase::MARK_FROM_ROOTS, this);
            _cm->mark_from_roots();
          }
```

具体的标记逻辑封装在 `G1CMTask` 中。


```cpp
// hotspot/share/gc/g1/g1ConcurrentMark.cpp#L2512


void G1CMTask::do_marking_step(double time_target_ms,
                               bool do_termination,
                               bool is_serial) {
  assert(time_target_ms >= 1.0, "minimum granularity is 1ms");
  assert(_concurrent == _cm->concurrent(), "they should be the same");

  _start_time_ms = os::elapsedVTime() * 1000.0;

  // If do_stealing is true then do_marking_step will attempt to
  // steal work from the other G1CMTasks. It only makes sense to
  // enable stealing when the termination protocol is enabled
  // and do_marking_step() is not being called serially.
  bool do_stealing = do_termination && !is_serial;

  double diff_prediction_ms = _g1h->g1_policy()->predictor().get_new_prediction(&_marking_step_diffs_ms);
  _time_target_ms = time_target_ms - diff_prediction_ms;

  // set up the variables that are used in the work-based scheme to
  // call the regular clock method
  _words_scanned = 0;
  _refs_reached  = 0;
  recalculate_limits();

  // clear all flags
  clear_has_aborted();
  _has_timed_out = false;
  _draining_satb_buffers = false;

  ++_calls;

  // Set up the bitmap and oop closures. Anything that uses them is
  // eventually called from this method, so it is OK to allocate these
  // statically.
  G1CMBitMapClosure bitmap_closure(this, _cm);
  G1CMOopClosure    cm_oop_closure(_g1h, _cm, this);
  set_cm_oop_closure(&cm_oop_closure);

  if (_cm->has_overflown()) {
    // This can happen if the mark stack overflows during a GC pause
    // and this task, after a yield point, restarts. We have to abort
    // as we need to get into the overflow protocol which happens
    // right at the end of this task.
    set_has_aborted();
  }

  // First drain any available SATB buffers. After this, we will not
  // look at SATB buffers before the next invocation of this method.
  // If enough completed SATB buffers are queued up, the regular clock
  // will abort this task so that it restarts.
  drain_satb_buffers();
  // ...then partially drain the local queue and the global stack
  drain_local_queue(true);
  drain_global_stack(true);

  do {
    if (!has_aborted() && _curr_region != NULL) {
      // This means that we're already holding on to a region.
      assert(_finger != NULL, "if region is not NULL, then the finger "
             "should not be NULL either");

      // We might have restarted this task after an evacuation pause
      // which might have evacuated the region we're holding on to
      // underneath our feet. Let's read its limit again to make sure
      // that we do not iterate over a region of the heap that
      // contains garbage (update_region_limit() will also move
      // _finger to the start of the region if it is found empty).
      update_region_limit();
      // We will start from _finger not from the start of the region,
      // as we might be restarting this task after aborting half-way
      // through scanning this region. In this case, _finger points to
      // the address where we last found a marked object. If this is a
      // fresh region, _finger points to start().
      MemRegion mr = MemRegion(_finger, _region_limit);

      assert(!_curr_region->is_humongous() || mr.start() == _curr_region->bottom(),
             "humongous regions should go around loop once only");

      // Some special cases:
      // If the memory region is empty, we can just give up the region.
      // If the current region is humongous then we only need to check
      // the bitmap for the bit associated with the start of the object,
      // scan the object if it's live, and give up the region.
      // Otherwise, let's iterate over the bitmap of the part of the region
      // that is left.
      // If the iteration is successful, give up the region.
      if (mr.is_empty()) {
        giveup_current_region();
        regular_clock_call();
      } else if (_curr_region->is_humongous() && mr.start() == _curr_region->bottom()) {
        if (_next_mark_bitmap->is_marked(mr.start())) {
          // The object is marked - apply the closure
          bitmap_closure.do_addr(mr.start());
        }
        // Even if this task aborted while scanning the humongous object
        // we can (and should) give up the current region.
        giveup_current_region();
        regular_clock_call();
      } else if (_next_mark_bitmap->iterate(&bitmap_closure, mr)) {
        giveup_current_region();
        regular_clock_call();
      } else {
        assert(has_aborted(), "currently the only way to do so");
        // The only way to abort the bitmap iteration is to return
        // false from the do_bit() method. However, inside the
        // do_bit() method we move the _finger to point to the
        // object currently being looked at. So, if we bail out, we
        // have definitely set _finger to something non-null.
        assert(_finger != NULL, "invariant");

        // Region iteration was actually aborted. So now _finger
        // points to the address of the object we last scanned. If we
        // leave it there, when we restart this task, we will rescan
        // the object. It is easy to avoid this. We move the finger by
        // enough to point to the next possible object header.
        assert(_finger < _region_limit, "invariant");
        HeapWord* const new_finger = _finger + ((oop)_finger)->size();
        // Check if bitmap iteration was aborted while scanning the last object
        if (new_finger >= _region_limit) {
          giveup_current_region();
        } else {
          move_finger_to(new_finger);
        }
      }
    }
    // At this point we have either completed iterating over the
    // region we were holding on to, or we have aborted.

    // We then partially drain the local queue and the global stack.
    // (Do we really need this?)
    drain_local_queue(true);
    drain_global_stack(true);

    // Read the note on the claim_region() method on why it might
    // return NULL with potentially more regions available for
    // claiming and why we have to check out_of_regions() to determine
    // whether we're done or not.
    while (!has_aborted() && _curr_region == NULL && !_cm->out_of_regions()) {
      // We are going to try to claim a new region. We should have
      // given up on the previous one.
      // Separated the asserts so that we know which one fires.
      assert(_curr_region  == NULL, "invariant");
      assert(_finger       == NULL, "invariant");
      assert(_region_limit == NULL, "invariant");
      HeapRegion* claimed_region = _cm->claim_region(_worker_id);
      if (claimed_region != NULL) {
        // Yes, we managed to claim one
        setup_for_region(claimed_region);
        assert(_curr_region == claimed_region, "invariant");
      }
      // It is important to call the regular clock here. It might take
      // a while to claim a region if, for example, we hit a large
      // block of empty regions. So we need to call the regular clock
      // method once round the loop to make sure it's called
      // frequently enough.
      regular_clock_call();
    }

    if (!has_aborted() && _curr_region == NULL) {
      assert(_cm->out_of_regions(),
             "at this point we should be out of regions");
    }
  } while ( _curr_region != NULL && !has_aborted());

  if (!has_aborted()) {
    // We cannot check whether the global stack is empty, since other
    // tasks might be pushing objects to it concurrently.
    assert(_cm->out_of_regions(),
           "at this point we should be out of regions");
    // Try to reduce the number of available SATB buffers so that
    // remark has less work to do.
    drain_satb_buffers();
  }

  // Since we've done everything else, we can now totally drain the
  // local queue and global stack.
  drain_local_queue(false);
  drain_global_stack(false);

  // Attempt at work stealing from other task's queues.
  if (do_stealing && !has_aborted()) {
    // We have not aborted. This means that we have finished all that
    // we could. Let's try to do some stealing...

    // We cannot check whether the global stack is empty, since other
    // tasks might be pushing objects to it concurrently.
    assert(_cm->out_of_regions() && _task_queue->size() == 0,
           "only way to reach here");
    while (!has_aborted()) {
      G1TaskQueueEntry entry;
      if (_cm->try_stealing(_worker_id, &_hash_seed, entry)) {
        scan_task_entry(entry);

        // And since we're towards the end, let's totally drain the
        // local queue and global stack.
        drain_local_queue(false);
        drain_global_stack(false);
      } else {
        break;
      }
    }
  }

  // We still haven't aborted. Now, let's try to get into the
  // termination protocol.
  if (do_termination && !has_aborted()) {
    // We cannot check whether the global stack is empty, since other
    // tasks might be concurrently pushing objects on it.
    // Separated the asserts so that we know which one fires.
    assert(_cm->out_of_regions(), "only way to reach here");
    assert(_task_queue->size() == 0, "only way to reach here");
    _termination_start_time_ms = os::elapsedVTime() * 1000.0;

    // The G1CMTask class also extends the TerminatorTerminator class,
    // hence its should_exit_termination() method will also decide
    // whether to exit the termination protocol or not.
    bool finished = (is_serial ||
                     _cm->terminator()->offer_termination(this));
    double termination_end_time_ms = os::elapsedVTime() * 1000.0;
    _termination_time_ms +=
      termination_end_time_ms - _termination_start_time_ms;

    if (finished) {
      // We're all done.

      if (_worker_id == 0) {
        // Let's allow task 0 to do this
        if (_concurrent) {
          assert(_cm->concurrent_marking_in_progress(), "invariant");
          // We need to set this to false before the next
          // safepoint. This way we ensure that the marking phase
          // doesn't observe any more heap expansions.
          _cm->clear_concurrent_marking_in_progress();
        }
      }

      // We can now guarantee that the global stack is empty, since
      // all other tasks have finished. We separated the guarantees so
      // that, if a condition is false, we can immediately find out
      // which one.
      guarantee(_cm->out_of_regions(), "only way to reach here");
      guarantee(_cm->mark_stack_empty(), "only way to reach here");
      guarantee(_task_queue->size() == 0, "only way to reach here");
      guarantee(!_cm->has_overflown(), "only way to reach here");
    } else {
      // Apparently there's more work to do. Let's abort this task. It
      // will restart it and we can hopefully find more things to do.
      set_has_aborted();
    }
  }

  // Mainly for debugging purposes to make sure that a pointer to the
  // closure which was statically allocated in this frame doesn't
  // escape it by accident.
  set_cm_oop_closure(NULL);
  double end_time_ms = os::elapsedVTime() * 1000.0;
  double elapsed_time_ms = end_time_ms - _start_time_ms;
  // Update the step history.
  _step_times_ms.add(elapsed_time_ms);

  if (has_aborted()) {
    // The task was aborted for some reason.
    if (_has_timed_out) {
      double diff_ms = elapsed_time_ms - _time_target_ms;
      // Keep statistics of how well we did with respect to hitting
      // our target only if we actually timed out (if we aborted for
      // other reasons, then the results might get skewed).
      _marking_step_diffs_ms.add(diff_ms);
    }

    if (_cm->has_overflown()) {
      // This is the interesting one. We aborted because a global
      // overflow was raised. This means we have to restart the
      // marking phase and start iterating over regions. However, in
      // order to do this we have to make sure that all tasks stop
      // what they are doing and re-initialize in a safe manner. We
      // will achieve this with the use of two barrier sync points.

      if (!is_serial) {
        // We only need to enter the sync barrier if being called
        // from a parallel context
        _cm->enter_first_sync_barrier(_worker_id);

        // When we exit this sync barrier we know that all tasks have
        // stopped doing marking work. So, it's now safe to
        // re-initialize our data structures. At the end of this method,
        // task 0 will clear the global data structures.
      }

      // We clear the local state of this task...
      clear_region_fields();

      if (!is_serial) {
        // ...and enter the second barrier.
        _cm->enter_second_sync_barrier(_worker_id);
      }
      // At this point, if we're during the concurrent phase of
      // marking, everything has been re-initialized and we're
      // ready to restart.
    }
  }
}
```

使用 SATB 算法标记各个 region。

## 4. Pause Remark

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L323

          // Delay remark pause for MMU.
          double mark_end_time = os::elapsedVTime();
          jlong mark_end = os::elapsed_counter();
          _vtime_mark_accum += (mark_end_time - cycle_start);
          delay_to_keep_mmu(g1_policy, true /* remark */);
          if (cm()->has_aborted()) break;

          // Pause Remark.
          log_info(gc, marking)("%s (%.3fs, %.3fs) %.3fms",
                                cm_title,
                                TimeHelper::counter_to_seconds(mark_start),
                                TimeHelper::counter_to_seconds(mark_end),
                                TimeHelper::counter_to_millis(mark_end - mark_start));
          mark_manager.set_phase(G1ConcurrentPhase::REMARK, false);
          CMCheckpointRootsFinalClosure final_cl(_cm);
          VM_CGC_Operation op(&final_cl, "Pause Remark");
          VMThread::execute(&op);
```

首先等待 MMU(Minimum Mutator Utilisation) 达到条件，默认为 1 - MaxGCPauseMillis/GCPauseIntervalMillis。

Remark 是 STW 操作，具体逻辑封装在 `CMCheckpointRootsFinalClosure` 中。

```cpp
// hotspot/share/gc/g1/g1ConcurrentMark.cpp#L1002

void G1ConcurrentMark::checkpoint_roots_final(bool clear_all_soft_refs) {
  // world is stopped at this checkpoint
  assert(SafepointSynchronize::is_at_safepoint(),
         "world should be stopped");

  G1CollectedHeap* g1h = G1CollectedHeap::heap();

  // If a full collection has happened, we shouldn't do this.
  if (has_aborted()) {
    g1h->collector_state()->set_mark_in_progress(false); // So bitmap clearing isn't confused
    return;
  }

  SvcGCMarker sgcm(SvcGCMarker::OTHER);

  if (VerifyDuringGC) {
    g1h->verifier()->verify(G1HeapVerifier::G1VerifyRemark, VerifyOption_G1UsePrevMarking, "During GC (before)");
  }
  g1h->verifier()->check_bitmaps("Remark Start");

  G1Policy* g1p = g1h->g1_policy();
  g1p->record_concurrent_mark_remark_start();

  double start = os::elapsedTime();

  checkpoint_roots_final_work();

  double mark_work_end = os::elapsedTime();

  weak_refs_work(clear_all_soft_refs);

  if (has_overflown()) {
    // We overflowed.  Restart concurrent marking.
    _restart_for_overflow = true;

    // Verify the heap w.r.t. the previous marking bitmap.
    if (VerifyDuringGC) {
      g1h->verifier()->verify(G1HeapVerifier::G1VerifyRemark, VerifyOption_G1UsePrevMarking, "During GC (overflow)");
    }

    // Clear the marking state because we will be restarting
    // marking due to overflowing the global mark stack.
    reset_marking_state();
  } else {
    SATBMarkQueueSet& satb_mq_set = JavaThread::satb_mark_queue_set();
    // We're done with marking.
    // This is the end of  the marking cycle, we're expected all
    // threads to have SATB queues with active set to true.
    satb_mq_set.set_active_all_threads(false, /* new active value */
                                       true /* expected_active */);

    if (VerifyDuringGC) {
      g1h->verifier()->verify(G1HeapVerifier::G1VerifyRemark, VerifyOption_G1UseNextMarking, "During GC (after)");
    }
    g1h->verifier()->check_bitmaps("Remark End");
    assert(!restart_for_overflow(), "sanity");
    // Completely reset the marking state since marking completed
    set_non_marking_state();
  }

  // Statistics
  double now = os::elapsedTime();
  _remark_mark_times.add((mark_work_end - start) * 1000.0);
  _remark_weak_ref_times.add((now - mark_work_end) * 1000.0);
  _remark_times.add((now - start) * 1000.0);

  g1p->record_concurrent_mark_remark_end();

  G1CMIsAliveClosure is_alive(g1h);
  _gc_tracer_cm->report_object_count_after_gc(&is_alive);
}
```

处理 SATB 缓冲区，已经并发标记阶段的漏网之鱼。

## 5. Concurrent Create Live Data

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L354
        G1ConcPhase p(G1ConcurrentPhase::CREATE_LIVE_DATA, this);
        cm()->create_live_data();
```

## 6. Pause Cleanup

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L365

        delay_to_keep_mmu(g1_policy, false /* cleanup */);

        if (!cm()->has_aborted()) {
          CMCleanUp cl_cl(_cm);
          VM_CGC_Operation op(&cl_cl, "Pause Cleanup");
          VMThread::execute(&op);
        }
```

也是 STW 操作，具体逻辑封装在 CMCleanUp 中

```cpp
// hotspot/share/gc/g1/g1ConcurrentMark.cpp#L1171

void G1ConcurrentMark::cleanup() {
  // world is stopped at this checkpoint
  assert(SafepointSynchronize::is_at_safepoint(),
         "world should be stopped");
  G1CollectedHeap* g1h = G1CollectedHeap::heap();

  // If a full collection has happened, we shouldn't do this.
  if (has_aborted()) {
    g1h->collector_state()->set_mark_in_progress(false); // So bitmap clearing isn't confused
    return;
  }

  g1h->verifier()->verify_region_sets_optional();

  if (VerifyDuringGC) {
    g1h->verifier()->verify(G1HeapVerifier::G1VerifyCleanup, VerifyOption_G1UsePrevMarking, "During GC (before)");
  }
  g1h->verifier()->check_bitmaps("Cleanup Start");

  G1Policy* g1p = g1h->g1_policy();
  g1p->record_concurrent_mark_cleanup_start();

  double start = os::elapsedTime();

  HeapRegionRemSet::reset_for_cleanup_tasks();

  {
    GCTraceTime(Debug, gc)("Finalize Live Data");
    finalize_live_data();
  }

  if (VerifyDuringGC) {
    GCTraceTime(Debug, gc)("Verify Live Data");
    verify_live_data();
  }

  g1h->collector_state()->set_mark_in_progress(false);

  double count_end = os::elapsedTime();
  double this_final_counting_time = (count_end - start);
  _total_counting_time += this_final_counting_time;

  if (log_is_enabled(Trace, gc, liveness)) {
    G1PrintRegionLivenessInfoClosure cl("Post-Marking");
    _g1h->heap_region_iterate(&cl);
  }

  // Install newly created mark bitMap as "prev".
  swap_mark_bitmaps();

  g1h->reset_gc_time_stamp();

  uint n_workers = _g1h->workers()->active_workers();

  // Note end of marking in all heap regions.
  G1ParNoteEndTask g1_par_note_end_task(g1h, &_cleanup_list, n_workers);
  g1h->workers()->run_task(&g1_par_note_end_task);
  g1h->check_gc_time_stamps();

  if (!cleanup_list_is_empty()) {
    // The cleanup list is not empty, so we'll have to process it
    // concurrently. Notify anyone else that might be wanting free
    // regions that there will be more free regions coming soon.
    g1h->set_free_regions_coming();
  }

  // call below, since it affects the metric by which we sort the heap
  // regions.
  if (G1ScrubRemSets) {
    double rs_scrub_start = os::elapsedTime();
    g1h->scrub_rem_set();
    _total_rs_scrub_time += (os::elapsedTime() - rs_scrub_start);
  }

  // this will also free any regions totally full of garbage objects,
  // and sort the regions.
  g1h->g1_policy()->record_concurrent_mark_cleanup_end();

  // Statistics.
  double end = os::elapsedTime();
  _cleanup_times.add((end - start) * 1000.0);

  // Clean up will have freed any regions completely full of garbage.
  // Update the soft reference policy with the new heap occupancy.
  Universe::update_heap_info_at_gc();

  if (VerifyDuringGC) {
    g1h->verifier()->verify(G1HeapVerifier::G1VerifyCleanup, VerifyOption_G1UsePrevMarking, "During GC (after)");
  }

  g1h->verifier()->check_bitmaps("Cleanup End");

  g1h->verifier()->verify_region_sets_optional();

  // We need to make this be a "collection" so any collection pause that
  // races with it goes around and waits for completeCleanup to finish.
  g1h->increment_total_collections();

  // Clean out dead classes and update Metaspace sizes.
  if (ClassUnloadingWithConcurrentMark) {
    ClassLoaderDataGraph::purge();
  }
  MetaspaceGC::compute_new_size();

  // We reclaimed old regions so we should calculate the sizes to make
  // sure we update the old gen/space data.
  g1h->g1mm()->update_sizes();
  g1h->allocation_context_stats().update_after_mark();
}
```

交换 mark bitmaps，整理堆，调整清理RSet。

## 7. Concurrent Complete Cleanup

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L379

      // Check if cleanup set the free_regions_coming flag. If it
      // hasn't, we can just skip the next step.
      if (g1h->free_regions_coming()) {
        // The following will finish freeing up any regions that we
        // found to be empty during cleanup. We'll do this part
        // without joining the suspendible set. If an evacuation pause
        // takes place, then we would carry on freeing regions in
        // case they are needed by the pause. If a Full GC takes
        // place, it would wait for us to process the regions
        // reclaimed by cleanup.

        // Now do the concurrent cleanup operation.
        G1ConcPhase p(G1ConcurrentPhase::COMPLETE_CLEANUP, this);
        _cm->complete_cleanup();

        // Notify anyone who's waiting that there are no more free
        // regions coming. We have to do this before we join the STS
        // (in fact, we should not attempt to join the STS in the
        // interval between finishing the cleanup pause and clearing
        // the free_regions_coming flag) otherwise we might deadlock:
        // a GC worker could be blocked waiting for the notification
        // whereas this thread will be blocked for the pause to finish
        // while it's trying to join the STS, which is conditional on
        // the GC workers finishing.
        g1h->reset_free_regions_coming();
      }
```

回收空 region


## 8. Concurrent Cleanup for Next Mark

```cpp
// hotspot/share/gc/g1/concurrentMarkThread.cpp#L446

        G1ConcPhase p(G1ConcurrentPhase::CLEANUP_FOR_NEXT_MARK, this);
        _cm->cleanup_for_next_mark();
```

清理 bitmap 和 live data。
