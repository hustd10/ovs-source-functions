
    （vport.c）
    /**
    *      ovs_vport_add - add vport device (for kernel callers)
    *
    * @parms: Information about new vport.
    *
    * Creates a new vport with the specified configuration (which is dependent on
    * device type).  ovs_mutex must be held.
    */
    struct vport *ovs_vport_add(const struct vport_parms *parms)
    {
        struct vport_ops *ops;
        struct vport *vport;
        // 根据 vport 的类型查找 vport_ops
        ops = ovs_vport_lookup(parms);
        if (ops) {
                struct hlist_head *bucket;

                if (!try_module_get(ops->owner))
                        return ERR_PTR(-EAFNOSUPPORT);
                // 调用 vport_ops->create() 创建 vport
                vport = ops->create(parms);
                if (IS_ERR(vport)) {
                        module_put(ops->owner);
                        return vport;
                }
                // 插入 dp 的哈希表中
                bucket = hash_bucket(ovs_dp_get_net(vport->dp),
                                     ovs_vport_name(vport));
                hlist_add_head_rcu(&vport->hash_node, bucket);
                return vport;
        }

        /* Unlock to attempt module load and return -EAGAIN if load
         * was successful as we need to restart the port addition
         * workflow.
         */
        ovs_unlock();
        request_module("vport-type-%d", parms->type);
        ovs_lock();

        if (!ovs_vport_lookup(parms))
                return ERR_PTR(-EAFNOSUPPORT);
        else
                return ERR_PTR(-EAGAIN);
    }

