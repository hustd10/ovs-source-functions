
    (datapath.c)
    static int ovs_vport_cmd_new(struct sk_buff *skb, struct genl_info *info)
    {
        struct nlattr **a = info->attrs;
        struct ovs_header *ovs_header = info->userhdr;
        struct vport_parms parms;
        struct sk_buff *reply;
        struct vport *vport;
        struct datapath *dp;
        u32 port_no;
        int err;
        // 必须指定端口名，类型和 upcall pid
        if (!a[OVS_VPORT_ATTR_NAME] || !a[OVS_VPORT_ATTR_TYPE] ||
            !a[OVS_VPORT_ATTR_UPCALL_PID])
                return -EINVAL;
        // 是否指定端口编号
        port_no = a[OVS_VPORT_ATTR_PORT_NO]
                ? nla_get_u32(a[OVS_VPORT_ATTR_PORT_NO]) : 0;
        if (port_no >= DP_MAX_PORTS)
                return -EFBIG;
        // 分配 netlink 响应的消息结构
        reply = ovs_vport_cmd_alloc_info();
        if (!reply)
                return -ENOMEM;

        ovs_lock();
    restart:
        // 获取 datapath
        dp = get_dp(sock_net(skb->sk), ovs_header->dp_ifindex);
        err = -ENODEV;
        if (!dp)
                goto exit_unlock_free;

        if (port_no) {
                // 如果指定了 port number，查找是否已经存在
                vport = ovs_vport_ovsl(dp, port_no);
                err = -EBUSY;
                if (vport)
                        goto exit_unlock_free;
        } else {
                // 如果没有指定 port number，自动查找没有使用的分配一个
                for (port_no = 1; ; port_no++) {
                        if (port_no >= DP_MAX_PORTS) {
                                err = -EFBIG;
                                goto exit_unlock_free;
                        }
                        vport = ovs_vport_ovsl(dp, port_no);
                        if (!vport)
                                break;
                }
        }
        // 新建 port 的参数
        parms.name = nla_data(a[OVS_VPORT_ATTR_NAME]);
        parms.type = nla_get_u32(a[OVS_VPORT_ATTR_TYPE]);
        parms.options = a[OVS_VPORT_ATTR_OPTIONS];
        parms.dp = dp;
        parms.port_no = port_no;
        parms.upcall_portids = a[OVS_VPORT_ATTR_UPCALL_PID];
        // 新建 port
        vport = new_vport(&parms);
        err = PTR_ERR(vport);
        if (IS_ERR(vport)) {
                if (err == -EAGAIN)
                        goto restart;
                goto exit_unlock_free;
        }
        // 填充 netlink 响应消息
        err = ovs_vport_cmd_fill_info(vport, reply, info->snd_portid,
                                      info->snd_seq, 0, OVS_VPORT_CMD_NEW);
        BUG_ON(err < 0);

        if (netdev_get_fwd_headroom(vport->dev) > dp->max_headroom)
                update_headroom(dp);
        else
                netdev_set_rx_headroom(vport->dev, dp->max_headroom);

        ovs_unlock();

        ovs_notify(&dp_vport_genl_family, &ovs_dp_vport_multicast_group, reply, info);
        return 0;

    exit_unlock_free:
        ovs_unlock();
        kfree_skb(reply);
        return err;
    }


