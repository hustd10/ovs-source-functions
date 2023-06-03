
	（lib/ofp-flow.c）
	static char * OVS_WARN_UNUSED_RESULT
	parse_ofp_str__(struct ofputil_flow_mod *fm, int command, char *string,
	                const struct ofputil_port_map *port_map,
	                const struct ofputil_table_map *table_map,
	                enum ofputil_protocol *usable_protocols)
	{
	    enum {
	        F_OUT_PORT = 1 << 0,
	        F_ACTIONS = 1 << 1,
	        F_IMPORTANCE = 1 << 2,
	        F_TIMEOUT = 1 << 3,
	        F_PRIORITY = 1 << 4,
	        F_FLAGS = 1 << 5,
	    } fields; // 流表的字段
	    char *act_str = NULL;
	    char *name, *value;
	
	    *usable_protocols = OFPUTIL_P_ANY;
	
	    if (command == -2) {
	        size_t len;
	
	        string += strspn(string, " \t\r\n");   /* Skip white space. */
	        len = strcspn(string, ", \t\r\n"); /* Get length of the first token. */
	
	        if (!strncmp(string, "add", len)) {
	            command = OFPFC_ADD;
	        } else if (!strncmp(string, "delete", len)) {
	            command = OFPFC_DELETE;
	        } else if (!strncmp(string, "delete_strict", len)) {
	            command = OFPFC_DELETE_STRICT;
	        } else if (!strncmp(string, "modify", len)) {
	            command = OFPFC_MODIFY;
	        } else if (!strncmp(string, "modify_strict", len)) {
	            command = OFPFC_MODIFY_STRICT;
	        } else {
	            len = 0;
	            command = OFPFC_ADD;
	        }
	        string += len;
	    }

		// 根据流表的操作，判断需要处理哪些字段
		switch (command) {
	    case -1:
	        fields = F_OUT_PORT;
	        break;
	
	    case OFPFC_ADD:
	        fields = F_ACTIONS | F_TIMEOUT | F_PRIORITY | F_FLAGS | F_IMPORTANCE;
	        break;
	
	    case OFPFC_DELETE:
	        fields = F_OUT_PORT;
	        break;
	
	    case OFPFC_DELETE_STRICT:
	        fields = F_OUT_PORT | F_PRIORITY;
	        break;
	
	    case OFPFC_MODIFY:
	        fields = F_ACTIONS | F_TIMEOUT | F_PRIORITY | F_FLAGS;
	        break;
	
	    case OFPFC_MODIFY_STRICT:
	        fields = F_ACTIONS | F_TIMEOUT | F_PRIORITY | F_FLAGS;
	        break;
	
	    default:
	        OVS_NOT_REACHED();
	    }

		// 初始化输出的结构 ofputil_flow_mod
		 *fm = (struct ofputil_flow_mod) {
	        .priority = OFP_DEFAULT_PRIORITY,
	        .table_id = 0xff,
	        .command = command,
	        .buffer_id = UINT32_MAX,
	        .out_port = OFPP_ANY,
	        .out_group = OFPG_ANY,
	    };
	    /* For modify, by default, don't update the cookie. */
	    if (command == OFPFC_MODIFY || command == OFPFC_MODIFY_STRICT) {
	        fm->new_cookie = OVS_BE64_MAX;
	    }

		// 解析 action 字段
		if (fields & F_ACTIONS) {
	        act_str = ofp_extract_actions(string);
	        if (!act_str) {
	            return xstrdup("must specify an action");
	        }
	    }
		// match 保存流表解析的结果
		struct match match = MATCH_CATCHALL_INITIALIZER;
		// 循环调用 ofputil_parse_key_value() 解析流表中指定的 key=value 字段，例如 "idle_timeout=0,dl_type=0x0800,nw_proto=1"
		// 每次解析到的 key 保存在 name，value 保存在 value
	    while (ofputil_parse_key_value(&string, &name, &value)) {
	        const struct ofp_protocol *p;
	        const struct mf_field *mf;
	        char *error = NULL;
			// 解析协议，如 tcp，udp
	        if (ofp_parse_protocol(name, &p)) {
	            match_set_dl_type(&match, htons(p->dl_type));
	            if (p->nw_proto) {
	                match_set_nw_proto(&match, p->nw_proto);
	            }
	            match_set_default_packet_type(&match);
	        } else if (!strcmp(name, "eth")) {
	            match_set_packet_type(&match, htonl(PT_ETH));
	        } else if (fields & F_FLAGS && !strcmp(name, "send_flow_rem")) {
	            fm->flags |= OFPUTIL_FF_SEND_FLOW_REM;
	        } else if (fields & F_FLAGS && !strcmp(name, "check_overlap")) {
	            fm->flags |= OFPUTIL_FF_CHECK_OVERLAP;
	        } else if (fields & F_FLAGS && !strcmp(name, "reset_counts")) {
	            fm->flags |= OFPUTIL_FF_RESET_COUNTS;
	            *usable_protocols &= OFPUTIL_P_OF12_UP;
	        } else if (fields & F_FLAGS && !strcmp(name, "no_packet_counts")) {
	            fm->flags |= OFPUTIL_FF_NO_PKT_COUNTS;
	            *usable_protocols &= OFPUTIL_P_OF13_UP;
	        } else if (fields & F_FLAGS && !strcmp(name, "no_byte_counts")) {
	            fm->flags |= OFPUTIL_FF_NO_BYT_COUNTS;
	            *usable_protocols &= OFPUTIL_P_OF13_UP;
	        } else if (!strcmp(name, "no_readonly_table")
	                   || !strcmp(name, "allow_hidden_fields")) {
	             /* ignore these fields. */
	        } else if ((mf = mf_from_name(name)) != NULL) {
	            error = ofp_parse_field(mf, value, port_map,
	                                    &match, usable_protocols);
	        } else if (strchr(name, '[')) {
	            error = parse_subfield(name, value, &match, usable_protocols);
			} else {
	            if (!*value) {
	                return xasprintf("field %s missing value", name);
	            }
				// 解析 table id
	            if (!strcmp(name, "table")) {
	                if (!ofputil_table_from_string(value, table_map,
	                                               &fm->table_id)) {
	                    return xasprintf("unknown table \"%s\"", value);
	                }
	                if (fm->table_id != 0xff) {
	                    *usable_protocols &= OFPUTIL_P_TID;
	                }
	            } else if (fields & F_OUT_PORT && !strcmp(name, "out_port")) {
	                if (!ofputil_port_from_string(value, port_map,
	                                              &fm->out_port)) {
	                    error = xasprintf("%s is not a valid OpenFlow port",
	                                      value);
	                }
	            } else if (fields & F_OUT_PORT && !strcmp(name, "out_group")) {
	                *usable_protocols &= OFPUTIL_P_OF11_UP;
	                if (!ofputil_group_from_string(value, &fm->out_group)) {
	                    error = xasprintf("%s is not a valid OpenFlow group",
	                                      value);
	                }
	            } else if (fields & F_PRIORITY && !strcmp(name, "priority")) {
	                uint16_t priority = 0;
	
	                error = str_to_u16(value, name, &priority);
	                fm->priority = priority;
	            } else if (fields & F_TIMEOUT && !strcmp(name, "idle_timeout")) {
	                error = str_to_u16(value, name, &fm->idle_timeout);
	            } else if (fields & F_TIMEOUT && !strcmp(name, "hard_timeout")) {
	                error = str_to_u16(value, name, &fm->hard_timeout);
	            } else if (fields & F_IMPORTANCE && !strcmp(name, "importance")) {
	                error = str_to_u16(value, name, &fm->importance);
	            } else if (!strcmp(name, "cookie")) {
	                char *mask = strchr(value, '/');
					if (mask) {
	                    /* A mask means we're searching for a cookie. */
	                    if (command == OFPFC_ADD) {
	                        return xstrdup("flow additions cannot use "
	                                       "a cookie mask");
	                    }
	                    *mask = '\0';
	                    error = str_to_be64(value, &fm->cookie);
	                    if (error) {
	                        return error;
	                    }
	                    error = str_to_be64(mask + 1, &fm->cookie_mask);
	
	                    /* Matching of the cookie is only supported through NXM or
	                     * OF1.1+. */
	                    if (fm->cookie_mask != htonll(0)) {
	                        *usable_protocols &= OFPUTIL_P_NXM_OF11_UP;
	                    }
	                } else {
	                    /* No mask means that the cookie is being set. */
	                    if (command != OFPFC_ADD && command != OFPFC_MODIFY
	                        && command != OFPFC_MODIFY_STRICT) {
	                        return xasprintf("cannot set cookie (to match on a "
	                                         "cookie, specify a mask, e.g. "
	                                         "cookie=%s/-1)", value);
	                    }
	                    error = str_to_be64(value, &fm->new_cookie);
	                    fm->modify_cookie = true;
	                }
	            } else if (!strcmp(name, "duration")
	                       || !strcmp(name, "n_packets")
	                       || !strcmp(name, "n_bytes")
	                       || !strcmp(name, "idle_age")
	                       || !strcmp(name, "hard_age")) {
	                /* Ignore these, so that users can feed the output of
	                 * "ovs-ofctl dump-flows" back into commands that parse
	                 * flows. */
	            } else {
	                error = xasprintf("unknown keyword %s", name);
	            }
	        }
	
	        if (error) {
	            return error;
	        }
	    }

		/* Copy ethertype to flow->dl_type for matches on packet_type
	     * (OFPHTN_ETHERTYPE, ethertype). */
	    if (match.wc.masks.packet_type == OVS_BE32_MAX &&
	            pt_ns(match.flow.packet_type) == OFPHTN_ETHERTYPE) {
	        match.flow.dl_type = pt_ns_type_be(match.flow.packet_type);
	    }
	    /* Check for usable protocol interdependencies between match fields. */
	    if (match.flow.dl_type == htons(ETH_TYPE_IPV6)) {
	        const struct flow_wildcards *wc = &match.wc;
	        /* Only NXM and OXM support matching L3 and L4 fields within IPv6.
	         *
	         * (IPv6 specific fields as well as arp_sha, arp_tha, nw_frag, and
	         *  nw_ttl are covered elsewhere so they don't need to be included in
	         *  this test too.)
	         */
	        if (wc->masks.nw_proto || wc->masks.nw_tos
	            || wc->masks.tp_src || wc->masks.tp_dst) {
	            *usable_protocols &= OFPUTIL_P_NXM_OXM_ANY;
	        }
	    }
	    if (!fm->cookie_mask && fm->new_cookie == OVS_BE64_MAX
	        && (command == OFPFC_MODIFY || command == OFPFC_MODIFY_STRICT)) {
	        /* On modifies without a mask, we are supposed to add a flow if
	         * one does not exist.  If a cookie wasn't been specified, use a
	         * default of zero. */
	        fm->new_cookie = htonll(0);
	    }

		if (fields & F_ACTIONS) {
	        enum ofputil_protocol action_usable_protocols;
	        struct ofpbuf ofpacts;
	        char *error;
	
	        ofpbuf_init(&ofpacts, 32);
	        struct ofpact_parse_params pp = {
	            .port_map = port_map,
	            .table_map = table_map,
	            .ofpacts = &ofpacts,
	            .usable_protocols = &action_usable_protocols
	        };
	        error = ofpacts_parse_instructions(act_str, &pp);
	        *usable_protocols &= action_usable_protocols;
	        if (!error) {
	            enum ofperr err;
	
	            struct ofpact_check_params cp = {
	                .match = &match,
	                .max_ports = OFPP_MAX,
	                .table_id = fm->table_id,
	                .n_tables = 255,
	            };
	            err = ofpacts_check(ofpacts.data, ofpacts.size, &cp);
	            *usable_protocols &= cp.usable_protocols;
	            if (!err && !*usable_protocols) {
	                err = OFPERR_OFPBAC_MATCH_INCONSISTENT;
	            }
	            if (err) {
	                error = xasprintf("actions are invalid with specified match "
	                                  "(%s)", ofperr_to_string(err));
	            }
	
	        }
	        if (error) {
	            ofpbuf_uninit(&ofpacts);
	            return error;
	        }
	
	        fm->ofpacts_len = ofpacts.size;
	        fm->ofpacts = ofpbuf_steal_data(&ofpacts);
	    } else {
	        fm->ofpacts_len = 0;
	        fm->ofpacts = NULL;
	    }
	    minimatch_init(&fm->match, &match);
	
	    return NULL;
	}




		
		