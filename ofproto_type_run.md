
	(ofproto/ofproto.c)
	// 参数中，datapath_type 为 datapath
	int
	ofproto_type_run(const char *datapath_type)
	{
	    const struct ofproto_class *class;
	    int error;
	
	    datapath_type = ofproto_normalize_type(datapath_type);
		// 根据 datapath_type 获取对应的 ofproto_class
	    class = ofproto_class_find__(datapath_type);
		// 调用 ofproto_class->type_run()
	    error = class->type_run ? class->type_run(datapath_type) : 0;
	    if (error && error != EAGAIN) {
	        VLOG_ERR_RL(&rl, "%s: type_run failed (%s)",
	                    datapath_type, ovs_strerror(error));
	    }
	    return error;
	}

