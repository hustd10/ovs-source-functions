
    （lib/dpdk.c）
    void
    dpdk_init(const struct smap *ovs_other_config)
    {
        static bool enabled = false;

        if (enabled || !ovs_other_config) {
            return;
        }

        const char *dpdk_init_val = smap_get_def(ovs_other_config, "dpdk-init",
                                                "false");

        bool try_only = !strcasecmp(dpdk_init_val, "try");
        // 检查 other_config 中 dpdk-init 的配置
        if (!strcasecmp(dpdk_init_val, "true") || try_only) {
            static struct ovsthread_once once_enable = OVSTHREAD_ONCE_INITIALIZER;

            if (ovsthread_once_start(&once_enable)) {
                VLOG_INFO("Using %s", rte_version());
                VLOG_INFO("DPDK Enabled - initializing...");
                // 调用 dpdk_init__() 进行初始化
                enabled = dpdk_init__(ovs_other_config);
                if (enabled) {
                    VLOG_INFO("DPDK Enabled - initialized");
                } else if (!try_only) {
                    ovs_abort(rte_errno, "Cannot init EAL");
                }
                ovsthread_once_done(&once_enable);
            } else {
                VLOG_ERR_ONCE("DPDK Initialization Failed.");
            }
        } else {
            VLOG_INFO_ONCE("DPDK Disabled - Use other_config:dpdk-init to enable");
        }
        atomic_store_relaxed(&dpdk_initialized, enabled);
    }

