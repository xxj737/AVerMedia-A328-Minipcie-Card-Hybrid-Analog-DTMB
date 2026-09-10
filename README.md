SAA7231（圆刚 A328 / [1131:7231]）Debian 11驱动程序的编译与加载
该驱动版本已针对kernel 5.10进行过优化，专为Debian 11（5.10.0-46-amd64）系统设计。

已做的适配 (相对上游)
saa7231_drv.c

屏蔽所有DVB-frontend相关的组件（cxd2850/cxd2817/cxd2861/tda18272/stv090x/stv6110x/lnbh24/tda10048/s5h1411/cxd2820r/a8290）。这些芯片在5.10版本中并不存在，而且A328也根本不需要它们。
frontend_attach 第一阶段改为使用空stub（仅注册DVB适配器，而不连接前端设备）。这样做的目的是先激活芯片内部的4条I2C总线，然后通过i2cdetect来检测LGS8G75（DTMB解调模块）和TDA18271（调谐模块）分别连接在哪条总线上。
屏蔽 v4l2 视频功能相关 include（视频功能本驱动未启用）。
saa7231_pci.c

修复了上游漏洞：msi_vectors_max 中使用了未初始化的msi_cap（数据读取顺序错误），在5.10版本中这可能会导致获取到无效数据，进而引发内存分配异常。现已将pci_read_config_dword(0x40)移到使用之前进行初始化处理。
Makefile

改为独立 out-of-tree 编译，自动探测 dvb-core 头文件位置。
使用步骤（在 Debian 上，需使用 root 权限）
1. 传代码（在 Windows PowerShell中）
2. 环境诊断
su -
cd /home/xxj/A328_D_Driver
bash check.sh
把输出贴回来。重点看:

第 4 节 “dvb-core 头文件位置” 是否已找到
第 3 节 CONFIG_DVB_LGS8GXX / CONFIG_MEDIA_TUNER_TDA18271 是否 =m 或 =y
3. 编译
bash build.sh
（等同于make。如果找不到头文件，会给出明确的错误提示，此时需按照提示安装linux-source）

4. 装载（只读侦察 + 驱动初始化）
bash load.sh
装载后把 dmesg | tail -40 贴回来。

预期正常输出:

dmesg显示“正在加载SAA7231 0.0.91版本”
“检测到SAA7231 PCI Express V1x”
“SAA7231 I2C核心已成功初始化”（4条总线）
“SAA7231设备：0已初始化”
/dev/i2c-N 出现 4 条新总线 (SAA7231 I2C:0..3)
/dev/dvb/adapter0..1 出现 (无前端)
5. 用 i2cdetect 探测前端芯片
apt install -y i2c-tools
i2cdetect -l | grep SAA7231        # 找到 4 条总线编号
i2cdetect -y <bus>                 # 逐个探测 0x00-0x7f
TDA18271的常见地址为0x60；LGS8G75的常见地址则为0x1b或0x08（需通过探测来确定）。请将探测结果反馈过来——这将是第二阶段编写frontend_attach函数时所需依据的资料。

安全说明

第一阶段frontend_attach为空，不会触碰LGS8G75/TDA18271的未知寄存器。
装载前确保无残留模块: rmmod saa7231_drv saa7231_core 2>/dev/null
第二阶段预告
拿到 I2C 探测结果后，会新增一个 A328 专属 config:

frontend_attach: 在正确的 I2C 总线上 dvb_attach(lgs8gxx_attach) + dvb_attach(tda18271_attach)
frontend_enable: A328的GPIO使能/复位时序（LGS8G75的RST/EN引脚）
ts0_cfg/ts0_clk: 按LGS8G75的串行TS输出模式进行配置，然后在Tvheadend中设置DTMB（中国国家标准）频率，这样就能接收频道了。
