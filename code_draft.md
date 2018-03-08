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
```
The diagram below shows an overview of the DPAA2 resource management
architecture:

         +--------------------------------------+
         |                  OS                  |
         |                        DPAA2 drivers |
         |                             |        |
         +-----------------------------|--------+
                                       |
                                       | (create,discover,connect
                                       |  config,use,destroy)
                                       |
                         DPAA2         |
         +------------------------| mc portal |-+
         |                             |        |
         |   +- - - - - - - - - - - - -V- - -+  |
         |   |                               |  |
         |   |   Management Complex (MC)     |  |
         |   |                               |  |
         |   +- - - - - - - - - - - - - - - -+  |
         |                                      |
         | Hardware                  Hardware   |   
         | Resources                 Objects    |   
         | ---------                 -------    |   
         | -queues                   -DPRC      |   
         | -buffer pools             -DPMCP     |   
         | -Eth MACs/ports           -DPIO      |   
         | -network interface        -DPNI      |   
         |  profiles                 -DPMAC     |   
         | -queue portals            -DPBP      |   
         | -MC portals                ...       |
         |  ...                                 |
         |                                      |
         +--------------------------------------+

The MC mediates operations such as create, discover,
connect, configuration, and destroy.  Fast-path operations
on data, such as packet transmit/receive, are not mediated by
the MC and are done directly using memory mapped regions in
DPIO objects.


The diagram below shows the objects needed for a simple
network interface configuration on a system with 2 CPUs.

              +---+---+ +---+---+
                 CPU0     CPU1
              +---+---+ +---+---+
                  |         |
              +---+---+ +---+---+
                 DPIO     DPIO
              +---+---+ +---+---+
                    \     /   
                     \   /   
                      \ /
                   +---+---+
                      DPNI  --- DPBP,DPMCP
                   +---+---+
                       |
                       |
                   +---+---+
                     DPMAC
                   +---+---+
                       |
                    port/PHY

    Below the objects are described.  For each object a brief description
    is provided along with a summary of the kinds of operations the object
    supports and a summary of key resources of the object (MMIO regions
    and IRQs).

       -DPMAC (Datapath Ethernet MAC): represents an Ethernet MAC, a
        hardware device that connects to an Ethernet PHY and allows
        physical transmission and reception of Ethernet frames.
           -MMIO regions: none
           -IRQs: DPNI link change
           -commands: set link up/down, link config, get stats,
            IRQ config, enable, reset

       -DPNI (Datapath Network Interface): contains TX/RX queues,
        network interface configuration, and RX buffer pool configuration
        mechanisms.  The TX/RX queues are in memory and are identified by
        queue number.
           -MMIO regions: none
           -IRQs: link state
           -commands: port config, offload config, queue config,
            parse/classify config, IRQ config, enable, reset

       -DPIO (Datapath I/O): provides interfaces to enqueue and dequeue
        packets and do hardware buffer pool management operations.  The DPAA2
        architecture separates the mechanism to access queues (the DPIO object)
        from the queues themselves.  The DPIO provides an MMIO interface to
        enqueue/dequeue packets.  To enqueue something a descriptor is written
        to the DPIO MMIO region, which includes the target queue number.
        There will typically be one DPIO assigned to each CPU.  This allows all
        CPUs to simultaneously perform enqueue/dequeued operations.  DPIOs are
        expected to be shared by different DPAA2 drivers.
           -MMIO regions: queue operations, buffer management
           -IRQs: data availability, congestion notification, buffer
                  pool depletion
           -commands: IRQ config, enable, reset

       -DPBP (Datapath Buffer Pool): represents a hardware buffer
        pool.
           -MMIO regions: none
           -IRQs: none
           -commands: enable, reset

       -DPMCP (Datapath MC Portal): provides an MC command portal.
        Used by drivers to send commands to the MC to manage
        objects.
           -MMIO regions: MC command portal
           -IRQs: command completion
           -commands: IRQ config, enable, reset

```


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

QORIQ-SDK-2.0-IC Key Information
------
*****

The Key Release Files: RCW, DPC and DPL  (LS2088A as an example)

RCW(LS2088A)
It gives flexibility to accommodate a large number of configuration parameters to support a high degree of configuration of the SoC.
Configuration parameters generally include:
1. Frequencies of various blocks including cores/DDR/interconnect.
2. IP pin-muxing configurations.
3. Other SoC configurations

I.E.
PBL_1600_600_1600_1600_0x2a_0x41.bin
1. Core: 1600 MHz
2. Platform: 600 MHz
3. DDR: 1600 MHz


DPC(Data path configuration file)
DPC contains board-specific and system-specific information that overrides the default DPAA hardware configuration.

The file is named dpc-0x2a41.dts. This file specifies the following information:

1. default logging mode for the MC
2. default LS2085ARDB/LS2088ARDB MACs
3. default number of DPAA channels with 2 and 8 work-queues

The DPC is based on a text source file(similar to a device tree source file(DTS)) and compiled with the DTC utility to form a binary structure(blob, similar to DTB). The DPC file should be compiled to a binary blob using standard DTC tool.

The DPC binary(dpc-0x2a41.dtb) is located in the IMAGE iso under dpl-examples/ls2088a/RDB/.


DPL(Data path layout file)
DPL defines the containers created during MC initialization.

A prerequisite for the DPL to define the containers created during MC initialization is that the device tree compiler(dtc) tool is installed on the host system, since the dtc command is used to compile the DPL.

dpl-examples package:
dpl-eth.0x2A_0x41.dts ---- used for simple Ethernet senarios.



******

How to operate in u-boot and relevant command
---

```
(layers/nxp-ls20xx/README)

6. Creating Partitioned Images(WIC)
===================================

User can use the OpenEmbedded Image Creator, wic, to create the properly
partitioned image on a SD card. The wic command
generates partitioned images from existing OpenEmbedded build artifacts.
User can refer to the below URL to get more WIC details:

http://www.yoctoproject.org/docs/2.4/mega-manual/mega-manual.html#creating-partitioned-images-using-wic

This BSP supports disk images for SD card.
After build the project, user will get a WIC image under the directory
tmp-glibc/deploy/images/<bsp name>/ ,such as:

tmp-glibc/deploy/images/nxp-ls20xx/wrlinux-image-glibc-small-nxp-ls20xx.wic

Then user can write the output image to a SD card:

Since this BSP doesn't have a firmware to read the uboot from a partition table,
WIC image only contains kernel, dtb and rootfs. We still need to write U-boot
image to NOR flash directly as "bootloader/README" described.

6.1 Burn images to SD card
--------------------------

To burn uboot and WIC images to SD card, user only need to execute the below command:

# dd if=wrlinux-image-glibc-small-nxp-ls20xx.wic of=/dev/your_sd_dev

6.2 Set uboot env
-----------------

Board can boot automatically by set the below uboot environment variables:

=> setenv bootfile Image; setenv fdtfile  fsl-ls2088a-rdb.dtb;  setenv loadaddr 0xa0000000; setenv fdtaddr 0x90000000;

=> setenv bootargs console=ttyS1,115200n8 root=/dev/mmcblk0p2 rw no_console_suspend ip=dhcp

=> setenv bootcmd 'fsl_mc start mc 0x580a00000 0x580e00000; fsl_mc lazyapply dpl 0x580d00000; fatload mmc 0:1 $loadaddr $bootfile; fatload mmc 0:1 $fdtaddr $fdtfile; booti $loadaddr - $fdtaddr'

=> saveenv; run bootcmd;
7. Enable on-board 10G ports in kernel
======================================

LS2088A-RDB contains 8 on-board 10G ports, 4 'SFP+ Optical ports' and 4 'Copper ports'.

They can be found in u-boot as below:
DPMAC1@xgmii to DPMAC4@xgmii are Optical ports,
DPMAC5@xgmii to DPMAC8@xgmii are Copper ports.

These ports' status are controlled by DPL. SDK provides a default DPL:

flexbuild/build/firmware/dpl-examples/ls2088a/RDB/dpl-eth.0x2A_0x41.dts

it only enables the first Copper port: DPMAC5@xgmii.
To enable other ports in kernel, user can modify this DPL or through an
userspace application "restool" which is provided by SDK.
Unfortunately, both DPL and "restool" have NXP's private license,
WindRiver Linux can't merge them.
So if user want to enable other ports in WindRiver Linux, user can
update SDK's DPL's "connections" section manually:

        connections {
                connection@0{
                        endpoint1 = "dpni@0";
                        endpoint2 = "dpmac@5";
                };
        };


For instance, if user want to enalbe the first Optical port, can replace

                        endpoint2 = "dpmac@5";
to

                        endpoint2 = "dpmac@1";

Then generate a new DPL on Host:

dtc -I dts -O dtb dpl-eth.0x2A_0x41.dts -o dpl.dtb

Upload this DPL to board and use it to init fsl_mc in u-boot's command line:

tftp 0xa9000000 dpl.dtb;fsl_mc start mc 0x580a00000 0x580e00000;fsl_mc lazyapply dpl 0xa9000000;

Other boot commands are same as usual.

To get more details about above descriptions, please refer to instructions
mentioned in the section

        "7.2.3.3 Data path layout file (DPL)"
in the NXP's SDK user manual

        "Layerscape Software Development Kit 17.09 Documentation.pdf".

8. kexec/kdump
==============

For discussion purposes, some useful terminology will be described here.

boot kernel - the first kernel that you start and supports kexec,
              from U-Boot for instance.
capture kernel - the kernel that you reboot into after a boot kernel crash.

To build the boot or capture kernel, use the following option with the configure
command for your project:

        --template=feature/kexec,feature/kdump

To reserve a memory region for the capture kernel, pass "crashkernel=512M"
via the U-Boot command line.

NOTES:
1. Use vmlinux as a secondary kernel. It can be found in the directory
   tmp-glibc/work/<bsp name>-wrs-linux/linux-yocto/<kernel version>/linux-<bsp name>-<kernel type>-build/vmlinux
2. 512 MB (the size of the memory reserved for crash kernel) is the
   recommended amount of memory. If you add more features to the kernel, you
   can increase this value to accommodate the capture kernel.

Workaround:
1. Kexec: DPAA2 is unsupported in 2nd kernel.
   User can use on-board Intel E1000e card to do Ethernet functions in 2nd
   kernel(e.g., mount NFS, scp).
   User can turn DPAA2 off in two ways:
a) Don't call "fsl_mc start" in U-Boot command line, this will disable the
   DPAA2 both in the 1st and 2nd kernel.
b) If 1st kernel has already enabled DPAA2, please use the below 3 commands
   to destroy all DPNI interfaces in 1st kernel before calling "kexec -e":

   # restool dprc show dprc.1 | grep dpni
   /* Above command is to find all DPNI interfaces. */

   # echo dpni.0 > /sys/bus/fsl-mc/devices/dpni.0/driver/unbind
   /* Unbind DPNI interface from fsl_mc driver */

   # restool dpni destroy dpni.0
   /* Destroy this DPNI interface */

2. Kdump: PCI-MSI and SMP is unsupported in 2nd kernel.
   User needs to append two arguments "pci=nomsi maxcpus=1".
   Since this 2nd kernel operates under significantly constrained
   initialization conditions and is not meant to be a "replacement" kernel.
   Rather it has a single goal -- to enable data collection with respect to
   the issue that impacted the initial kernel, and nothing else.

For more detailed information about kdump, please refer to
Documentation/kdump/kdump.txt in the kernel source tree.


```
