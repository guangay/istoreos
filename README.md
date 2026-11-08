# iStoreOS 25.12 for NLnet XiGuaPi V3 (XGPV3)

本仓库是 [istoreos/istoreos](https://github.com/istoreos/istoreos) 在 `istoreos-25.12` 分支上的**定制分支**，用于给 **NLnet 西瓜派 V3（XGPV3，RK3568）** 编译完整可用的 iStoreOS 固件，由 GitHub Actions 自动化构建。

> 这不是通用 iStoreOS 固件，只适用于 XGPV3 这一块板子。其他 RK3568 设备请勿刷入。

---

## 一、设备信息

| 项目 | 规格 |
|---|---|
| 设备 | NLnet XiGuaPi V3（XGPV3） |
| SoC | Rockchip RK3568（4×Cortex-A55，ARMv8） |
| GPU | Mali-G52（Bifrost，panfrost 驱动） |
| 内存 / 存储 | LPDDR4 / eMMC（f2fs 根分区） |
| 有线网口 | 双 GMAC + RTL8211F PHY（eth0 = LAN，eth1 = WAN） |
| 无线 | MT7915 / MT7916（PCIe，mt76 开源驱动） |
| 5G 模组 | Quectel RM520N-GL（PCIe + MHI，接口 `rmnet_mhi0.x`；也支持 USB3 QMI/MBIM） |
| 屏幕 | GC9307 240×320 SPI LCD（SPI3 + PWM14 背光） |
| 其他 | RK817 PMIC、HDMI 输出、风扇 PWM、NVMe |

---

## 二、固件包含的组件

| 组件 | 版本 / 上游来源 | 在本固件中的作用 |
|---|---|---|
| **iStoreOS 25.12** | [istoreos/istoreos](https://github.com/istoreos/istoreos) `istoreos-25.12` | 系统底座，Linux 6.12.112，含 iStoreOS 定制（`istoreos-files`、LuCI 主题、任务计划等） |
| **设备 DTS** | 移植自 [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) 主线风格 DTS，落在 `target/linux/rockchip/dts/rk3568/rk3568-xiguapi-v3.dts` | 定义双网口、MT7915、5G、GC9307 屏幕、GPU、风扇等硬件节点 |
| **U-Boot** | upstream u-boot 2025.10，复用 `generic-rk3568_defconfig` | 新增 `xiguapi-v3-rk3568` 目标，产出 `*-u-boot-rockchip.bin` 并打包进镜像 |
| **QModem** | [FUjr/QModem](https://github.com/FUjr/QModem) `main`（以 feed 方式 `src-git qmodem`） | 5G/4G 模组管理框架：`qmodem` 核心 + `luci-app-qmodem-next`（新版 JS 前端）+ `kmod-pcie_mhi`（Quectel PCIe MHI 驱动，提供 `rmnet_mhi` 接口） |
| **XGP V3 Screen** | [junhong-l/xgp-v3-screen](https://github.com/junhong-l/xgp-v3-screen) `2.2` | 屏幕主程序 + LuCI 界面 + GC9307 驱动；开机由 `zz_xgp_screen` 服务自动拉起 |
| **GC9307 驱动** | 同上（`kmod-fb-tft-gc9307`） | 基于内核 fbtft 框架的 SPI LCD 驱动，`FBTFT_REGISTER_DRIVER("fb_gc9307","galaxycore,gc9307")`，`AutoLoad 09` |
| **fbtft 框架** | OpenWrt/iStoreOS 标准内核包 `kmod-fb-tft` | 屏幕驱动的基础框架 |
| **MT7915/MT7916 无线** | iStoreOS `mt76`（OpenWrt 主线 mt76，非联发科闭源驱动） | `kmod-mt7915e`（同一驱动覆盖 MT7915 与国产 MT7916）+ `kmod-mt7915-firmware` + `kmod-mt7916-firmware`（`mt7916_wa/wm/rom_patch.bin`）；两者共用 `mt7915e.ko`，MT7916 由固件包区分 |
| **Mali GPU（panfrost）** | 包定义移植自 [immortalwrt/immortalwrt](https://github.com/immortalwrt/immortalwrt)，构建时追加进 `package/kernel/linux/modules/video.mk` | `kmod-drm-panfrost`，依赖 `kmod-drm` / `kmod-drm-sched` / `kmod-drm-shmem-helper`；这三个依赖带 `@DISPLAY_SUPPORT`，所以 `.config` 必须同时打开 `CONFIG_DISPLAY_SUPPORT=y`（iStoreOS 默认关闭，不打开会静默丢包） |
| **iStore 商店** | [linkease/istore](https://github.com/linkease/istore)（`store` feed） | `luci-app-store`（是否成功装入以构建摘要为准） |

所有第三方组件均通过构建时拉取（feed 或 git clone），仓库本身不存放第三方源码。

---

## 三、默认系统配置

首次**不保留配置**刷入后，`/etc/uci-defaults/99-xgpv3-defaults` 会自动应用以下设置：

| 项目 | 值 |
|---|---|
| LAN 地址 | `10.10.10.1/24`（DHCP 池 `10.10.10.100-250`） |
| 用户名 / 密码 | `root` / `qweasd` |
| 主机名 | `zzXGP` |
| 时区 | `Asia/Shanghai`（CST-8） |
| 管理入口 | `http://10.10.10.1`（LuCI） / `ssh root@10.10.10.1` |

> 通过 sysupgrade **保留配置**升级时，上述默认值不会覆盖你已有的配置。

---

## 四、构建

工作流：`.github/workflows/build-xgpv3.yml`

**触发方式**

1. 手动：Actions → `Build iStoreOS 25.12 for XGPV3` → Run workflow
2. 推送：修改 workflow 文件本身会自动触发
3. **定时：每周六 20:00（北京时间 / UTC `0 12 * * 6`）自动构建一次**

**构建流程要点**

- 清洗 `package/boot/uboot-rockchip/Makefile`（截断到首个 `BuildPackage/U-Boot`，避免重复定义）
- 追加 `kmod-drm-panfrost` 内核包定义，并在 `.config` 里打开 `CONFIG_DISPLAY_SUPPORT=y`
- 生成 `99-xgpv3-defaults`（含 root 密码的 SHA-512 crypt hash，构建时生成）
- 添加 QModem feed（FUjr，`src-git qmodem`）与屏幕包（zz-packages 软链）
- 生成 `.config` 后做符号校验：设备定义、`DISPLAY_SUPPORT`、屏幕三件套、panfrost、qmodem、pcie_mhi 任一缺失即**立即失败**（几分钟内暴露，不必等编译完）
- 编译完成写 `GITHUB_STEP_SUMMARY`：关键包清单 + 镜像大小

**产物**

- **Releases（推荐）**：每次构建成功后自动发布到 [Releases](../../releases)，tag 形如 `xgpv3-r<构建号>`，含两个 sysupgrade 镜像 + manifest + config.buildinfo + sha256sums
- Actions 运行页 Artifacts（临时，90 天过期）：同上内容，另含 `build.log` / `prereq.log` / `download.log`

镜像文件：

- `istoreos-rockchip-armv8-nlnet_xiguapi-v3-squashfs-sysupgrade.img.gz`（刷机/升级用）
- `istoreos-rockchip-armv8-nlnet_xiguapi-v3-ext4-sysupgrade.img.gz`（TF 卡/扩容场景）

---

## 五、刷机

```bash
# 上传到设备后（不保留配置）
sysupgrade -n /tmp/istoreos-rockchip-armv8-nlnet_xiguapi-v3-squashfs-sysupgrade.img.gz
```

也可以直接在 LuCI「系统 → 备份/升级」上传镜像，取消勾选「保留配置」以应用默认 IP 和密码。

刷完开机后屏幕应自动点亮。若黑屏，SSH 上去检查：

```bash
dmesg | grep -i -E 'gc9307|fbtft|panfrost'
ls /dev/fb*
/etc/init.d/zz_xgp_screen status
```

---

## 六、已知限制

1. **CGNAT 环境**：若运营商分配的是私网地址（如 `10.x.x.x`），UPnP/PCDN 端口映射仍会失效，与固件无关。
2. **panfrost 仅内核驱动**：没有打包 mesa 用户态，OpenGL/EGL 不可用，只能做 DRM/KMS 与 framebuffer 加速测试。
3. **MT7916 使用开源驱动**：走 OpenWrt 主线 mt76，不是联发科原厂闭源驱动，因此 160MHz 等特性以 mt76 支持为准；若 5G/6G 组合异常，可在 LuCI 里把信道宽度降到 80MHz。
4. **5G 走 PCIe MHI**：`kmod-pcie_mhi` 是 Quectel 树外驱动，若某次内核升级后编译失败，会在构建日志里明确报错，可临时删掉 `CONFIG_PACKAGE_kmod-pcie_mhi=y` 改用 USB QMI 模式。
5. **iStore 商店**：已修复。根因是 tar 的 `PACKAGE_TAR_XZ` 默认开启、`xz-utils` 未选中时，`luci-app-store` 的守卫条件 `depends on !(PACKAGE_TAR_XZ)||PACKAGE_xz-utils` 为假，整个符号被 kconfig 静默丢弃（连 `# is not set` 都不写）。`.config` 显式加入 `CONFIG_PACKAGE_xz-utils=y` 后即可正常编译进固件，并把 store 加入 fail-fast 校验清单。
6. **屏幕亮度**：调节入口在「服务 → XGP Screen」页面内（原 `luci-app-xgp-v3-screen-brightness` 独立包在上游已合并，不再单独提供）。

---

## 七、致谢与许可

- [istoreos/istoreos](https://github.com/istoreos/istoreos) — 系统底座
- [coolsnowwolf/lede](https://github.com/coolsnowwolf/lede) — XGPV3 设备树来源
- [FUjr/QModem](https://github.com/FUjr/QModem) — 模组管理
- [junhong-l/xgp-v3-screen](https://github.com/junhong-l/xgp-v3-screen)、[zzzz0317/xgp-v3-screen](https://github.com/zzzz0317/xgp-v3-screen) — 屏幕程序与驱动
- [immortalwrt/immortalwrt](https://github.com/immortalwrt/immortalwrt) — panfrost 内核包定义
- [openwrt/openwrt](https://github.com/openwrt/openwrt) — 上游构建系统与 mt76

各组件版权归其原作者所有，遵循各自许可证（GPL-2.0 / GPL-3.0 等）。本仓库仅提供设备适配与构建脚本。
