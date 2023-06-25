
    （lib/dpif-netdev.c）
    // 参数中，
    // dp 为 netdev 类型的 datapath 对象
    // devname 为待添加的端口名称
    // type 为端口类型，对应的是 netdev_class 中的 type 字段，如 dpdk
    // port_no 为 openflow 端口的编号值
    static int
    do_add_port(struct dp_netdev *dp, const char *devname, const char *type,
                odp_port_t port_no)
        OVS_REQ_WRLOCK(dp->port_rwlock)
    {
        struct netdev_saved_flags *sf;
        struct dp_netdev_port *port;
        int error;

        /* Reject devices already in 'dp'. */
        // 检查对应名称的端口是否已经存在
        if (!get_port_by_name(dp, devname, &port)) {
            return EEXIST;
        }

        // 创建对应的 port
        error = port_create(devname, type, port_no, &port);
        if (error) {
            return error;
        }

        // 将 port 保存到 dp->ports 中
        hmap_insert(&dp->ports, &port->node, hash_port_no(port_no));
        seq_change(dp->port_seq);

        // 重新配置 datapath
        reconfigure_datapath(dp);

        /* Check that port was successfully configured. */
        if (!dp_netdev_lookup_port(dp, port_no)) {
            return EINVAL;
        }

        /* Updating device flags triggers an if_notifier, which triggers a bridge
        * reconfiguration and another attempt to add this port, leading to an
        * infinite loop if the device is configured incorrectly and cannot be
        * added.  Setting the promisc mode after a successful reconfiguration,
        * since we already know that the device is somehow properly configured. */
        error = netdev_turn_flags_on(port->netdev, NETDEV_PROMISC, &sf);
        if (error) {
            VLOG_ERR("%s: cannot set promisc flag", devname);
            do_del_port(dp, port);
            return error;
        }
        port->sf = sf;

        return 0;
    }
