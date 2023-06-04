
	(ofproto/ofproto-dpif-upcall.c）
	/* Starts the handler and revalidator threads. */
    // 参数中，n_handlers_ 为期望的 handler 线程数目
    // n_revalidators_ 为期望的 revalidator 线程数目
	static void
	udpif_start_threads(struct udpif *udpif, uint32_t n_handlers_,
	                    uint32_t n_revalidators_)
	{
	    if (udpif && n_handlers_ && n_revalidators_) {
	        /* Creating a thread can take a significant amount of time on some
	         * systems, even hundred of milliseconds, so quiesce around it. */
	        ovsrcu_quiesce_start();
	
	        udpif->n_handlers = n_handlers_;
	        udpif->n_revalidators = n_revalidators_;
			// 每个 handler 结构代表一个 handler 线程
	        udpif->handlers = xzalloc(udpif->n_handlers * sizeof *udpif->handlers);
	        for (size_t i = 0; i < udpif->n_handlers; i++) {
	            struct handler *handler = &udpif->handlers[i];
	
	            handler->udpif = udpif;
	            handler->handler_id = i;
	            handler->thread = ovs_thread_create(
	                "handler", udpif_upcall_handler, handler);
	        }
	
	        atomic_init(&udpif->enable_ufid, udpif->backer->rt_support.ufid);
	        dpif_enable_upcall(udpif->dpif);
	
	        ovs_barrier_init(&udpif->reval_barrier, udpif->n_revalidators);
	        ovs_barrier_init(&udpif->pause_barrier, udpif->n_revalidators + 1);
	        udpif->reval_exit = false;
	        udpif->pause = false;
	        udpif->offload_rebalance_time = time_msec();
            // 每个 revalidator 结构代表一个 revalidator 线程
	        udpif->revalidators = xzalloc(udpif->n_revalidators
	                                      * sizeof *udpif->revalidators);
	        for (size_t i = 0; i < udpif->n_revalidators; i++) {
	            struct revalidator *revalidator = &udpif->revalidators[i];
	
	            revalidator->udpif = udpif;
	            revalidator->thread = ovs_thread_create(
	                "revalidator", udpif_revalidator, revalidator);
	        }
	        ovsrcu_quiesce_end();
	    }
	}


