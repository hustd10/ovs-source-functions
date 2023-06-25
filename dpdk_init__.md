dpdk_init__()：

    （lib/dpdk.c）
    static bool
    dpdk_init__(const struct smap *ovs_other_config)
    {
        char **argv = NULL;
        int result;
        bool auto_determine = true;
        struct ovs_numa_dump *affinity = NULL;
        struct svec args = SVEC_EMPTY_INITIALIZER;
        // 设置日志配置
        log_stream = fopencookie(NULL, "w+", dpdk_log_func);
        if (log_stream == NULL) {
            VLOG_ERR("Can't redirect DPDK log: %s.", ovs_strerror(errno));
        } else {
            setbuf(log_stream, NULL);
            rte_openlog_stream(log_stream);
        }

        svec_add(&args, ovs_get_program_name());
        // 解析 other_config 中的相关配置，解析结果保存到 args
        construct_dpdk_args(ovs_other_config, &args);

    #ifdef DPDK_IN_MEMORY_SUPPORTED
        if (!args_contains(&args, "--in-memory") &&
                !args_contains(&args, "--legacy-mem")) {
            svec_add(&args, "--in-memory");
        }
    #endif

    // 如果指定了 dpdk-lcore-mask 即线程的亲和性，则不自动决定，否则自动决定
    if (args_contains(&args, "-c") || args_contains(&args, "-l")) {
        auto_determine = false;
    }

    /**
     * NOTE: This is an unsophisticated mechanism for determining the DPDK
     * main core.
     */
    if (auto_determine) {
        const struct ovs_numa_info_core *core;
        int cpu = 0;

        /* Get the main thread affinity */
        affinity = ovs_numa_thread_getaffinity_dump();
        if (affinity) {
            cpu = INT_MAX;
            FOR_EACH_CORE_ON_DUMP (core, affinity) {
                if (cpu > core->core_id) {
                    cpu = core->core_id;
                }
            }
        } else {
            /* User did not set dpdk-lcore-mask and unable to get current
             * thread affintity - default to core #0 */
            VLOG_ERR("Thread getaffinity failed. Using core #0");
        }
        svec_add(&args, "-l");
        svec_add_nocopy(&args, xasprintf("%d", cpu));
    }

    svec_terminate(&args);

    optind = 1;

    if (VLOG_IS_INFO_ENABLED()) {
        struct ds eal_args = DS_EMPTY_INITIALIZER;
        char *joined_args = svec_join(&args, " ", ".");

        ds_put_format(&eal_args, "EAL ARGS: %s", joined_args);
        VLOG_INFO("%s", ds_cstr_ro(&eal_args));
        ds_destroy(&eal_args);
        free(joined_args);
    }

    /* Copy because 'rte_eal_init' will change the argv, i.e. it will remove
     * some arguments from it. '+1' to copy the terminating NULL.  */
    argv = xmemdup(args.names, (args.n + 1) * sizeof args.names[0]);

    /* Make sure things are initialized ... */
    result = rte_eal_init(args.n, argv);

    free(argv);
    svec_destroy(&args);

    /* Set the main thread affinity back to pre rte_eal_init() value */
    if (affinity) {
        ovs_numa_thread_setaffinity_dump(affinity);
        ovs_numa_dump_destroy(affinity);
    }

    if (result < 0) {
        VLOG_EMER("Unable to initialize DPDK: %s", ovs_strerror(rte_errno));
        return false;
    }

    if (!rte_mp_disable()) {
        VLOG_EMER("Could not disable multiprocess, DPDK won't be available.");
        rte_eal_cleanup();
        return false;
    }

    if (VLOG_IS_DBG_ENABLED()) {
        size_t size;
        char *response = NULL;
        FILE *stream = open_memstream(&response, &size);

        if (stream) {
            fprintf(stream, "rte_memzone_dump:\n");
            rte_memzone_dump(stream);
            fprintf(stream, "rte_log_dump:\n");
            rte_log_dump(stream);
            fclose(stream);
            VLOG_DBG("%s", response);
            free(response);
        } else {
            VLOG_DBG("Could not dump memzone and log levels. "
                     "Unable to open memstream: %s.", ovs_strerror(errno));
        }
    }

    unixctl_command_register("dpdk/lcore-list", "", 0, 0,
                             dpdk_unixctl_mem_stream, rte_lcore_dump);
    unixctl_command_register("dpdk/log-list", "", 0, 0,
                             dpdk_unixctl_mem_stream, rte_log_dump);
    unixctl_command_register("dpdk/log-set", "{level | pattern:level}", 0,
                             INT_MAX, dpdk_unixctl_log_set, NULL);
    unixctl_command_register("dpdk/get-malloc-stats", "", 0, 0,
                             dpdk_unixctl_mem_stream,
                             malloc_dump_stats_wrapper);

    /* We are called from the main thread here */
    RTE_PER_LCORE(_lcore_id) = NON_PMD_CORE_ID;

    /* Finally, register the dpdk classes */
    netdev_dpdk_register(ovs_other_config);
    netdev_register_flow_api_provider(&netdev_offload_dpdk);
    return true;
}



construct_dpdk_args()：

    （lib/dpdk.c）
    // 该函数用于解析 ovs-dpdk 相关的配置
    // 参数中，ovs_other_config 为输入，即配置的来源
    // args 为输出，保存解析后的结果
    static void
    construct_dpdk_args(const struct smap *ovs_other_config, struct svec *args)
    {
        // 获取 dpdk-extra 字段的配置，下面是一个示例：
        // "-a 0000:06:00.0,representor=[65535],vf_nums=0,debug_level=0"
        const char *extra_configuration = smap_get(ovs_other_config, "dpdk-extra");

        // 解析 extra 配置
        if (extra_configuration) {
            svec_parse_words(args, extra_configuration);
        }

        construct_dpdk_options(ovs_other_config, args);
        construct_dpdk_mutex_options(ovs_other_config, args);
    }


construct_dpdk_options():

    (lib/dpdk.c)
    static void
    construct_dpdk_options(const struct smap *ovs_other_config, struct svec *args)
    {
        // 解析 dpdk-lcore-mask、dpdk-hugepage-dir、dpdk-socket-limit 
        struct dpdk_options_map {
            const char *ovs_configuration;
            const char *dpdk_option;
            bool default_enabled;
            const char *default_value;
        } opts[] = {
            {"dpdk-lcore-mask",   "-c",             false, NULL},
            {"dpdk-hugepage-dir", "--huge-dir",     false, NULL},
            {"dpdk-socket-limit", "--socket-limit", false, NULL},
        };

        int i;

        /*First, construct from the flat-options (non-mutex)*/
        for (i = 0; i < ARRAY_SIZE(opts); ++i) {
            const char *value = smap_get(ovs_other_config,
                                        opts[i].ovs_configuration);
            if (!value && opts[i].default_enabled) {
                value = opts[i].default_value;
            }

            if (value) {
                if (!args_contains(args, opts[i].dpdk_option)) {
                    svec_add(args, opts[i].dpdk_option);
                    svec_add(args, value);
                } else {
                    VLOG_WARN("Ignoring database defined option '%s' due to "
                            "dpdk-extra config", opts[i].dpdk_option);
                }
            }
        }
    }   


construct_dpdk_mutex_options():

    (lib/dpdk.c)
    // 参数 args 用于输出，保存解析结果
    construct_dpdk_mutex_options(const struct smap *ovs_other_config,
                             struct svec *args)
    {
        struct dpdk_exclusive_options_map {
            const char *category;
            const char *ovs_dpdk_options[MAX_DPDK_EXCL_OPTS];
            const char *eal_dpdk_options[MAX_DPDK_EXCL_OPTS];
            const char *default_value;
            int default_option;
        } excl_opts[] = {
            {"memory type",
            {"dpdk-alloc-mem", "dpdk-socket-mem", NULL,},
            {"-m",             "--socket-mem",    NULL,},
            NULL, 0
            },
        };

        int i;
        for (i = 0; i < ARRAY_SIZE(excl_opts); ++i) {
            int found_opts = 0, scan, found_pos = -1;
            const char *found_value;
            struct dpdk_exclusive_options_map *popt = &excl_opts[i];

            for (scan = 0; scan < MAX_DPDK_EXCL_OPTS
                    && popt->ovs_dpdk_options[scan]; ++scan) {
                const char *value = smap_get(ovs_other_config,
                                            popt->ovs_dpdk_options[scan]);
                if (value && strlen(value)) {
                    found_opts++;
                    found_pos = scan;
                    found_value = value;
                }
            }

            if (!found_opts) {
                if (popt->default_option) {
                    found_pos = popt->default_option;
                    found_value = popt->default_value;
                } else {
                    continue;
                }
            }

            if (found_opts > 1) {
                VLOG_ERR("Multiple defined options for %s. Please check your"
                        " database settings and reconfigure if necessary.",
                        popt->category);
            }

            if (!args_contains(args, popt->eal_dpdk_options[found_pos])) {
                svec_add(args, popt->eal_dpdk_options[found_pos]);
                svec_add(args, found_value);
            } else {
                VLOG_WARN("Ignoring database defined option '%s' due to "
                        "dpdk-extra config", popt->eal_dpdk_options[found_pos]);
            }
        }
    }
