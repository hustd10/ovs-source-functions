
	（vswitchd/bridge.c）
	/* Deletes "struct port"s and "struct iface"s under 'br' which aren't
	 * consistent with 'br->cfg'.  Updates 'br->if_cfg_queue' with interfaces which
	 * 'br' needs to complete its configuration. */
	// 参数中，wanted_ports 为该 bridge 期望配置的端口, port_name => ovsrec_port
	static void
	bridge_del_ports(struct bridge *br, const struct shash *wanted_ports)
	{
	    struct shash_node *port_node;
	    struct port *port;
	
	    /* Get rid of deleted ports.
	     * Get rid of deleted interfaces on ports that still exist. */
	    // 遍历 bridge 现有的端口
	    HMAP_FOR_EACH_SAFE (port, hmap_node, &br->ports) {
	        port->cfg = shash_find_data(wanted_ports, port->name);
	        if (!port->cfg) {
				// 如果在期望配置中不存在，则调用 port_destroy() 从内存中删除
	            port_destroy(port);
	        } else {
				// 如果 port 存在，对比 port 的 interface 配置，删除需要删除的 interface
	            port_del_ifaces(port);
	        }
	    }
	
	    /* Update iface->cfg and iface->type in interfaces that still exist. */
	    SHASH_FOR_EACH (port_node, wanted_ports) {
	        const struct ovsrec_port *port_rec = port_node->data;
	        size_t i;
	
	        for (i = 0; i < port_rec->n_interfaces; i++) {
	            const struct ovsrec_interface *cfg = port_rec->interfaces[i];
	            struct iface *iface = iface_lookup(br, cfg->name);
	            const char *type = iface_get_type(cfg, br->cfg);
	
	            if (iface) {
	                iface->cfg = cfg;
	                iface->type = type;
	            } else {
	                /* We will add new interfaces later. */
	            }
	        }
	    }
	}


