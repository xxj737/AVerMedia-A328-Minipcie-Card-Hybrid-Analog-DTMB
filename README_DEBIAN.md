# SAA7231 (圆刚 A328 / [1131:7231]) Debian 11 驱动编译与装载


本目录是经过 **kernel 5.10 适配** 的版本，专用于 Debian 11 (5.10.0-46-amd64)。

## 已做的适配 (相对上游)

1. **saa7231_drv.c**
   - 屏蔽所有 dvb-frontend 依赖 (cxd2850/cxd2817/cxd2861/tda18272/stv090x/stv6110x/lnbh24/tda10048/s5h1411/cxd2820r/a8290)。
     这些芯片在 5.10 主线部分不存在，且 A328 根本不用它们。
   - `frontend_attach` 第一阶段改为空 stub（只注册 DVB adapter、不挂前端），
     目的是先把**芯片内部 4 条 I2C 总线**拉起来，用 `i2cdetect` 探测
     LGS8G75 (DTMB 解调) 和 TDA18271 (调谐) 到底挂在哪条总线上。
   - 屏蔽 v4l2 视频功能相关 include（视频功能本驱动未启用）。

2. **saa7231_pci.c**
   - 修复上游 bug: `msi_vectors_max` 使用了未初始化的 `msi_cap`（读取顺序颠倒），
     5.10 下可能拿到垃圾值导致内存分配异常。已把 `pci_read_config_dword(0x40)`
     移到使用之前。

3. **Makefile**
   - 改为独立 out-of-tree 编译，自动探测 dvb-core 头文件位置。

## 使用步骤 (在 Debian 上, 用 root)

### 1. 传代码 (在 Windows PowerShell)
```

### 2. 环境诊断
```
su -
cd /home/xxj/A328_D_Driver
bash check.sh
```
把输出贴回来。重点看:
- 第 4 节 "dvb-core 头文件位置" 是否 FOUND
- 第 3 节 CONFIG_DVB_LGS8GXX / CONFIG_MEDIA_TUNER_TDA18271 是否 =m 或 =y

### 3. 编译
```
bash build.sh
```
(等价于 `make`。若头文件没找到会明确报错，再按提示装 linux-source)

### 4. 装载 (只读侦察 + 驱动初始化)
```
bash load.sh
```
装载后把 `dmesg | tail -40` 贴回来。

**预期正常输出**:
- dmesg 显示 "Loading SAA7231 ver 0.0.91"
- "SAA7231 PCI Express V1x found"
- "SAA7231 I2C Core succesfully initialized" (4 条总线)
- "SAA7231 device:0 initialized"
- /dev/i2c-N 出现 4 条新总线 (SAA7231 I2C:0..3)
- /dev/dvb/adapter0..1 出现 (无前端)

### 5. 用 i2cdetect 探测前端芯片
```
apt install -y i2c-tools
i2cdetect -l | grep SAA7231        # 找到 4 条总线编号
i2cdetect -y <bus>                 # 逐个探测 0x00-0x7f
```
TDA18271 常见地址: 0x60；LGS8G75 常见地址: 0x1b 或 0x08 (待探测确认)。
把探测结果贴回来 —— 这是第二阶段写 frontend_attach (LGS8G75+TD18271) 的依据。

## 安全说明
- 本驱动是 BlackGold 官方 GPL 驱动，probe 时的寄存器写操作 (CGU/MSI/I2C 初始化)
  是 SAA7231 芯片的标准初始化流程，与 A328 是同一颗桥芯片，风险可控。
- 第一阶段 frontend_attach 为空，不会触碰 LGS8G75/TDA18271 的未知寄存器。
- 装载前确保无残留模块: `rmmod saa7231_drv saa7231_core 2>/dev/null`

## 第二阶段预告
拿到 I2C 探测结果后，会新增一个 A328 专属 config:
- `frontend_attach`: 在正确的 I2C 总线上 `dvb_attach(lgs8gxx_attach)` + `dvb_attach(tda18271_attach)`
- `frontend_enable`: A328 的 GPIO 使能/复位时序 (LGS8G75 的 RST/EN 引脚)
- `ts0_cfg/ts0_clk`: 按 LGS8G75 串行 TS 输出配置
然后 Tvheadend 配置 DTMB (中国国标) 频率即可收台。
