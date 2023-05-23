
	(vswitchd/bridge.c)
	// 参数中，cfg 为 Open_VSwitch 表的内容
	static void
	datapath_reconfigure(const struct ovsrec_open_vswitch *cfg)
	{
	    struct datapath *dp;
	
	    /* Add new 'datapath's or update existing ones. */
		// 遍历 Open_VSwitch 表中的 datapath
	    for (size_t i = 0; i < cfg->n_datapaths; i++) {
	        struct ovsrec_datapath *dp_cfg = cfg->value_datapaths[i];
	        char *dp_name = cfg->key_datapaths[i];
	
	        dp = datapath_lookup(dp_name);
	        if (!dp) {
	            dp = datapath_create(dp_name);
	            dp_capability_reconfigure(dp, dp_cfg);
	        }
	        dp->last_used = idl_seqno;
	        ct_zones_reconfigure(dp, dp_cfg);
	    }
	
	    /* Purge deleted 'datapath's. */
	    HMAP_FOR_EACH_SAFE (dp, node, &all_datapaths) {
	        if (dp->last_used != idl_seqno) {
	            datapath_destroy(dp);
	        }
	    }
	}

