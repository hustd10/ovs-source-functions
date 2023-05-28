
	(lib/ofp-parse.c)
    // 该函数解析匹配流表规则中指定的协议，在 parse_ofp_str__() 中被调用
	// 参数中，name 为协议名称，如 tcp
    // p_out 为输出，返回匹配到的 ofp_protocol 结构
	bool
	ofp_parse_protocol(const char *name, const struct ofp_protocol **p_out)
	{
	    static const struct ofp_protocol protocols[] = {
	        { "ip", ETH_TYPE_IP, 0 },
	        { "ipv4", ETH_TYPE_IP, 0 },
	        { "ip4", ETH_TYPE_IP, 0 },
	        { "arp", ETH_TYPE_ARP, 0 },
	        { "icmp", ETH_TYPE_IP, IPPROTO_ICMP },
	        { "tcp", ETH_TYPE_IP, IPPROTO_TCP },
	        { "udp", ETH_TYPE_IP, IPPROTO_UDP },
	        { "sctp", ETH_TYPE_IP, IPPROTO_SCTP },
	        { "ipv6", ETH_TYPE_IPV6, 0 },
	        { "ip6", ETH_TYPE_IPV6, 0 },
	        { "icmp6", ETH_TYPE_IPV6, IPPROTO_ICMPV6 },
	        { "tcp6", ETH_TYPE_IPV6, IPPROTO_TCP },
	        { "udp6", ETH_TYPE_IPV6, IPPROTO_UDP },
	        { "sctp6", ETH_TYPE_IPV6, IPPROTO_SCTP },
	        { "rarp", ETH_TYPE_RARP, 0},
	        { "mpls", ETH_TYPE_MPLS, 0 },
	        { "mplsm", ETH_TYPE_MPLS_MCAST, 0 },
	    };
	    const struct ofp_protocol *p;
	
	    for (p = protocols; p < &protocols[ARRAY_SIZE(protocols)]; p++) {
	        if (!strcmp(p->name, name)) {
	            *p_out = p;
	            return true;
	        }
	    }
	    *p_out = NULL;
	    return false;
	}

