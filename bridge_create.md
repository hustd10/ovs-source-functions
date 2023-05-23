
	（vswitchd/bridge.c）

	// 添加 ovsdb 中的 bridge 到 vswitchd 的内存
	/* Bridge reconfiguration functions. */
	static void
	bridge_create(const struct ovsrec_bridge *br_cfg)
	{
	    struct bridge *br;
	
	    ovs_assert(!bridge_lookup(br_cfg->name));
	    br = xzalloc(sizeof *br);
	
	    br->name = xstrdup(br_cfg->name);
	    br->type = xstrdup(ofproto_normalize_type(br_cfg->datapath_type));
	    br->cfg = br_cfg;
	
	    /* Derive the default Ethernet address from the bridge's UUID.  This should
	     * be unique and it will be stable between ovs-vswitchd runs.  */
	    memcpy(&br->default_ea, &br_cfg->header_.uuid, ETH_ADDR_LEN);
	    eth_addr_mark_random(&br->default_ea);
	
	    hmap_init(&br->ports);
	    hmap_init(&br->ifaces);
	    hmap_init(&br->iface_by_name);
	    hmap_init(&br->mirrors);
	
	    hmap_init(&br->mappings);
	    hmap_insert(&all_bridges, &br->node, hash_string(br->name, 0));
	}
