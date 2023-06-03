
	（lib/dpif.c）
	/* Tries to open a datapath with the given 'name' and 'type', creating it if it
	 * does not exist.  'type' may be either NULL or the empty string to specify
	 * the default system type.  Returns 0 if successful, otherwise a positive
	 * errno value. On success stores a pointer to the datapath in '*dpifp',
	 * otherwise a null pointer. */
	// 参数中，name 为 dpif_backer 的名称，参考 open_dpif_backer() 中，为 "ovs-"+ datapath_type，如 ovs-system
	// type 为 datapath_type
	// dpifp 为输出，返回创建的 dpif 对象
	int
	dpif_create_and_open(const char *name, const char *type, struct dpif **dpifp)
	{
	    int error;
	
	    error = dpif_create(name, type, dpifp);
	    if (error == EEXIST || error == EBUSY) {
	        error = dpif_open(name, type, dpifp);
	        if (error) {
	            VLOG_WARN("datapath %s already exists but cannot be opened: %s",
	                      name, ovs_strerror(error));
	        }
	    } else if (error) {
	        VLOG_WARN("failed to create datapath %s: %s",
	                  name, ovs_strerror(error));
	    }
	    return error;
	}
