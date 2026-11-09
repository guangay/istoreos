# iStoreOS 24.10 for XiGuaPi V3 (XGPV3)

基于 [iStoreOS](https://github.com/istoreos/istoreos) `istoreos-24.10` 分支（内核 6.6，ipk/opkg 包体系），为 **NLnet XiGuaPi V3（RK3568）** 定制的自动构建固件。

每周六 20:00（北京时间）自动构建；workflow 变更也会触发。构建成功后固件自动发布到 [Releases](../../releases)。

---

## 一、设备信息

| 项目 | 规格 |
|---|---|
| 型号 | NLnet XiGuaPi V3（西瓜派 V3） |
| SoC | Rockchip RK3568（4×Cortex-A55） |
| GPU | Mali-G52（panfrost 开源驱动） |
| 网口 | 双 GMAC + RTL8211F（eth0=LAN，eth1=WAN） |
| 无线 | MT7915/MT7916（PCIe，mt76 开源驱动 + 官方固件） |
| 5G 模组 | Quectel RM520N-GL 等（PCIe MHI，rmnet_mhi） |
| 屏幕 | GC9307 SPI 彩屏（SPI3，背光 PWM14） |
| 存储 | TF 卡 / eMMC / NVMe |

## 二、固件包含的组件

| 组件 | 来源 | 说明 |
|---|---|---|
| **QModem** | [FUjr/QModem](https://github.com/FUjr/QModem) `main`（feed 方式集成） | 5G/4G 模组管理框架：`qmodem` 核心 + `luci-app-qmodem-next`（JS 前端）+ `kmod-pcie_mhi`（Quectel MHI PCIe 驱动） |
| **屏幕驱动与显示** | [junhong-l/xgp-v3-screen](https://github.com/junhong-l/xgp-v3-screen) | `kmod-fb-tft` + `kmod-fb-tft-gc9307` + `xgp-v3-screen`（LuCI 页面含亮度调节） |
| **MT7915/MT7916 无线** | iStoreOS `mt76`（OpenWrt 主线） | `kmod-mt7915e` + `kmod-mt7915-firmware` + `kmod-mt7916-firmware` |
| **Mali GPU（panfrost）** | 包定义移植自 ImmortalWrt | 注入 `kmod-drm-panfrost` + `kmod-drm-sched`（24.10 上游两者皆无），依赖 `kmod-drm` + `kmod-drm-shmem-helper` |
| **iStore 商店** | [linkease/istore](https://github.com/linkease/istore) | `luci-app-store` + taskd 链；构建中显式选中 `xz-utils` 以解除 `TAR_XZ` 守卫导致的静默丢包 |
| **AP 守护** | iStoreOS 主线 | `wpad-basic-mbedtls`（WPA2-PSK） |
| **Docker** | iStoreOS 主线 | `dockerd` + `docker` + `docker-compose` + `luci-app-dockerman`（容器管理页面）+ `ttyd`；已开启内核 cgroups/namespaces/veth；rootfs 为 f2fs，与 overlay2 不兼容，默认存储驱动设为 `vfs`（数据目录 `/opt/docker/`） |
| **TurboACC 网络加速** | [chenmozhijin/turboacc](https://github.com/chenmozhijin/turboacc)（`luci` 分支，官方 `add_turboacc.sh` 集成） | `luci-app-turboacc`：BBR + 流量分载 + Shortcut-FE + FullCone NAT；对内核打 SFE 补丁（6.6 受官方支持）并替换 firewall4/nftables 为补丁版 |
| **Kucat 主题** | [sirpdboy/luci-theme-kucat](https://github.com/sirpdboy/luci-theme-kucat) | `luci-theme-kucat`（JS 版主题，在「系统 → 语言和界面」切换） |
| **Lucky** | [gdy666/luci-app-lucky](https://github.com/gdy666/luci-app-lucky)（官方 OpenWrt 打包） | `lucky` + `luci-app-lucky`：DDNS / 反向代理 / 端口转发 / AList 等 |
| **UPnP** | iStoreOS 主线 | `luci-app-upnp` + `miniupnpd` |
| **网页终端** | iStoreOS 主线 | `luci-app-ttyd`（ttyd 随 Docker 已装） |
| **SmartDNS** | iStoreOS 主线 | `luci-app-smartdns`：DNS 加速与分流 |
| **OpenList 网盘** | iStoreOS 主线 | `luci-app-openlist`：多网盘聚合挂载（AList 社区后继） |
| **QuickStart 首页/网络向导** | linkease 官方（[nas-packages-luci](https://github.com/linkease/nas-packages-luci) + [nas-packages](https://github.com/linkease/nas-packages)） | `luci-app-quickstart` + `quickstart` 后端：iStoreOS 风格首页与快速配网向导 |

## 三、默认系统配置

| 项目 | 值 |
|---|---|
| 后台地址 | `10.10.10.1`（LAN=eth0，掩码 255.255.255.0） |
| 账号 / 密码 | `root` / `qweasd` |
| 主机名 | `XGPV3` |
| 时区 | Asia/Shanghai (CST-8) |
| **WiFi** | **默认开启**，SSID `XGPV3`（与主机名一致），密码 `88888889`（WPA2-PSK，国家码 US） |
| SSH / LuCI | 默认启用（dropbear / uhttpd） |

> 通过构建时注入的 `uci-defaults/99-xgpv3-defaults` 实现：网络、主机名、root 密码（SHA-512 crypt，构建时生成）、无线启用与 SSID/密钥。无线配置不存在时先执行 `wifi config` 生成再修改，兼容首启时序。

## 四、构建

workflow：`.github/workflows/build-xgpv3.yml`

- 触发：手动 / workflow 文件 push / 每周六 20:00（`cron 0 12 * * 6`）
- 关键步骤：
  1. 注入 `kmod-drm-sched` + `kmod-drm-panfrost` 内核包定义
  2. 生成 `99-xgpv3-defaults`（含 root 密码哈希，构建时生成）
  3. 追加 QModem feed（FUjr）+ 屏幕包软链
  4. `make defconfig` 后 fail-fast 校验：设备符号、屏幕三件套、panfrost、qmodem、store、wpad 任一缺失即**立即失败**
  5. 编译 → manifest 摘要 → 上传 Artifacts → **发布 Release**（tag `xgpv3-2410-r<构建号>`）

**产物**：

- [Releases](../../releases)（推荐，永久）：两个 sysupgrade 镜像 + manifest + config.buildinfo + sha256sums
- Actions Artifacts（90 天过期）：同上 + 完整构建日志

镜像：

- `istoreos-rockchip-armv8-nlnet_xiguapi-v3-squashfs-sysupgrade.img.gz`（刷机/升级）
- `istoreos-rockchip-armv8-nlnet_xiguapi-v3-ext4-sysupgrade.img.gz`（TF 卡/扩容）

## 五、刷机

```bash
gzip -d istoreos-rockchip-armv8-nlnet_xiguapi-v3-squashfs-sysupgrade.img.gz
# 用 RKDevTool / dd 写入 TF 卡或 eMMC，或从旧固件后台直接 sysupgrade
```

首启后验证：

```bash
ip addr show br-lan          # 10.10.10.1
iw dev                       # 无线接口应已 up
lsmod | grep -E 'panfrost|fb_gc9307|pcie_mhi|mt7915'
cat /etc/openwrt_release
```

## 六、已知限制

1. **包体系是 ipk/opkg**（24.10），与 25.12 的 apk 不通用；安装软件用 `opkg install`。
2. `kmod-pcie_mhi`（Quectel MHI）在 6.6 内核上的兼容性以构建结果为准；若编译失败会暂时移除并在此注明。
3. MT7916 走 OpenWrt 主线 mt76 开源驱动，非联发科原厂闭源驱动。
4. iStore 商店若后续再出现丢包，fail-fast 校验会让构建直接失败（不会带病出固件）。
5. **Docker 使用 vfs 存储驱动**：rootfs 为 f2fs，overlay2 在其上不兼容。vfs 功能完整但每个容器层都完整复制、占用空间较大；如需 overlay2 性能，可把 `data_root` 指向 ext4/btrfs 格式的 NVMe 分区后改回。

## 七、致谢与许可

- [iStoreOS](https://github.com/istoreos/istoreos)（固件基座）
- [FUjr/QModem](https://github.com/FUjr/QModem)（5G 模组框架）
- [junhong-l/xgp-v3-screen](https://github.com/junhong-l/xgp-v3-screen)（屏幕驱动与显示）
- [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)（panfrost 包定义参考）
- 各上游项目许可证不变；本仓库的构建配置与设备适配文件遵循 GPL-2.0。
