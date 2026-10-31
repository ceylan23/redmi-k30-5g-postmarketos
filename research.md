# Redmi K30 5G (picasso) postmarketOS 下游移植 — 前期核实笔记

- 日期：2026-10-31
- 目标：为写 pmaports 的 `device-xiaomi-picasso` + `linux-xiaomi-picasso` 包提供已核实资料
- 核实方法（本文所有结论的来源方式）：
  - `git ls-remote` 查仓库/分支存在性（匿名访问，404 的仓库会要求用户名）
  - 浅克隆（`--depth 1 --filter=blob:none`）后直接读 Makefile / DTS / defconfig 文件
  - GitHub API 搜索（`search_repositories`）查仓库全名与默认分支
  - 克隆 pmaports master（https://gitlab.postmarketos.org/postmarketOS/pmaports.git ，克隆时自动重定向到 gitlab.nura.eco 镜像）读现行 deviceinfo / APKBUILD / schema
  - 网页搜索（XDA、PhoneDB、TrebleDroid wiki）
- 标记约定：**未查到** = 用上述方法认真查过但没有证据，不等于不存在。

---

## 0. 结论速览

| 项 | 结论 |
|---|---|
| 首选内核 | `LineageOS/android_kernel_xiaomi_sm7250`（分支见 §1.2）——同 SoC 的 pmaports 已合并设备 monet 就用这个仓库 |
| 备选内核 | `EndCredits/android_kernel_xiaomi_sm7250` 分支 `13.0`（picasso 专属 CAF rebase，4.19.269） |
| 小米官方源 | `MiCode/Xiaomi_Kernel_OpenSource` 有 `picasso-q-oss` / `picasso-r-oss`，但**按文件名检索不到 arm64 的 picasso/sm7250 vendor DTS**（见 §1.1 的注意事项），单独用它大概率拼不出可启动镜像 |
| LineageOS 官方 picasso device 树 | **不存在**（未查到）；picasso 的 device 树都在社区作者名下（PeterCxy / EndCredits / alecchangod） |
| boot.img | header version **2**，pagesize 4096，独立 DTBO 分区；A-only + system-as-root + 动态分区（super） |
| 模板 | pmaports `device/downstream/device-xiaomi-monet` + `linux-xiaomi-monet`（同为 SM7250、同内核仓库）——直接照抄改名 |
| 面板/触摸 | 面板代号 G7A（3 个批次变体，DSC video 模式）；触摸 = Novatek NT36672C（TDDI，SPI 接口） |
| WiFi/蓝牙 | 蓝牙 DTS 节点明确为 WCN3990；WiFi 走 icnss + qcacld-3.0 下游驱动 |
| 充电/电池 | PM7250B PMIC 内置充电（pm7250b_charger）+ QG 电量计；电池 4500 mAh，3 份 batterydata |

---

## 1. 内核源码候选

### 1.1 (a) 小米官方 MiCode/Xiaomi_Kernel_OpenSource

仓库：https://github.com/MiCode/Xiaomi_Kernel_OpenSource

`git ls-remote` 实测存在以下 picasso 分支（全仓没有其他 picasso 分支，也没有 lito 分支）：

| 分支 | 对应安卓 | 内核版本（读该分支 Makefile 实测） | HEAD commit | commit 日期 |
|---|---|---|---|---|
| `picasso-q-oss` | Android 10 (Q) | 4.19.81 | `236f9eb092623c2bc69fa2e88eb647ad30562049` | 2020-02-17 |
| `picasso-r-oss` | Android 11 (R) | 4.19.113 | `61e4abf674639d96a710cd9de44e68ec26eb09dd` | 2021-02-03 |

- defconfig：两分支都有 `arch/arm64/configs/picasso_user_defconfig` 和 `arch/arm64/configs/vendor/picasso_user_defconfig`。
- **注意事项（实测）**：对两分支做全树文件名检索，含 `picasso` 的文件只有上面两个 defconfig；含 `sm7250` 的文件为 **0 个**；`arch/arm64/boot/dts/vendor/` 路径不存在，`arch/arm64/boot/dts/qcom/` 下只有 36 个老式文件（apq/msm 系列，无 lito/sm7250）。也就是说 GitHub 上的官方 picasso 分支**没有随树附带 arm64 的 vendor 设备树**。写包时不要指望只靠官方源出 DTB；DTS 要从 §1.2/§1.3 的树里拿（它们都带 `arch/arm64/boot/dts/vendor/qcom/picasso-sm7250.dts/.dtsi`）。
- 官方源没有 Android 12 (S) 及以上的 picasso 分支（用户这台是 MIUI 13 / Android 12 固件，见 §6）。

### 1.2 (b) LineageOS 的 sm7250 内核 + device 树

**内核仓库存在**：`LineageOS/android_kernel_xiaomi_sm7250`
https://github.com/LineageOS/android_kernel_xiaomi_sm7250

`git ls-remote` 实测分支（只有这 4 条）：

| 分支 | HEAD commit |
|---|---|
| `lineage-18.1` | `e0bb4472e7550e1cfc1dc6b8b2aba39b34a4d78e` |
| `lineage-19.1` | `fe4abf71920f4685218fb9951ca9950ff6dac0a9` |
| `lineage-22.1` | `bf6ad6ec272d0d25c35ec41dcd0a02129638ce28` |
| `lineage-23.0` | `8e49d06af7ff5757c03ab181874cf1e90a1e1192`（默认分支） |

- `lineage-22.1` 浅克隆实测：Makefile 版本 **4.19.325**；树内含 picasso 全套 DTS（见 §2）；defconfig 有 `arch/arm64/configs/milito_defconfig`、`lito-perf_defconfig`、`vendor/lito_defconfig` 等（没有 picasso 专属 defconfig，picasso 在这棵树里走 milito/lito 配置，见 §3 的 common 树证据）。
- 注意：**没有** `lineage-20` / `lineage-21` 分支。

**device 树仓库**：

| 仓库 | 是否存在 | 说明 |
|---|---|---|
| `LineageOS/android_device_xiaomi_picasso` | **不存在**（ls-remote 返回 404/要求认证；GitHub 搜索无此仓库） | picasso 从来不是 LineageOS 官方设备 |
| `LineageOS/android_device_xiaomi_sm7250-common` | 存在 | 分支 `lineage-18.1` / `lineage-19.1` / `lineage-22.1` / `lineage-23.0`（默认 `lineage-23.0`；22.1 与 23.0 同一 commit `137463a1b23e011334195b8c6ffb08107273605a`）。但它服务的是官方设备（如 monet），picasso 的 BoardConfig 参数不要直接照抄它的 super 分区大小（见 §3 的不一致警告） |
| `LineageOS/android_device_xiaomi_monet` | 存在 | 分支同上 4 条；monet 是同 SoC 官方设备，可作对照 |

**picasso 社区 device 树（GitHub 搜索 + 逐仓核实）**：

| 仓库 | 核实到的分支/用途 |
|---|---|
| `PeterCxy/android_device_xiaomi_picasso` | 最早的 LOS 17.1 移植作者（XDA 帖见 §3）；实测可读其 `lineage-17.1` 分支 BoardConfig.mk（header v2 等参数见 §3） |
| `EndCredits/android_device_xiaomi_picasso` | 默认分支 `lineage-20.0`；本文 §2/§3 的多数 BoardConfig/fstab 数据取自此分支 |
| `EndCredits/android_device_xiaomi_picasso-oss` | 默认分支 `15`（从 common 树 inherit 的新式结构） |
| `alecchangod/android_device_xiaomi_picasso` | 默认分支 `lineage-22.1`；实测克隆：其 BoardConfig.mk 只 `include device/xiaomi/sm7250-common/BoardConfigCommon.mk`（配套 `alecchangod/android_device_xiaomi_sm7250-common` 分支 `lineage-22.1-reb`），对应社区 LOS 21/22 非官方构建 |
| 其他 | ArrowOS-Devices、aospa-picasso 等组织下也有同名仓库（多为 2021–2023 年的 ROM 适配），未逐一核实 |

### 1.3 其他内核候选

| 仓库 | 分支 | 说明 |
|---|---|---|
| `EndCredits/android_kernel_xiaomi_sm7250` | `13.0`（默认；另有 12.0 系列、android-4.19 等多条分支） | 仓库描述自述：“CAF rebased kernel based on MiCode/Xiaomi Kernel OpenSource for Redmi K30 5G (picasso)”。浅克隆实测 Makefile = **4.19.269**；带 `arch/arm64/configs/picasso_user_defconfig` 与 `vendor/picasso_user_defconfig`，且带 picasso DTS。这是 picasso 专属维护的 rebase 树（A13 时代社区 ROM 常用） |
| `xiaomi-sm7250-devs/android_kernel_xiaomi_sm7250` | `lineage-18.1` / `lineage-19.1` / `picasso-r-oss` / `test` | monet 社区维护组织；其中 `lineage-18.1`、`lineage-19.1` 的 HEAD 与 LineageOS 官方仓库完全一致（同 commit），`picasso-r-oss` 是另一条线 |
| `xiaomi-sm7250-devs/android_device_xiaomi_sm7250-common` | `lineage-18.1` / `lineage-19.1` / `test` | 同上组织的 common 树 |

**选型建议（写包用）**：首选 `LineageOS/android_kernel_xiaomi_sm7250 @ lineage-22.1`（commit `bf6ad6ec…`，4.19.325）——理由：pmaports 已合并的同 SoC 设备 monet 就固定在这个 commit 上（§4），config 基础同为 `milito_defconfig`，踩过的坑最少。若 milito 配置在 picasso 上点不亮屏，再换 EndCredits `13.0` + `vendor/picasso_user_defconfig` 组合。

---

## 2. 硬件细节

证据主要来自 `LineageOS/android_kernel_xiaomi_sm7250 @ lineage-22.1` 树内 DTS（路径均为 `arch/arm64/boot/dts/vendor/qcom/`）：
`picasso-sm7250.dts`、`picasso-sm7250.dtsi`（982 行）、`picasso-sm7250-overlay.dts`、`lito.dtsi`、`xiaomi-sm7250-common.dtsi`，以及 PhoneDB 规格页。

### 2.1 显示面板

- 规格：6.67″ IPS LCD，1080×2400，120 Hz（PhoneDB：https://phonedb.net/index.php?m=device&id=16954&c=xiaomi_redmi_k30_5g_standard_edition_dual_sim_td-lte_cn_64gb_m2001g7ae__m2001g7ac__xiaomi_picasso&d=detailed_specs ）；device 树里 `TARGET_SCREEN_DENSITY := 440`。
- 面板代号 **G7A**，下游 SDE（`lito-sde-display.dtsi`）驱动，DSC video 模式。树里有 3 个面板 dtsi 变体：
  - `dsi-panel-g7a-36-02-0c-dsc-video.dtsi` → “xiaomi 36 02 0c video mode dsc dsi panel”
  - `dsi-panel-g7a-37-02-0a-dsc-video.dtsi` → “xiaomi 37 02 0a video mode dsc dsi panel”
  - `dsi-panel-g7a-37-02-0b-dsc-video.dtsi` → “xiaomi 37 02 0b video mode dsc dsi panel”
- 三个变体由 bootloader 按面板 ID 选择（下游内核机制），写包时不用挑死，但**实机是哪个变体要从手机确认**（见 §6）。

### 2.2 触摸

`picasso-sm7250.dtsi` 第 132 行起 `ts_novatek` 节点（实测原文）：

- compatible = `"novatek,NVT-ts-spi"`，走 **SPI** 接口（节点在 `touch_spi@0` 下），reset GPIO = tlmm 4，irq GPIO = tlmm 5，TDDI reset = pm8150l_gpios 3。
- 芯片 = **Novatek NT36672C**（TDDI：触控+显示驱动合一），由固件文件名证实：`novatek_nt36672c_g7a_fw01.bin` / `…_fw02.bin` / `…_fw03.bin`（及对应 mp 文件）。
- 有 3 组 cfg（`novatek,cfg_0/1/2`），区别在 `tp-vendor`（0x41/0x41/0x46）和 `display-maker`（0x37/0x37/0x36）——即触摸固件也按面板批次分版本；`display-maker` 的 0x36/0x37 与 §2.1 面板变体的 36/37 编号对应。maker 编号对应的厂商名字（天马/华星等）**未查到**权威出处，不要写进包里。
- 含义：下游内核自带 Novatek SPI 驱动，固件 bin 需要从手机 `/vendor` 提取放进 firmware 包（后续阶段）。

### 2.3 WiFi / 蓝牙

- 蓝牙：`lito.dtsi` 第 556–566 行明确有 `bt_wcn3990` 节点，compatible = `"qca,wcn3990"`，注释 “Slimbus Slave DT for WCN3990” → **WCN3990** combo 芯片。
- WiFi：`lito.dtsi` 第 3682 行 `icnss: qcom,icnss@18800000`（compatible `"qcom,icnss"`，带 `qcom,wlan-msa-fixed-region` 固件保留内存）；内核树带下游驱动 `drivers/staging/qcacld-3.0` 与 `qca-wifi-host-cmn`。DTS 里没有单独写出 WLAN 芯片丝印型号；按 WCN3990 combo 架构，WLAN 侧与蓝牙同芯片（此为架构推断，DTS 直接证据只到 icnss + WCN3990 蓝牙节点为止）。
- 固件依赖：icnss 需要 WLAN 固件由 modem/WLAN firmware 镜像提供（PIL 加载，`pil_wlan_fw_mem` 保留区）——第一阶段 WiFi 能否起来取决于固件提取，属高风险项。

### 2.4 存储（UFS）

- `lito.dtsi`：`ufshc_mem: ufshc@1d84000`，compatible `"qcom,ufshc"`，aliases 里 `ufshc1` 为内嵌 UFS 槽；另有 `sdhc_1`（eMMC 命名槽位）和 microSD 扩展（PhoneDB 标注 TransFlash/microSD 扩展接口）。
- 介质规格：UFS 2.1（PhoneDB 128GB 版简介）。deviceinfo 里要照 monet 写 `deviceinfo_rootfs_image_sector_size="4096"`（UFS 必需）。

### 2.5 电池 / 充电

- `picasso-sm7250.dtsi` 第 703 行起 `&pm7250b_charger`：充电器在 **PM7250B PMIC** 内（qcom 下游 SMB 类充电驱动），带 `pm7250b_pdphy`（USB-PD），`qcom,sec-charger-config = <0>`。
- 电量计：`&pm7250b_qg`（QG），`qcom,battery-data = <&mtp_batterydata>`。
- 电池数据：`batterydata-picasso-G7ADEFAULT-4500mah.dtsi`、`batterydata-picasso-G7ANVTBM4P-4500mah.dtsi`、`batterydata-pi
...[truncated 7963 chars]