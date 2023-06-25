
    (lib/dpif-netdev.c)
    /* Must be called each time a port is added/removed or the cmask changes.
    * This creates and destroys pmd threads, reconfigures ports, opens their
    * rxqs and assigns all rxqs/txqs to pmd threads. */
    // 参数中，dp 为待配置的 datapath
    static void
    reconfigure_datapath(struct dp_netdev *dp)
        OVS_REQ_RDLOCK(dp->port_rwlock)
    {
        struct hmapx busy_threads = HMAPX_INITIALIZER(&busy_threads);
        struct dp_netdev_pmd_thread *pmd;
        struct dp_netdev_port *port;
        int wanted_txqs;

        dp->last_reconfigure_seq = seq_read(dp->reconfigure_seq);

        /* Step 1: Adjust the pmd threads based on the datapath ports, the cores
        * on the system and the user configuration. */
        // 第一步，根据 datapath 端口调整 pmd 线程数目和配置
        reconfigure_pmd_threads(dp);

        wanted_txqs = cmap_count(&dp->poll_threads);

        /* The number of pmd threads might have changed, or a port can be new:
        * adjust the txqs. */
        // pmd 线程的数目可能发生了变化，调整 datapath 上 port 的发送队列数目为 pmd 线程的数目
        HMAP_FOR_EACH (port, node, &dp->ports) {
            netdev_set_tx_multiq(port->netdev, wanted_txqs);
        }

        /* Step 2: Remove from the pmd threads ports that have been removed or
        * need reconfiguration. */

        /* Check for all the ports that need reconfiguration.  We cache this in
        * 'port->need_reconfigure', because netdev_is_reconf_required() can
        * change at any time.
        * Also mark for reconfiguration all ports which will likely change their
        * 'txq_mode' parameter.  It's required to stop using them before
        * changing this setting and it's simpler to mark ports here and allow
        * 'pmd_remove_stale_ports' to remove them from threads.  There will be
        * no actual reconfiguration in 'port_reconfigure' because it's
        * unnecessary.  */
        // 获取需要重新配置的 port
        HMAP_FOR_EACH (port, node, &dp->ports) {
            if (netdev_is_reconf_required(port->netdev)
                || ((port->txq_mode == TXQ_MODE_XPS)
                    != (netdev_n_txq(port->netdev) < wanted_txqs))
                || ((port->txq_mode == TXQ_MODE_XPS_HASH)
                    != (port->txq_requested_mode == TXQ_REQ_MODE_HASH
                        && netdev_n_txq(port->netdev) > 1))) {
                port->need_reconfigure = true;
            }
        }

        /* Remove from the pmd threads all the ports that have been deleted or
        * need reconfiguration. */
        // 如果 port 需要重新配置，先把它从 pmd 线程中移除
        CMAP_FOR_EACH (pmd, node, &dp->poll_threads) {
            pmd_remove_stale_ports(dp, pmd);
        }

        /* Reload affected pmd threads.  We must wait for the pmd threads before
        * reconfiguring the ports, because a port cannot be reconfigured while
        * it's being used. */
        // reload 受影响的 pmd 线程，使得前面调整 port 的操作生效
        reload_affected_pmds(dp);

        /* Step 3: Reconfigure ports. */
        // 第三步，重新配置端口
        /* We only reconfigure the ports that we determined above, because they're
        * not being used by any pmd thread at the moment.  If a port fails to
        * reconfigure we remove it from the datapath. */
        HMAP_FOR_EACH_SAFE (port, node, &dp->ports) {
            int err;

            if (!port->need_reconfigure) {
                continue;
            }

            // 重新配置端口
            err = port_reconfigure(port);
            if (err) {
                hmap_remove(&dp->ports, &port->node);
                seq_change(dp->port_seq);
                port_destroy(port);
            } else {
                /* With a single queue, there is no point in using hash mode. */
                if (port->txq_requested_mode == TXQ_REQ_MODE_HASH &&
                    netdev_n_txq(port->netdev) > 1) {
                    port->txq_mode = TXQ_MODE_XPS_HASH;
                } else if (netdev_n_txq(port->netdev) < wanted_txqs) {
                    port->txq_mode = TXQ_MODE_XPS;
                } else {
                    port->txq_mode = TXQ_MODE_STATIC;
                }
            }
        }

        /* Step 4: Compute new rxq scheduling.  We don't touch the pmd threads
        * for now, we just update the 'pmd' pointer in each rxq to point to the
        * wanted thread according to the scheduling policy. */
        // 第4步，重新在 pmd 线程上调度接收队列

        /* Reset all the pmd threads to non isolated. */
        CMAP_FOR_EACH (pmd, node, &dp->poll_threads) {
            pmd->isolated = false;
        }

        /* Reset all the queues to unassigned */
        HMAP_FOR_EACH (port, node, &dp->ports) {
            for (int i = 0; i < port->n_rxq; i++) {
                port->rxqs[i].pmd = NULL;
            }
        }
        // 重新调度
        rxq_scheduling(dp);

        /* Step 5: Remove queues not compliant with new scheduling. */
        // 第 5 步，把不符合新调度配置的接收队列移除掉

        /* Count all the threads that will have at least one queue to poll. */
        HMAP_FOR_EACH (port, node, &dp->ports) {
            for (int qid = 0; qid < port->n_rxq; qid++) {
                struct dp_netdev_rxq *q = &port->rxqs[qid];

                if (q->pmd) {
                    hmapx_add(&busy_threads, q->pmd);
                }
            }
        }

        CMAP_FOR_EACH (pmd, node, &dp->poll_threads) {
            struct rxq_poll *poll;

            ovs_mutex_lock(&pmd->port_mutex);
            HMAP_FOR_EACH_SAFE (poll, node, &pmd->poll_list) {
                if (poll->rxq->pmd != pmd) {
                    dp_netdev_del_rxq_from_pmd(pmd, poll);

                    /* This pmd might sleep after this step if it has no rxq
                    * remaining. Tell it to busy wait for new assignment if it
                    * has at least one scheduled queue. */
                    if (hmap_count(&pmd->poll_list) == 0 &&
                        hmapx_contains(&busy_threads, pmd)) {
                        atomic_store_relaxed(&pmd->wait_for_reload, true);
                    }
                }
            }
            ovs_mutex_unlock(&pmd->port_mutex);
        }

        hmapx_destroy(&busy_threads);

        /* Reload affected pmd threads.  We must wait for the pmd threads to remove
        * the old queues before readding them, otherwise a queue can be polled by
        * two threads at the same time. */
        // reload pmd 线程，使得前面对接收队列的调整生效
        reload_affected_pmds(dp);

        /* Step 6: Add queues from scheduling, if they're not there already. */
        // 把接收队列调度结果添加到 pmd 线程上
        HMAP_FOR_EACH (port, node, &dp->ports) {
            if (!netdev_is_pmd(port->netdev)) {
                continue;
            }

            for (int qid = 0; qid < port->n_rxq; qid++) {
                struct dp_netdev_rxq *q = &port->rxqs[qid];

                if (q->pmd) {
                    ovs_mutex_lock(&q->pmd->port_mutex);
                    dp_netdev_add_rxq_to_pmd(q->pmd, q);
                    ovs_mutex_unlock(&q->pmd->port_mutex);
                }
            }
        }

        /* Add every port and bond to the tx port and bond caches of
        * every pmd thread, if it's not there already and if this pmd
        * has at least one rxq to poll.
        */
        CMAP_FOR_EACH (pmd, node, &dp->poll_threads) {
            ovs_mutex_lock(&pmd->port_mutex);
            if (hmap_count(&pmd->poll_list) || pmd->core_id == NON_PMD_CORE_ID) {
                struct tx_bond *bond;

                HMAP_FOR_EACH (port, node, &dp->ports) {
                    dp_netdev_add_port_tx_to_pmd(pmd, port);
                }

                CMAP_FOR_EACH (bond, node, &dp->tx_bonds) {
                    dp_netdev_add_bond_tx_to_pmd(pmd, bond, false);
                }
            }
            ovs_mutex_unlock(&pmd->port_mutex);
        }

        /* Reload affected pmd threads. */
        reload_affected_pmds(dp);

        /* PMD ALB will need to recheck if dry run needed. */
        dp->pmd_alb.recheck_config = true;
    }

