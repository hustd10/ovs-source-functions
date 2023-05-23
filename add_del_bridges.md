
这个函数同步 ovsdb 中 bridge 的信息到 vswitchd 的内存中，对于 vswitchd 内存中存在的但 ovsdb 中不存在的，调用 bridge_destroy() 从 vswitchd 的内存中删除，对于 vswitchd 内存中不存在但 ovsdb 中存在的，调用 bridge_create() 添加到 vswitchd 内存中。

	(vswitchd/bridge.c）
	// 同步 ovsdb 中 bridge 的信息到 vswitchd 的内存中
	// 参数中，cfg 为 ovsdb 中 Open_VSwitch 表的内容
	static void
	add_del_bridges(const struct ovsrec_open_vswitch *cfg)
	{
	    struct bridge *br;
	    struct shash_node *node;
	    struct shash new_br;
	    size_t i;
	
	    /* Collect new bridges' names and types. */
	    shash_init(&new_br);
		// 遍历 Open_VSwitch 表中的 bridge 信息
		// new_br 中保存 bridge_name => ovsrec_bridge
	    for (i = 0; i < cfg->n_bridges; i++) {
	        static struct vlog_rate_limit rl = VLOG_RATE_LIMIT_INIT(1, 5);
	        const struct ovsrec_bridge *br_cfg = cfg->bridges[i];
	
	        if (strchr(br_cfg->name, '/') || strchr(br_cfg->name, '\\')) {
	            /* Prevent remote ovsdb-server users from accessing arbitrary
	             * directories, e.g. consider a bridge named "../../../etc/".
	             *
	             * Prohibiting "\" is only necessary on Windows but it's no great
	             * loss elsewhere. */
	            VLOG_WARN_RL(&rl, "ignoring bridge with invalid name \"%s\"",
	                         br_cfg->name);
	        } else if (!shash_add_once(&new_br, br_cfg->name, br_cfg)) {
	            VLOG_WARN_RL(&rl, "bridge %s specified twice", br_cfg->name);
	        }
	    }
	
	    /* Get rid of deleted bridges or those whose types have changed.
	     * Update 'cfg' of bridges that still exist. */
	    // 从 vswitchd 的内存中删除 vswitchd 中存在，但 ovsdb 中不存在的 bridge
	    HMAP_FOR_EACH_SAFE (br, node, &all_bridges) {
	        br->cfg = shash_find_data(&new_br, br->name);
	        if (!br->cfg || strcmp(br->type, ofproto_normalize_type(
	                                   br->cfg->datapath_type))) {
	            bridge_destroy(br, true);
	        }
	    }
	
	    /* Add new bridges. */
		// 在 vswitchd 的内存中添加新的 bridge
	    SHASH_FOR_EACH(node, &new_br) {
	        const struct ovsrec_bridge *br_cfg = node->data;
	        if (!bridge_lookup(br_cfg->name)) {
	            bridge_create(br_cfg);
	        }
	    }
	
	    shash_destroy(&new_br);
	}


