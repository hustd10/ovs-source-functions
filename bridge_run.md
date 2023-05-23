
	(vswitchd/bridge.c)
	void
	bridge_run(void)
	{
	    static struct ovsrec_open_vswitch null_cfg;
	    const struct ovsrec_open_vswitch *cfg;
	
	    ovsrec_open_vswitch_init(&null_cfg);
	
	    ovsdb_idl_run(idl);
	
	    if_notifier_run();
	
	    if (ovsdb_idl_is_lock_contended(idl)) {
	        static struct vlog_rate_limit rl = VLOG_RATE_LIMIT_INIT(1, 1);
	        struct bridge *br;
	
	        VLOG_ERR_RL(&rl, "another ovs-vswitchd process is running, "
	                    "disabling this process (pid %ld) until it goes away",
	                    (long int) getpid());
	
	        HMAP_FOR_EACH_SAFE (br, node, &all_bridges) {
	            bridge_destroy(br, false);
	        }
	        /* Since we will not be running system_stats_run() in this process
	         * with the current situation of multiple ovs-vswitchd daemons,
	         * disable system stats collection. */
	        system_stats_enable(false);
	        return;
	    } else if (!ovsdb_idl_has_lock(idl)
	               || !ovsdb_idl_has_ever_connected(idl)) {
	        /* Returns if not holding the lock or not done retrieving db
	         * contents. */
	        return;
	    }
		// 获取 Open_VSwitch 表的内容
	    cfg = ovsrec_open_vswitch_first(idl);
		if (cfg) {
	        netdev_set_flow_api_enabled(&cfg->other_config);
			// 初始化 dpdk
	        dpdk_init(&cfg->other_config);
	        userspace_tso_init(&cfg->other_config);
	    }
	
	    /* Initialize the ofproto library.  This only needs to run once, but
	     * it must be done after the configuration is set.  If the
	     * initialization has already occurred, bridge_init_ofproto()
	     * returns immediately. */
	    bridge_init_ofproto(cfg);
	
	    /* Once the value of flow-restore-wait is false, we no longer should
	     * check its value from the database. */
	    if (cfg && ofproto_get_flow_restore_wait()) {
	        ofproto_set_flow_restore_wait(smap_get_bool(&cfg->other_config,
	                                        "flow-restore-wait", false));
	    }
	
	    bridge_run__();

		/* Re-configure SSL.  We do this on every trip through the main loop,
	     * instead of just when the database changes, because the contents of the
	     * key and certificate files can change without the database changing.
	     *
	     * We do this before bridge_reconfigure() because that function might
	     * initiate SSL connections and thus requires SSL to be configured. */
	    if (cfg && cfg->ssl) {
	        const struct ovsrec_ssl *ssl = cfg->ssl;
	
	        stream_ssl_set_key_and_cert(ssl->private_key, ssl->certificate);
	        stream_ssl_set_ca_cert_file(ssl->ca_cert, ssl->bootstrap_ca_cert);
	    }
	
	    if (ovsdb_idl_get_seqno(idl) != idl_seqno ||
	        if_notifier_changed(ifnotifier)) {
	        struct ovsdb_idl_txn *txn;
	
	        idl_seqno = ovsdb_idl_get_seqno(idl);
	        txn = ovsdb_idl_txn_create(idl);
	        bridge_reconfigure(cfg ? cfg : &null_cfg);
	
	        if (cfg) {
	            ovsrec_open_vswitch_set_cur_cfg(cfg, cfg->next_cfg);
	            discover_types(cfg);
	        }
	
	        /* If we are completing our initial configuration for this run
	         * of ovs-vswitchd, then keep the transaction around to monitor
	         * it for completion. */
	        if (initial_config_done) {
	            /* Always sets the 'status_txn_try_again' to check again,
	             * in case that this transaction fails. */
	            status_txn_try_again = true;
	            ovsdb_idl_txn_commit(txn);
	            ovsdb_idl_txn_destroy(txn);
	        } else {
	            initial_config_done = true;
	            daemonize_txn = txn;
	        }
	    }

		 if (daemonize_txn) {
	        enum ovsdb_idl_txn_status status = ovsdb_idl_txn_commit(daemonize_txn);
	        if (status != TXN_INCOMPLETE) {
	            ovsdb_idl_txn_destroy(daemonize_txn);
	            daemonize_txn = NULL;
	
	            /* ovs-vswitchd has completed initialization, so allow the
	             * process that forked us to exit successfully. */
	            daemonize_complete();
	
	            vlog_enable_async();
	
	            VLOG_INFO_ONCE("%s (Open vSwitch) %s", program_name, VERSION);
	        }
	    }
	
	    run_stats_update();
	    run_status_update();
	    run_system_stats();
	}


