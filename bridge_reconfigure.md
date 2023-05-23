
	(vswitchd/bridge.c)
	// 参数中，ovs_cfg 为 ovsdb 中 Open_VSwitch 表的内容
	static void
	bridge_reconfigure(const struct ovsrec_open_vswitch *ovs_cfg)
	{
	    struct sockaddr_in *managers;
	    struct bridge *br;
	    int sflow_bridge_number;
	    size_t n_managers;
	
	    COVERAGE_INC(bridge_reconfigure);
	
	    ofproto_set_flow_limit(smap_get_uint(&ovs_cfg->other_config, "flow-limit",
	                                        OFPROTO_FLOW_LIMIT_DEFAULT));
	    ofproto_set_max_idle(smap_get_uint(&ovs_cfg->other_config, "max-idle",
	                                      OFPROTO_MAX_IDLE_DEFAULT));
	    ofproto_set_max_revalidator(smap_get_uint(&ovs_cfg->other_config,
	                                             "max-revalidator",
	                                             OFPROTO_MAX_REVALIDATOR_DEFAULT));
	    ofproto_set_min_revalidate_pps(
	        smap_get_uint(&ovs_cfg->other_config, "min-revalidate-pps",
	                     OFPROTO_MIN_REVALIDATE_PPS_DEFAULT));
	    ofproto_set_offloaded_stats_delay(
	        smap_get_uint(&ovs_cfg->other_config, "offloaded-stats-delay",
	                      OFPROTO_OFFLOADED_STATS_DELAY));
	    ofproto_set_vlan_limit(smap_get_int(&ovs_cfg->other_config, "vlan-limit",
	                                       LEGACY_MAX_VLAN_HEADERS));
	    ofproto_set_bundle_idle_timeout(smap_get_uint(&ovs_cfg->other_config,
	                                                 "bundle-idle-timeout", 0));
	    ofproto_set_threads(
	        smap_get_int(&ovs_cfg->other_config, "n-handler-threads", 0),
	        smap_get_int(&ovs_cfg->other_config, "n-revalidator-threads", 0));
	
	    /* Destroy "struct bridge"s, "struct port"s, and "struct iface"s according
	     * to 'ovs_cfg', with only very minimal configuration otherwise.
	     *
	     * This is mostly an update to bridge data structures. Nothing is pushed
	     * down to ofproto or lower layers. */
		// 根据 ovsdb 在内存中添加/删除 bridge 信息
	    add_del_bridges(ovs_cfg);
	    HMAP_FOR_EACH (br, node, &all_bridges) {
	        bridge_collect_wanted_ports(br, &br->wanted_ports);
	        bridge_del_ports(br, &br->wanted_ports);
	    }

		/* Start pushing configuration changes down to the ofproto layer:
	     *
	     *   - Delete ofprotos that are no longer configured.
	     *
	     *   - Delete ports that are no longer configured.
	     *
	     *   - Reconfigure existing ports to their desired configurations, or
	     *     delete them if not possible.
	     *
	     * We have to do all the deletions before we can do any additions, because
	     * the ports to be added might require resources that will be freed up by
	     * deletions (they might especially overlap in name). */
	    bridge_delete_ofprotos();
	    HMAP_FOR_EACH (br, node, &all_bridges) {
	        if (br->ofproto) {
	            bridge_delete_or_reconfigure_ports(br);
	        }
	    }
	
	    /* Finish pushing configuration changes to the ofproto layer:
	     *
	     *     - Create ofprotos that are missing.
	     *
	     *     - Add ports that are missing. */
	    HMAP_FOR_EACH_SAFE (br, node, &all_bridges) {
	        if (!br->ofproto) {
	            int error;
	
	            error = ofproto_create(br->name, br->type, &br->ofproto);
	            if (error) {
	                VLOG_ERR("failed to create bridge %s: %s", br->name,
	                         ovs_strerror(error));
	                shash_destroy(&br->wanted_ports);
	                bridge_destroy(br, true);
	            } else {
	                /* Trigger storing datapath version. */
	                seq_change(connectivity_seq_get());
	            }
	        }
	    }
	
	    config_ofproto_types(&ovs_cfg->other_config);

		HMAP_FOR_EACH (br, node, &all_bridges) {
	        bridge_add_ports(br, &br->wanted_ports);
	        shash_destroy(&br->wanted_ports);
	    }
	
	    reconfigure_system_stats(ovs_cfg);
	    datapath_reconfigure(ovs_cfg);
	
	    /* Complete the configuration. */
	    sflow_bridge_number = 0;
	    collect_in_band_managers(ovs_cfg, &managers, &n_managers);
	    HMAP_FOR_EACH (br, node, &all_bridges) {
	        struct port *port;
	
	        /* We need the datapath ID early to allow LACP ports to use it as the
	         * default system ID. */
	        bridge_configure_datapath_id(br);
	
	        HMAP_FOR_EACH (port, hmap_node, &br->ports) {
	            struct iface *iface;
	
	            port_configure(port);
	
	            LIST_FOR_EACH (iface, port_elem, &port->ifaces) {
	                iface_set_ofport(iface->cfg, iface->ofp_port);
	                /* Clear eventual previous errors */
	                ovsrec_interface_set_error(iface->cfg, NULL);
	                iface_configure_cfm(iface);
	                iface_configure_qos(iface, port->cfg->qos);
	                iface_set_mac(br, port, iface);
	                ofproto_port_set_bfd(br->ofproto, iface->ofp_port,
	                                     &iface->cfg->bfd);
	                ofproto_port_set_lldp(br->ofproto, iface->ofp_port,
	                                      &iface->cfg->lldp);
	                ofproto_port_set_config(br->ofproto, iface->ofp_port,
	                                        &iface->cfg->other_config);
	            }
	        }
	        bridge_configure_mirrors(br);
	        bridge_configure_forward_bpdu(br);
	        bridge_configure_mac_table(br);
	        bridge_configure_mcast_snooping(br);
	        bridge_configure_remotes(br, managers, n_managers);
	        bridge_configure_netflow(br);
	        bridge_configure_sflow(br, &sflow_bridge_number);
	        bridge_configure_ipfix(br);
	        bridge_configure_spanning_tree(br);
	        bridge_configure_tables(br);
	        bridge_configure_dp_desc(br);
	        bridge_configure_serial_desc(br);
	        bridge_configure_aa(br);
	    }
	    free(managers);

		/* The ofproto-dpif provider does some final reconfiguration in its
	     * ->type_run() function.  We have to call it before notifying the database
	     * client that reconfiguration is complete, otherwise there is a very
	     * narrow race window in which e.g. ofproto/trace will not recognize the
	     * new configuration (sometimes this causes unit test failures). */
	    bridge_run__();
	}

