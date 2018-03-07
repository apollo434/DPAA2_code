Simply begin:
-----
```
[pek-lpd-ccm2:fsl-dpaa2]$find . | grep -r "module_init" *
ethernet/dpaa2-eth.c:module_init(dpaa2_eth_driver_init);

[pek-lpd-ccm2:fsl-dpaa2]$find . | grep -r "dpaa2_eth_driver" *
ethernet/dpaa2-eth.c:static struct fsl_mc_driver dpaa2_eth_driver = {

[pek-lpd-ccm2:fsl-dpaa2]$find . | grep -r "net_device_ops" *
ethernet/dpaa2-eth.c:static const struct net_device_ops dpaa2_eth_ops = {
ethsw/switch.c:static const struct net_device_ops ethsw_ops = {
ethsw/switch.c:static const struct net_device_ops ethsw_port_ops = {
evb/evb.c:static const struct net_device_ops evb_port_ops = {
evb/evb.c:static const struct net_device_ops evb_ops = {
mac/mac.c:static const struct net_device_ops dpaa2_mac_ndo_ops = {
[pek-lpd-ccm2:fsl-dpaa2]$
```

The key structure:
-----

```
struct net_device_ops{}
struct fsl_mc_driver{}


(ethernet/dpaa2-eth.c)
static struct fsl_mc_driver dpaa2_eth_driver = {
        .driver = {
                .name = KBUILD_MODNAME,
                .owner = THIS_MODULE,
        },
        .probe = dpaa2_eth_probe,
        .remove = dpaa2_eth_remove,
        .match_id_table = dpaa2_eth_match_id_table
};

(ethernet/dpaa2-eth.c)
static const struct net_device_ops dpaa2_eth_ops = {
        .ndo_open = dpaa2_eth_open,
        .ndo_start_xmit = dpaa2_eth_tx,
        .ndo_stop = dpaa2_eth_stop,
        .ndo_init = dpaa2_eth_init,
        .ndo_set_mac_address = dpaa2_eth_set_addr,
        .ndo_get_stats64 = dpaa2_eth_get_stats,
        .ndo_change_mtu = dpaa2_eth_change_mtu,
        .ndo_set_rx_mode = dpaa2_eth_set_rx_mode,
        .ndo_set_features = dpaa2_eth_set_features,
        .ndo_do_ioctl = dpaa2_eth_ioctl,
};

(ethsw/switch.c)
static const struct net_device_ops ethsw_ops = {
        .ndo_open               = &ethsw_open,
        .ndo_stop               = &ethsw_stop,

        .ndo_bridge_setlink     = &ethsw_setlink,
        .ndo_bridge_getlink     = &ethsw_getlink,
        .ndo_bridge_dellink     = &ethsw_dellink,

        .ndo_start_xmit         = &ethsw_dropframe,
};

static const struct net_device_ops ethsw_port_ops = {
        .ndo_open               = &ethsw_port_open,
        .ndo_stop               = &ethsw_port_stop,

        .ndo_fdb_add            = &ethsw_port_fdb_add,
        .ndo_fdb_del            = &ethsw_port_fdb_del,
        .ndo_fdb_dump           = &ndo_dflt_fdb_dump,

        .ndo_get_stats64        = &ethsw_port_get_stats,
        .ndo_change_mtu         = &ethsw_port_change_mtu,

        .ndo_start_xmit         = &ethsw_dropframe,
};

(evb/evb.c)
static const struct net_device_ops evb_port_ops = {
        .ndo_open               = &evb_port_open,

        .ndo_start_xmit         = &evb_dropframe,

        .ndo_fdb_add            = &evb_port_fdb_add,
        .ndo_fdb_del            = &evb_port_fdb_del,

        .ndo_get_stats64        = &evb_port_get_stats,
        .ndo_change_mtu         = &evb_change_mtu,
};

static const struct net_device_ops evb_ops = {
        .ndo_start_xmit         = &evb_dropframe,
        .ndo_open               = &evb_open,
        .ndo_stop               = &evb_close,

        .ndo_bridge_setlink     = &evb_setlink,
        .ndo_bridge_getlink     = &evb_getlink,
        .ndo_bridge_dellink     = &evb_dellink,

        .ndo_get_stats64        = &evb_port_get_stats,
        .ndo_change_mtu         = &evb_change_mtu,
};


(mac/mac.c)
static const struct net_device_ops dpaa2_mac_ndo_ops = {
        .ndo_start_xmit         = &dpaa2_mac_drop_frame,
        .ndo_open               = &dpaa2_mac_open,
        .ndo_stop               = &dpaa2_mac_stop,
        .ndo_get_stats64        = &dpaa2_mac_get_stats,
};

static const struct ethtool_ops dpaa2_mac_ethtool_ops = {
        .get_settings           = &dpaa2_mac_get_settings,
        .set_settings           = &dpaa2_mac_set_settings,
        .get_strings            = &dpaa2_mac_get_strings,
        .get_ethtool_stats      = &dpaa2_mac_get_ethtool_stats,
        .get_sset_count         = &dpaa2_mac_get_sset_count,
};


(ethernet/dpaa2-eth.h)
/* Driver private data */
struct dpaa2_eth_priv {
        struct net_device *net_dev;

        /* Standard statistics */
        struct rtnl_link_stats64 __percpu *percpu_stats;
        /* Extra stats, in addition to the ones known by the kernel */
        struct dpaa2_eth_drv_stats __percpu *percpu_extras;
        struct iommu_domain *iommu_domain;

        bool ts_tx_en; /* Tx timestamping enabled */
        bool ts_rx_en; /* Rx timestamping enabled */

        u16 tx_data_offset;
        u16 rx_buf_align;

        u16 bpid;
        u16 tx_qdid;

        int tx_pause_frames;
        int num_bufs;
        int refill_thresh;

        /* Tx congestion notifications are written here */
        void *cscn_mem;
        void *cscn_unaligned;
        dma_addr_t cscn_dma;

        u8 num_fqs;
        /* Tx queues are at the beginning of the array */
        struct dpaa2_eth_fq fq[DPAA2_ETH_MAX_QUEUES];

        u8 num_channels;
        struct dpaa2_eth_channel *channel[DPAA2_ETH_MAX_DPCONS];

        int dpni_id;
        struct dpni_attr dpni_attrs;
        struct fsl_mc_device *dpbp_dev;

        struct fsl_mc_io *mc_io;
        /* SysFS-controlled affinity mask for TxConf FQs */
        struct cpumask txconf_cpumask;
        /* Cores which have an affine DPIO/DPCON.
         * This is the cpu set on which Rx frames are processed;
         * Tx confirmation frames are processed on a subset of this,
         * depending on user settings.
         */
        struct cpumask dpio_cpumask;

        u16 mc_token;

        struct dpni_link_state link_state;
        struct fsl_mc_device *dpbp_dev;

        struct fsl_mc_io *mc_io;
        /* SysFS-controlled affinity mask for TxConf FQs */
        struct cpumask txconf_cpumask;
        /* Cores which have an affine DPIO/DPCON.
         * This is the cpu set on which Rx frames are processed;
         * Tx confirmation frames are processed on a subset of this,
         * depending on user settings.
         */
        struct cpumask dpio_cpumask;

        u16 mc_token;

        struct dpni_link_state link_state;
        bool do_link_poll;
        struct task_struct *poll_thread;

        struct dpaa2_eth_hash_fields *hash_fields;
        u8 num_hash_fields;
        /* enabled ethtool hashing bits */
        u64 rx_flow_hash;

#ifdef CONFIG_FSL_DPAA2_ETH_DEBUGFS
        struct dpaa2_debugfs dbg;
#endif

        /* array of classification rules */
        struct dpaa2_eth_cls_rule *cls_rule;

        struct dpni_tx_shaping_cfg shaping_cfg;
};

```

DPAA2 Init:
-----

```
(ethernet/dpaa2-eth.c)
dpaa2_eth_driver_init
{
  fsl_mc_driver_register(&dpaa2_eth_driver)
>>>>>>>>>
  (fsl-mc/include/mc.h)
  #define fsl_mc_driver_register(drv) \
    __fsl_mc_driver_register(drv, THIS_MODULE)
>>>>>>>>>>>>>>>>>
    /**
 * __fsl_mc_driver_register - registers a child device driver with the
 * MC bus
 *
 * This function is implicitly invoked from the registration function of
 * fsl_mc device drivers, which is generated by the
 * module_fsl_mc_driver() macro.
 */
int __fsl_mc_driver_register(struct fsl_mc_driver *mc_driver,
                             struct module *owner)
{
        int error;

        mc_driver->driver.owner = owner;
        mc_driver->driver.bus = &fsl_mc_bus_type;

        if (mc_driver->probe)
                mc_driver->driver.probe = fsl_mc_driver_probe;

        if (mc_driver->remove)
                mc_driver->driver.remove = fsl_mc_driver_remove;

        if (mc_driver->shutdown)
                mc_driver->driver.shutdown = fsl_mc_driver_shutdown;

        error = driver_register(&mc_driver->driver);
        if (error < 0) {
                pr_err("driver_register() failed for %s: %d\n",
                       mc_driver->driver.name, error);
                return error;
        }

        return 0;
}
EXPORT_SYMBOL_GPL(__fsl_mc_driver_register);
>>>>>>>>>>>>>>>>>
>>>>>>>>>  
}


(ethernet/dpaa2-eth.c)
static int dpaa2_eth_probe(struct fsl_mc_device *dpni_dev)
{
        struct device *dev;
        struct net_device *net_dev = NULL;
        struct dpaa2_eth_priv *priv = NULL;

        /* Net device */
        net_dev = alloc_etherdev_mq(sizeof(*priv), DPAA2_ETH_MAX_TX_QUEUES);

        /* Obtain a MC portal */
        err = fsl_mc_portal_allocate(dpni_dev, FSL_MC_IO_ATOMIC_CONTEXT_PORTAL,
                                     &priv->mc_io);

        /* MC objects initialization and configuration */
        err = setup_dpni(dpni_dev);

        err = setup_dpio(priv);

        setup_fqs(priv);

        err = setup_dpbp(priv);

        err = bind_dpni(priv);

        /* Add a NAPI context for each channel */
        add_ch_napi(priv);
        enable_ch_napi(priv);

        /* Percpu statistics */
        priv->percpu_stats = alloc_percpu(*priv->percpu_stats);

        priv->percpu_extras = alloc_percpu(*priv->percpu_extras);

        /* Configure checksum offload based on current interface flags */
        err = set_rx_csum(priv, !!(net_dev->features & NETIF_F_RXCSUM));

        err = set_tx_csum(priv, !!(net_dev->features &
                                   (NETIF_F_IP_CSUM | NETIF_F_IPV6_CSUM)));

        err = alloc_rings(priv);

        net_dev->ethtool_ops = &dpaa2_ethtool_ops;

        err = setup_irqs(dpni_dev);

        dpaa2_eth_sysfs_init(&net_dev->dev);

#ifdef CONFIG_FSL_DPAA2_ETH_DEBUGFS
        dpaa2_dbg_add(priv);
#endif

        dev_info(dev, "Probed interface %s\n", net_dev->name);
        return 0;
}
```

****
Regarding dpio-driver in mc, please check the **dpio-driver.txt** and **MC_README.txt**
****

Restool how to be sent to mc
-----

```
(drivers/staging/fsl-mc/mc-restool.c)
static const struct file_operations fsl_mc_restool_dev_fops = {
        .owner = THIS_MODULE,
        .open = fsl_mc_restool_dev_open,
        .release = fsl_mc_restool_dev_release,
        .unlocked_ioctl = fsl_mc_restool_dev_ioctl,
};

fsl_mc_restool_dev_ioctl
  restool_send_mc_command
    mc_send_command (mc-sys.c)
      mc_write_command
        __raw_writeq
>>>>
(arch/arm64/include/asm/io.h)       
#define __raw_writeq __raw_writeq
static inline void __raw_writeq(u64 val, volatile void __iomem *addr)
{
        asm volatile("str %x0, [%1]" : : "rZ" (val), "r" (addr));
}
>>>>

```
