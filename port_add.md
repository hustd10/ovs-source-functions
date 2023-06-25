
    (ofproto/ofproto-dpif.c)
    static int
    port_add(struct ofproto *ofproto_, struct netdev *netdev)
    {
        struct ofproto_dpif *ofproto = ofproto_dpif_cast(ofproto_);
        // netdev 的设备名称
        const char *devname = netdev_get_name(netdev);
        char namebuf[NETDEV_VPORT_NAME_BUFSIZE];
        const char *dp_port_name;

        // 如果 vport 是 patch，添加到 ofproto->ghost_ports 中
        if (netdev_vport_is_patch(netdev)) {
            sset_add(&ofproto->ghost_ports, netdev_get_name(netdev));
            return 0;
        }
        // 获取 netdev 对应的 dpif port 名称
        dp_port_name = netdev_vport_get_dpif_port(netdev, namebuf, sizeof namebuf);
        // 如果不存在，则调用 dpif_port_add() 添加到 dpif 中
        if (!dpif_port_exists(ofproto->backer->dpif, dp_port_name)) {
            odp_port_t port_no = ODPP_NONE;
            int error;

            error = dpif_port_add(ofproto->backer->dpif, netdev, &port_no);
            if (error) {
                return error;
            }
            if (netdev_get_tunnel_config(netdev)) {
                simap_put(&ofproto->backer->tnl_backers,
                        dp_port_name, odp_to_u32(port_no));
            }
        } else {
            struct dpif *dpif = ofproto->backer->dpif;
            const char *dpif_type_str = dpif_normalize_type(dpif_type(dpif));
            netdev_set_dpif_type(netdev, dpif_type_str);
        }

        if (netdev_get_tunnel_config(netdev)) {
            sset_add(&ofproto->ghost_ports, devname);
        } else {
            sset_add(&ofproto->ports, devname);
        }
        return 0;
    }
