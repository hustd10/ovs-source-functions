
    （vswitchd/bridge.c）
    /* Opens a network device for 'if_cfg' and configures it.  Adds the network
    * device to br->ofproto and stores the OpenFlow port number in '*ofp_portp'.
    *
    * If successful, returns 0 and stores the network device in '*netdevp'.  On
    * failure, returns a positive errno value and stores NULL in '*netdevp'. */
    // 为 interface 创建或打开一个网络设备，并对其做配置
    // 参数中，br 为输入，表示对应的网桥
    // iface_cfg 为输入，表示待创建 interface 的 ovsdb 中的配置
    // ofp_portp 为输出，表示这个 interface 对应的额 OpenFlow port number
    // netdevp 为输出，表示创建或打开的网络设备
    // errp 为输出，失败时保存错误信息
    static int
    iface_do_create(const struct bridge *br,
                    const struct ovsrec_interface *iface_cfg,
                    ofp_port_t *ofp_portp, struct netdev **netdevp,
                    char **errp)
    {
        struct netdev *netdev = NULL;
        int error;
        const char *type;

        // 检查 interface 是否是系统预留的网络设备名称，预留的不允许使用
        if (netdev_is_reserved_name(iface_cfg->name)) {
            VLOG_WARN("could not create interface %s, name is reserved",
                    iface_cfg->name);
            error = EINVAL;
            goto error;
        }

        type = ofproto_port_open_type(br->ofproto,
                                    iface_get_type(iface_cfg, br->cfg));
        // netdev_open() 是这个函数的重点，它打开一个 netdev，保存在 netdev 中
        error = netdev_open(iface_cfg->name, type, &netdev);
        if (error) {
            VLOG_WARN_BUF(errp, "could not open network device %s (%s)",
                        iface_cfg->name, ovs_strerror(error));
            goto error;
        }
        // 设置 netdev 的配置，例如 dpdk 中的 dpdk-devargs
        error = iface_set_netdev_config(iface_cfg, netdev, errp);
        if (error) {
            goto error;
        }
        // 设置 netdev 的 mtu
        iface_set_netdev_mtu(iface_cfg, netdev);

        // 获取一个 Openflow port number
        *ofp_portp = iface_pick_ofport(iface_cfg);
        // 把 netdev 作为一个 port 添加到 ofproto
        error = ofproto_port_add(br->ofproto, netdev, ofp_portp);
        if (error) {
            static struct vlog_rate_limit rl = VLOG_RATE_LIMIT_INIT(1, 5);

            *errp = xasprintf("could not add network device %s to ofproto (%s)",
                            iface_cfg->name, ovs_strerror(error));
            if (!VLOG_DROP_WARN(&rl)) {
                VLOG_WARN("%s", *errp);
            }
            goto error;
        }

        VLOG_INFO("bridge %s: added interface %s on port %d",
                br->name, iface_cfg->name, *ofp_portp);

        *netdevp = netdev;
        return 0;

    error:
        *netdevp = NULL;
        netdev_close(netdev);
        return error;
    }


