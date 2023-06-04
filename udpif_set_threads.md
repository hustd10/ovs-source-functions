
	（ofproto/ofproto-dpif-upcall.c）
	/* Tells 'udpif' how many threads it should use to handle upcalls.
	 * 'n_handlers_' and 'n_revalidators_' can never be zero.  'udpif''s
	 * datapath handle must have packet reception enabled before starting
	 * threads. */
	// 参数中，n_handlers_ 为配置的 handler 线程的数目
    // n_revalidators_ 为配置的 revalidator 线程的数目
	void
	udpif_set_threads(struct udpif *udpif, uint32_t n_handlers_,
	                  uint32_t n_revalidators_)
	{
	    ovs_assert(udpif);
	    uint32_t n_handlers_requested;
	    uint32_t n_revalidators_requested;
	    bool forced = false;
	
		// 调用 dpif_class->number_handlers_required() 函数看对应的 dpif_class 是否指定了 handler 线程的数目
	    if (dpif_number_handlers_required(udpif->dpif, &n_handlers_requested)) {
	        forced = true;
	        if (!n_revalidators_) {
	            n_revalidators_requested = n_handlers_requested / 4 + 1;
	        } else {
	            n_revalidators_requested = n_revalidators_;
	        }
	    } else {
	        int threads = MAX(count_cpu_cores(), 2);
	
	        n_revalidators_requested = MAX(n_revalidators_, 0);
	        n_handlers_requested = MAX(n_handlers_, 0);
	
	        if (!n_revalidators_requested) {
	            n_revalidators_requested = n_handlers_requested
	                    ? MAX(threads - (int) n_handlers_requested, 1)
	                    : threads / 4 + 1;
	        }
	
	        if (!n_handlers_requested) {
	            n_handlers_requested = MAX(threads -
	                                       (int) n_revalidators_requested, 1);
	        }
	    }
		// 反正到这里，n_handlers_requested 是期望的 handler 线程的数目
        // n_revalidators_requested 是期望的 revalidator 线程的数目

        // 检查是否需要调整线程数目
        // 需要调整时，先调用 udpif_stop_threads() 把当前线程停掉
        // 停掉后 udpif->n_handlers 和 udpif->revalidators 都为 0
	    if (udpif->n_handlers != n_handlers_requested
	        || udpif->n_revalidators != n_revalidators_requested) {
	        if (forced) {
	            VLOG_INFO("Overriding n-handler-threads to %u, setting "
	                      "n-revalidator-threads to %u", n_handlers_requested,
	                      n_revalidators_requested);
	        } else {
	            VLOG_INFO("Setting n-handler-threads to %u, setting "
	                      "n-revalidator-threads to %u", n_handlers_requested,
	                      n_revalidators_requested);
	        }
	        udpif_stop_threads(udpif, true);
	    }
	    // 启动新线程
	    if (!udpif->handlers && !udpif->revalidators) {
	        VLOG_INFO("Starting %u threads", n_handlers_requested +
	                                         n_revalidators_requested);
	        int error;
			// 调用 dpif_class->handlers_set() 通知其 handler 变化
	        error = dpif_handlers_set(udpif->dpif, n_handlers_requested);
	        if (error) {
	            VLOG_ERR("failed to configure handlers in dpif %s: %s",
	                     dpif_name(udpif->dpif), ovs_strerror(error));
	            return;
	        }
            // 启动 handler 和 revalidator 线程
	        udpif_start_threads(udpif, n_handlers_requested,
	                            n_revalidators_requested);
	    }
	}

