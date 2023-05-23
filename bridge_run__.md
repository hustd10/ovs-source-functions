
	(vswitchd/bridge.c)
	static void
	bridge_run__(void)
	{
	    struct bridge *br;
	    struct sset types;
	    const char *type;
	
	    /* Let each datapath type do the work that it needs to do. */
	    sset_init(&types);
        // 枚举所有注册的 datapath types，保存在 types 变量中
	    ofproto_enumerate_types(&types);
		// 遍历每个 datapath type，调用 ofproto_type_run() 处理
	    SSET_FOR_EACH (type, &types) {
	        ofproto_type_run(type);
	    }
	    sset_destroy(&types);
	
	    /* Let each bridge do the work that it needs to do. */
		// 遍历所有的 bridge 进行处理
	    HMAP_FOR_EACH (br, node, &all_bridges) {
	        ofproto_run(br->ofproto);
	    }
	}

