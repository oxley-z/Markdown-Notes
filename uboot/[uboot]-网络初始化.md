```c
|-->initr_net(common/board_r.c)                        /* 初始化网络设备 */
    |-->eth_initialize(net/eth-uclass.c)
        |-->eth_common_init(net/eth_common.c)
            |-->phy_init(drivers/net/phy/phy.c)
        |-->uclass_first_device_check(drivers/core/uclass.c)
            |-->uclass_find_first_device(drivers/core/uclass.c)
            |-->device_probe(drivers/core/device.c)
                |-->device_of_to_plat(drivers/core/device.c)
                    |-->drv->of_to_plat
                        |-->fecmxc_of_to_plat(drivers/net/fec_mxc.c)/* 解析设备树信息 */
                |-->device_get_uclass_id(drivers/core/device.c)
                |-->uclass_pre_probe_device(drivers/core/uclass.c)
                |-->drv->probe(dev)
                    /* drivers/net/fec_mxc.c */
                    U_BOOT_DRIVER(fecmxc_gem) = {
                        .name    = "fecmxc",
                        .id    = UCLASS_ETH,
                        .of_match = fecmxc_ids,
                        .of_to_plat = fecmxc_of_to_plat,
                        .probe    = fecmxc_probe,
                        .remove    = fecmxc_remove,
                        .ops    = &fecmxc_ops,
                        .priv_auto    = sizeof(struct fec_priv),
                        .plat_auto    = sizeof(struct eth_pdata),
                    };
                    |-->fecmxc_probe(drivers/net/fec_mxc.c)/* 探测和初始化 */
                        |-->fec_get_miibus(drivers/net/fec_mxc.c)
                            |-->mdio_alloc(drivers/net/fec_mxc.c)
                            |-->bus->read = fec_phy_read;
                            |-->bus->write = fec_phy_write;
                            |-->mdio_register(common/miiphyutil.c)
                            |-->fec_mii_setspeed(drivers/net/fec_mxc.c)
                        |-->fec_phy_init(drivers/net/fec_mxc.c)
                            |-->device_get_phy_addr(drivers/net/fec_mxc.c)
                            |-->phy_connect(drivers/net/phy/phy.c)
                                |-->phy_find_by_mask(drivers/net/phy/phy.c)
                                    |-->bus->reset(bus)
                                    |-->get_phy_device_by_mask(drivers/net/phy/phy.c)
                                        |-->create_phy_by_mask(drivers/net/phy/phy.c)
                                            |-->phy_device_create(drivers/net/phy/phy.c)
                                                |-->phy_probe(drivers/net/phy/phy.c)
                                |-->phy_connect_dev(drivers/net/phy/phy.c)
                                    |-->phy_reset(drivers/net/phy/phy.c)
                            |-->phy_config(drivers/net/phy/phy.c)
                                |-->board_phy_config(drivers/net/phy/phy.c)
                                    |-->phydev->drv->config(phydev)
                                        /* drivers/net/phy/smsc.c */
                                        static struct phy_driver lan8710_driver = {
                                            .name = "SMSC LAN8710/LAN8720",
                                            .uid = 0x0007c0f0,
                                            .mask = 0xffff0,
                                            .features = PHY_BASIC_FEATURES,
                                            .config = &genphy_config_aneg,
                                            .startup = &genphy_startup,
                                            .shutdown = &genphy_shutdown,
                                        };
                                        |-->genphy_config_aneg(drivers/net/phy/phy.c)
                                            |-->phy_reset(需要手动调用)(drivers/net/phy/phy.c)
                                            |-->genphy_setup_forced(drivers/net/phy/phy.c)
                                            |-->genphy_config_advert(drivers/net/phy/phy.c)
                                            |-->genphy_restart_aneg(drivers/net/phy/phy.c)
                |-->uclass_post_probe_device(drivers/core/uclass.c)
                    |-->uc_drv->post_probe(drivers/core/uclass.c)
                        /* net/eth-uclass.c */
                        UCLASS_DRIVER(ethernet) = {
                            .name        = "ethernet",
                            .id        = UCLASS_ETH,
                            .post_bind    = eth_post_bind,
                            .pre_unbind    = eth_pre_unbind,
                            .post_probe    = eth_post_probe,
                            .pre_remove    = eth_pre_remove,
                            .priv_auto    = sizeof(struct eth_uclass_priv),
                            .per_device_auto    = sizeof(struct eth_device_priv),
                            .flags        = DM_UC_FLAG_SEQ_ALIAS,
                        };
                        |-->eth_post_probe(net/eth-uclass.c)
                            |-->eth_write_hwaddr(drivers/core/uclass.c)
```

