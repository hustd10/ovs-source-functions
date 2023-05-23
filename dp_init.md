
	（datapath.c）
	static int __init dp_init(void)
	{
        int err;

        BUILD_BUG_ON(sizeof(struct ovs_skb_cb) > FIELD_SIZEOF(struct sk_buff, cb));

        pr_info("Open vSwitch switching datapath %s\n", VERSION);

        err = action_fifos_init();
        if (err)
                goto error;

        err = ovs_internal_dev_rtnl_link_register();
        if (err)
                goto error_action_fifos_exit;

        err = ovs_flow_init();
        if (err)
                goto error_unreg_rtnl_link;

        err = ovs_vport_init();
        if (err)
                goto error_flow_exit;

        err = register_pernet_device(&ovs_net_ops);
        if (err)
                goto error_vport_exit;

        err = compat_init();
        if (err)
                goto error_netns_exit;

        err = register_netdevice_notifier(&ovs_dp_device_notifier);
        if (err)
                goto error_compat_exit;

        err = ovs_netdev_init();
        if (err)
                goto error_unreg_notifier;

        err = dp_register_genl();
        if (err < 0)
                goto error_unreg_netdev;
        err = init_filter_sketch(10,2,10);
        if(err)
                goto error_filter_sketch;
        err = sketch_report_init();
        if(err)
                goto error_sketch_report;
        //countmax = new_countmax_sketch(100,2);

        return 0;
	error_sketch_report:
        sketch_report_clean();
	error_filter_sketch:
        clean_filter_sketch();
	error_unreg_netdev:
        ovs_netdev_exit();
	error_unreg_notifier:
        unregister_netdevice_notifier(&ovs_dp_device_notifier);
	error_compat_exit:
        compat_exit();
	error_netns_exit:
        unregister_pernet_device(&ovs_net_ops);
	error_vport_exit:
        ovs_vport_exit();
	error_flow_exit:
        ovs_flow_exit();
	error_unreg_rtnl_link:
        ovs_internal_dev_rtnl_link_unregister();
	error_action_fifos_exit:
        action_fifos_exit();
	error:
        return err;
	}


