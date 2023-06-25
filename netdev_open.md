
    （lib/netdev.c）
    /* Opens the network device named 'name' (e.g. "eth0") of the specified 'type'
    * (e.g. "system") and returns zero if successful, otherwise a positive errno
    * value.  On success, sets '*netdevp' to the new network device, otherwise to
    * null.
    *
    * Some network devices may need to be configured (with netdev_set_config())
    * before they can be used.
    *
    * Before opening rxqs or sending packets, '*netdevp' may need to be
    * reconfigured (with netdev_is_reconf_required() and netdev_reconfigure()).
    * */
    // 打开一个 network device
    // 参数中，name 为网络设备的名称，传入的是 ovs 中 interface 的名称
    // type 为类型
    // netdevp 为输出，返回网络设备的对象
    // 成功时返回0，否则返回非0
    int
    netdev_open(const char *name, const char *type, struct netdev **netdevp)
        OVS_EXCLUDED(netdev_mutex)
    {
        struct netdev *netdev;
        int error = 0;

        if (!name[0]) {
            /* Reject empty names.  This saves the providers having to do this.  At
            * least one screwed this up: the netdev-linux "tap" implementation
            * passed the name directly to the Linux TUNSETIFF call, which treats
            * an empty string as a request to generate a unique name. */
            return EINVAL;
        }

        netdev_initialize();

        ovs_mutex_lock(&netdev_mutex);
        // 查找同名的 netdev 是否已经存在
        netdev = shash_find_data(&netdev_shash, name);

        if (netdev && type && type[0]) {
            // 如果 netdev 已经存在，并且类型不一致
            if (strcmp(type, netdev->netdev_class->type)) {

                if (netdev->auto_classified) {
                    /* If this device was first created without a classification
                    * type, for example due to routing or tunneling code, and they
                    * keep a reference, a "classified" call to open will fail.
                    * In this case we remove the classless device, and re-add it
                    * below. We remove the netdev from the shash, and change the
                    * sequence, so owners of the old classless device can
                    * release/cleanup. */
                    if (netdev->node) {
                        shash_delete(&netdev_shash, netdev->node);
                        netdev->node = NULL;
                        netdev_change_seq_changed(netdev);
                    }

                    netdev = NULL;
                } else {
                    // 报错
                    error = EEXIST;
                }
            } else if (netdev->auto_classified) {
                /* If netdev reopened with type "system", clear auto_classified. */
                netdev->auto_classified = false;
            }
        }

        // netdev 不存在，创建
        if (!netdev) {
            struct netdev_registered_class *rc;

            // 根据类型查找相应的 netdev_class
            rc = netdev_lookup_class(type && type[0] ? type : "system");
            if (rc && ovs_refcount_try_ref_rcu(&rc->refcnt)) {
                // 调用 netdev_class 的 alloc() 函数创建 netdev
                netdev = rc->class->alloc();
                if (netdev) {
                    memset(netdev, 0, sizeof *netdev);
                    netdev->netdev_class = rc->class;
                    netdev->auto_classified = type && type[0] ? false : true;
                    netdev->name = xstrdup(name);
                    netdev->change_seq = 1;
                    netdev->reconfigure_seq = seq_create();
                    netdev->last_reconfigure_seq =
                        seq_read(netdev->reconfigure_seq);
                    ovsrcu_set(&netdev->flow_api, NULL);
                    netdev->hw_info.oor = false;
                    atomic_init(&netdev->hw_info.miss_api_supported, false);
                    // 添加到全局的 netdev_shash 中
                    netdev->node = shash_add(&netdev_shash, name, netdev);

                    /* By default enable one tx and rx queue per netdev. */
                    netdev->n_txq = netdev->netdev_class->send ? 1 : 0;
                    netdev->n_rxq = netdev->netdev_class->rxq_alloc ? 1 : 0;

                    ovs_list_init(&netdev->saved_flags_list);
                    // 调用 netdev_class 的 construct() 函数构造 netdev
                    error = rc->class->construct(netdev);
                    if (!error) {
                        netdev_change_seq_changed(netdev);
                    } else {
                        ovs_refcount_unref(&rc->refcnt);
                        seq_destroy(netdev->reconfigure_seq);
                        free(netdev->name);
                        ovs_assert(ovs_list_is_empty(&netdev->saved_flags_list));
                        shash_delete(&netdev_shash, netdev->node);
                        rc->class->dealloc(netdev);
                    }
                } else {
                    error = ENOMEM;
                }
            } else {
                    // 已经存在，报错
                    VLOG_WARN("could not create netdev %s of unknown type %s",
                            name, type);
                    error = EAFNOSUPPORT;
            }
        }

        if (!error) {
            netdev->ref_cnt++;
            *netdevp = netdev;
        } else {
            *netdevp = NULL;
        }
        ovs_mutex_unlock(&netdev_mutex);

        return error;
    }
