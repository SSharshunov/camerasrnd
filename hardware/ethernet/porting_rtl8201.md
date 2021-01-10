> Данная статья является переводом с китайского языка. <br>Оригинал статьи находится тут https://blog.csdn.net/zpzyf/article/details/52187279

# RTL8201 портирование
Аппаратная платформа: NUC977  
Программная платформа: linux-3.10.101

Конфигурация ядра поддерживает
![](https://img-blog.csdn.net/20160815132942173)
![](https://img-blog.csdn.net/20160815132955970)
Драйвер модификации Realtek PHY для добавления поддержки rtl8201
```diff -Npur a/drivers/net/phy/realtek.c b/drivers/net/phy/realtek.c
--- a/drivers/net/phy/realtek.c 2016-05-12 10:32:15.000000000 +0800
+++ b/drivers/net/phy/realtek.c 2016-08-15 13:44:43.420720361 +0800
@@ -16,9 +16,16 @@
 #include <linux/phy.h>
 #include <linux/module.h>

-#define RTL821x_PHYSR      0x11
-#define RTL821x_PHYSR_DUPLEX   0x2000
-#define RTL821x_PHYSR_SPEED    0xc000
+//#define RTL821x_PHYSR        0x11
+//#define RTL821x_PHYSR_DUPLEX 0x2000
+//#define RTL821x_PHYSR_SPEED  0xc000
+/* page 0 register 30 - interrupt indicators and SNR display register */
+#define RTL8201F_ISR       0x1e
+/* page 0 register 31 - page select register */
+#define RTL8201F_PSR       0x1f
+/* page 7 register 19 - interrupt, WOL enable, and LEDs function register */
+#define RTL8201F_IER       0x13
+
 #define RTL821x_INER       0x12
 #define RTL821x_INER_INIT  0x6400
 #define RTL821x_INSR       0x13
@@ -29,6 +36,15 @@ MODULE_DESCRIPTION("Realtek PHY driver")
 MODULE_AUTHOR("Johnson Leung");
 MODULE_LICENSE("GPL");

+static int rtl8201f_ack_interrupt(struct phy_device *phydev)
+{
+   int err;
+
+   err = phy_read(phydev, RTL8201F_ISR);
+
+   return (err < 0) ? err : 0;
+}
+
 static int rtl821x_ack_interrupt(struct phy_device *phydev)
 {
    int err;
@@ -38,13 +54,29 @@ static int rtl821x_ack_interrupt(struct
    return (err < 0) ? err : 0;
 }

-static int rtl8211b_config_intr(struct phy_device *phydev)
+static int rtl8201f_config_intr(struct phy_device *phydev)
 {
    int err;

+   phy_write(phydev, RTL8201F_PSR, 0x0007);    /* select page 7 */
+
    if (phydev->interrupts == PHY_INTERRUPT_ENABLED)
-       err = phy_write(phydev, RTL821x_INER,
-               RTL821x_INER_INIT);
+       err = phy_write(phydev, RTL8201F_IER, 0x3800 |
+                       phy_read(phydev, RTL8201F_IER));
+   else
+       err = phy_write(phydev, RTL8201F_IER, ~0x3800 &
+                       phy_read(phydev, RTL8201F_IER));
+
+   phy_write(phydev, RTL8201F_PSR, 0x0000);    /* back to page 0 */
+
+   return err;
+}
+
+static int rtl8211b_config_intr(struct phy_device *phydev)
+{
+   int err;
+   if (phydev->interrupts == PHY_INTERRUPT_ENABLED)
+       err = phy_write(phydev, RTL821x_INER, RTL821x_INER_INIT);
    else
        err = phy_write(phydev, RTL821x_INER, 0);

@@ -64,6 +96,21 @@ static int rtl8211e_config_intr(struct p
    return err;
 }

+/* RTL8201F */
+static struct phy_driver rtl8201f_driver = {
+   .phy_id     = 0x001cc816,
+   .name       = "RTL8201F 10/100Mbps Ethernet",
+   .phy_id_mask    = 0x001fffff,
+   .features   = PHY_BASIC_FEATURES,
+   .flags      = PHY_HAS_INTERRUPT,
+   .config_aneg    = &genphy_config_aneg,
+   .read_status    = &genphy_read_status,
+   .ack_interrupt  = &rtl8201f_ack_interrupt,
+   .config_intr    = &rtl8201f_config_intr,
+   .driver     = { .owner = THIS_MODULE,},
+};
+
+
 /* RTL8211B */
 static struct phy_driver rtl8211b_driver = {
    .phy_id     = 0x001cc912,
@@ -96,16 +143,24 @@ static struct phy_driver rtl8211e_driver

 static int __init realtek_init(void)
 {
-   int ret;
+// int ret;

-   ret = phy_driver_register(&rtl8211b_driver);
-   if (ret < 0)
+// ret = phy_driver_register(&rtl8211b_driver);
+// if (ret < 0)
+    if(phy_driver_register(&rtl8201f_driver) < 0)
+    {
+       return -ENODEV;
+    }
+   if(phy_driver_register(&rtl8211b_driver) < 0)
+   {
        return -ENODEV;
+   }
    return phy_driver_register(&rtl8211e_driver);
 }

 static void __exit realtek_exit(void)
 {
+    phy_driver_unregister(&rtl8201f_driver);
    phy_driver_unregister(&rtl8211b_driver);
    phy_driver_unregister(&rtl8211e_driver);
 }
@@ -114,6 +169,7 @@ module_init(realtek_init);
 module_exit(realtek_exit);

 static struct mdio_device_id __maybe_unused realtek_tbl[] = {
+    { 0x001cc816, 0x001fffff },
    { 0x001cc912, 0x001fffff },
    { 0x001cc915, 0x001fffff },
    { }```

Отладка связанных команд
![](https://img-blog.csdn.net/20160815135115063)

