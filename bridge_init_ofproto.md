
    (vswitchd/bridge.c)
    // 参数中， cfg 为 Open_VSwitch 表的内容
    static void
    bridge_init_ofproto(const struct ovsrec_open_vswitch *cfg)
    {
        struct shash iface_hints;
        static bool initialized = false;
        int i;

        if (initialized) {
            return;
        }

        shash_init(&iface_hints);

        if (cfg) {
            // 遍历 bridge
            for (i = 0; i < cfg->n_bridges; i++) {
                const struct ovsrec_bridge *br_cfg = cfg->bridges[i];
                int j;
                // 遍历 port
                for (j = 0; j < br_cfg->n_ports; j++) {
                    struct ovsrec_port *port_cfg = br_cfg->ports[j];
                    int k;
                    // 遍历 interface
                    for (k = 0; k < port_cfg->n_interfaces; k++) {
                        struct ovsrec_interface *if_cfg = port_cfg->interfaces[k];
                        struct iface_hint *iface_hint;

                        iface_hint = xmalloc(sizeof *iface_hint);
                        iface_hint->br_name = br_cfg->name;
                        iface_hint->br_type = br_cfg->datapath_type;
                        // 这里是重点，看 interface 中有没有使用 ofport_request 指定 openflow 端口号
                        iface_hint->ofp_port = iface_pick_ofport(if_cfg);
                        // interface name -> iface_hint 的映射
                        shash_add(&iface_hints, if_cfg->name, iface_hint);
                    }
                }
            }
        }

        ofproto_init(&iface_hints);

        shash_destroy_free_data(&iface_hints);
        initialized = true;
    }

