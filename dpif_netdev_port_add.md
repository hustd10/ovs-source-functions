
    （lib/dpif-netdev.c）
    // 把 netdev 添加为 ofproto 的 port
    // 参数中，netdev 为待添加的对象
    // port_nop 返回 openflow port number 的值
    static int
    dpif_netdev_port_add(struct dpif *dpif, struct netdev *netdev,
                        odp_port_t *port_nop)
    {
        struct dp_netdev *dp = get_dp_netdev(dpif);
        char namebuf[NETDEV_VPORT_NAME_BUFSIZE];
        const char *dpif_port;
        odp_port_t port_no;
        int error;

        ovs_rwlock_wrlock(&dp->port_rwlock);
        dpif_port = netdev_vport_get_dpif_port(netdev, namebuf, sizeof namebuf);
        // 如果指定了 port number，检查是否被占用
        if (*port_nop != ODPP_NONE) {
            port_no = *port_nop;
            error = dp_netdev_lookup_port(dp, *port_nop) ? EBUSY : 0;
        } else {
            // 否则自动分配一个
            port_no = choose_port(dp, dpif_port);
            error = port_no == ODPP_NONE ? EFBIG : 0;
        }
        if (!error) {
            *port_nop = port_no;
            // 在 datapath（dp)中创建 dp_netdev_port 对象
            error = do_add_port(dp, dpif_port, netdev_get_type(netdev), port_no);
        }
        ovs_rwlock_unlock(&dp->port_rwlock);

        return error;
    }

