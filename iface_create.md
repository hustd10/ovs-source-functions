
    （vswitchd/bridge.c）
    /* Creates a new iface on 'br' based on 'if_cfg'.  The new iface has OpenFlow
    * port number 'ofp_port'.  If ofp_port is OFPP_NONE, an OpenFlow port is
    * automatically allocated for the iface.  Takes ownership of and
    * deallocates 'if_cfg'.
    *
    * Return true if an iface is successfully created, false otherwise. */
    // 参数中，br 为对应的网桥
    // iface_cfg 为待创建的 interface 的 ovsdb 中的配置
    // port_cfg 为 interface 所属的 port 的 ovsdb 中的配置
    // 返回 true 时表示创建成功，否则表示失败
    static bool
    iface_create(struct bridge *br, const struct ovsrec_interface *iface_cfg,
                const struct ovsrec_port *port_cfg)
    {
        struct netdev *netdev;
        struct iface *iface;
        ofp_port_t ofp_port;
        struct port *port;
        char *errp = NULL;
        int error;

        /* Do the bits that can fail up front. */
        // 首先确保 interface 不存在
        ovs_assert(!iface_lookup(br, iface_cfg->name));
        error = iface_do_create(br, iface_cfg, &ofp_port, &netdev, &errp);
        if (error) {
            iface_clear_db_record(iface_cfg, errp);
            free(errp);
            return false;
        }

