# Redmi K30 5G (picasso) postmarketOS 构建总结 — sherbook

- 日期：2026-10-31
- 机器：Arch 笔记本 sherbook（Ryzen 7 4700U，8 线程，15G 内存）
- 结果：**boot.img 构建成功**（见文末产物）

## 环境与来源

| 项 | 内容 |
|---|---|
| pmbootstrap | 3.12.1，从 gitlab 源码直跑（`~/k30-pmos/pmbootstrap-src/pmbootstrap.py`，venv `~/k30-pmos/venv`，Python 3.14.8）。原因：PyPI 清华镜像没有 pmbootstrap 3.x 的任何文件（只有被 yank 的 1.x–2.1.0），pip 装不上 |
| pmaports | `git clone --depth 1` 自 gitlab.postmarketos.org（经 FlClash 代理 127.0.0.1:7890；直连 nura.eco 镜像只有约 20KB/s，走代理不到一分钟拉完 119M） |
| picasso 两个包 | 从维护者本机正本 scp 到笔记本后拷入 `pmaports/device/downstream/`（GitHub 直连不通时的兜底，与本机此前成功出 apk 的版本一致） |
| 配置 | device=xiaomi-picasso，ui=console，user=user，hostname=xiaomi-picasso，timezone=Asia/Shanghai，work=`~/k30-pmos/work`，二进制镜像走 init 自动选的中科大镜像 |

## 执行步骤与结果

1. `pmbootstrap init`（PTY 脚本自动应答）— 成功，work version=8。脚本曾把 user/hostname 误答为 `y`，已用 `pmbootstrap config user user` / `config hostname xiaomi-picasso` / `config ui console` 修正并用 getter 复核。
2. `pmbootstrap checksum linux-xiaomi-picasso device-xiaomi-picasso` — **通过（exit 0），两个 APKBUILD 内容零改动**，所有记录的 sha512 原本就正确（含内核 tarball 的 b1a22324…，与实下载文件一致）。
3. `pmbootstrap build linux-xiaomi-picasso` — **一遍通过**，19:09–19:25（约 16 分钟），产物 `linux-xiaomi-picasso-4.19.325-r1.apk`（20M）。内核总共只编了这一遍。
4. `pmbootstrap build device-xiaomi-picasso` — 通过，产物 `device-xiaomi-picasso-1-r0.apk`。
5. `pmbootstrap install --no-fde`（PTY 脚本驱动，用户密码设为 <install-time password, not published>）— 通过，rootfs 镜像 `xiaomi-picasso.img` = 1,363,148,800 字节。
6. `pmbootstrap export` — 通过，导出目录 `/tmp/postmarketOS-export/`（boot.img、vmlinuz、initramfs、dtbs/、xiaomi-picasso.img），其中 dtbs 里是 `picasso-sm7250.dtb`。

## 过程中修过的环境问题（均未改包内容）

- 缺宿主机程序 kpartx → `sudo pacman -S multipath-tools`。
- chroot 内 busybox wget 经代理拉 GitHub 内核 tarball 返回 HTTP 502 → 改在宿主机用 curl 经代理下载 192M tarball，验 sha512 后放入 `work/cache_distfiles/`，abuild 直接命中缓存。
- scp 保留了本机的 640/750 文件权限，pmbootstrap 回拷 APKBUILD 失败 → 两个包目录 `chmod -R a+rX` 后解决。

## 产物

| 文件 | 大小 | 说明 |
|---|---|---|
| `~/k30-pmos/artifacts/boot.img` | 60,174,336 字节（58M） | sha256 `b4e40f79a6e7d6c9eb90e97310a03a7aaaa1723f135e3ff587bbd76102183cb5`；`file` 识别为 Android bootimg，page size 4096，kernel 偏移 0x8000，ramdisk 偏移 0x1000000，cmdline 为 picasso 的 androidboot 参数 |
| `~/k30-pmos/artifacts/xiaomi-picasso.img` | 1,363,148,800 字节（1.3G） | rootfs 镜像（console UI，systemd），用户 user / 密码 <install-time password, not published> |

原始日志：`~/k30-pmos/build.log`（checksum + 两次 build）、`~/k30-pmos/install.log`、`~/k30-pmos/export.log`、`~/k30-pmos/work/log.txt`。

下一步（未做）：把 boot.img + rootfs 镜像传到能连手机的机器上，`fastboot boot boot.img` 实机验证。
