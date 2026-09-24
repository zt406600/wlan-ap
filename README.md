# OpenWiFi AP NOS

OpenWrt-based access point network operating system (AP NOS) for TIP OpenWiFi.
Read more at [openwifi.tip.build](https://openwifi.tip.build/).

> **京东云 ER2 / JDCloud ER2 用户**：本分支（`er2`）已包含该板子的完整板级支持，并附带了作者的
> 完整构建快照 `defconfig/jdc-er2.config`。想复刻出「和作者一致的固件（含完整 LuCI 网页界面）」，
> 请看下方 [JDCloud ER2 固件构建](#jdcloud-er2-固件构建) 一节。

## Building

### Setting up your build machine

Building requires a recent Linux installation. Older systems without Python 3.7
will have trouble. See this guide for details:
https://openwrt.org/docs/guide-developer/toolchain/beginners-build-guide

Install build packages on Debian/Ubuntu (or see above guide for other systems):
```
sudo apt install build-essential libncurses5-dev gawk git libssl-dev gettext zlib1g-dev swig unzip time rsync python3 python3-setuptools python3-yaml
```

### Doing a native build on Linux

Use `./build.sh <target>`, or follow the manual steps below:

1. Clone and set up the tree. This will create an `openwrt/` directory.
```shell
./setup.py --setup    # for subsequent builds, use --rebase instead
```

2. Select the profile and base package selection. This setup will install the
   feeds and packages and generate the `.config` file.
```shell
cd openwrt
./scripts/gen_config.py linksys_ea8300
```

3. Build the tree (replace `-j 8` with the number of cores to use).
```shell
make -j 8 V=s
```

### Build output

The build results are located in the `openwrt/bin/` directory:

| Type             | Path                                                 |
| ---------------- | ---------------------------------------------------- |
| Firmware images  | `openwrt/bin/targets/<target>/<subtarget>/`          |
| Kernel modules   | `openwrt/bin/targets/<target>/<subtarget>/packages/` |
| Package binaries | `openwrt/bin/packages/<platform>/<feed>/`            |

## JDCloud ER2 固件构建

本分支（`er2`）在 OpenWiFi AP NOS / OpenWrt 25.12 的基础上，为**京东云 ER2**
（DTS compatible `jdcloud,er2`，SoC ipq5332，target `ipq53xx/generic`）补上了板级支持，
并把作者长期在用的**完整 `.config` 快照**放在 `defconfig/jdc-er2.config` 里。
想编出「和作者尽量一致的固件」（含完整 LuCI 网页界面、QCA 硬件加速、fullcone 等），请用下面的
**方式 B（快照复刻）**；方式 A 只是最小系统。

### 两种构建方式（互斥，二选一）

| 方式 | 命令 | 结果 |
| --- | --- | --- |
| A. profile 流程 | `./build.sh jdcloud_er2` | 用 `profiles/jdcloud_er2.yml` 重新生成 `.config`：base system + 内核包 + ucentral 的 WebUI 依赖，约 80 个包。**没有完整 LuCI 界面**，也没有作者常用的工具/插件；而且 `gen_config.py` 会**删除**已有的 `.config` |
| B. 快照复刻 | 见「复刻作者的固件」 | 作者的完整配置快照：351 个 `CONFIG_PACKAGE_*=y`，与作者的成品基本一致 |

方式 B 的 `.config` 里包含（挑重点）：

- **完整 LuCI**：`luci`、`luci-ssl`、`luci-mod-admin-full`、`luci-mod-{network,status,system}`、
  `luci-app-firewall`、`luci-app-package-manager`、`luci-app-upnp`、`luci-app-nss-overview`、
  `luci-proto-{ppp,ipv6,relay}`、`luci-theme-argon` + `luci-theme-bootstrap`、简体中文语言包；
- **QCA 加速栈**：`kmod-qca-nss-dp-qca`、`kmod-qca-nss-phy`、`kmod-qca-nss-ppe`(+`-bridge-mgr`/
  `-vlan-mgr`/`-netlink`/`-pppoe-mgr`/`-ds`/`-rule`/`-vp`)、`kmod-qca-nss-sfe`、
  `kmod-qca-nss-ecm-standard`、`kmod-qca-ssdk-qca-nohnat`、`qca-ssdk-shell`、`kmod-qca8084`、
  `kmod-qca8k-cc`、`kmod-phylink`、`kmod-phy-aquantia`；
- **常用工具/子系统**：`curl`、`wget-ssl`、`jq`、`htop`、`tcpdump`、`ethtool-full`、`iperf3`、
  `smartmontools`、`lm-sensors`、`nvme-cli`、`mmc-utils`、`parted`/`e2fsprogs`/`f2fs` 系列、
  `wireguard` + `kmod-tcp-bbr`、`kmod-nft-fullcone`（fullcone NAT）等。

> 两种方式不要混用：先跑 A 会把 `.config` 覆盖掉；先 `cp` 快照再跑 `./build.sh` 也一样会被删掉。
> `profiles/jdcloud_er2.yml` 只服务于方式 A，方式 B 完全不走它。

### 复刻作者的固件（方式 B）

构建依赖（Debian/Ubuntu，其它发行版见上方 Building 一节）：

```bash
sudo apt install build-essential libncurses5-dev gawk git libssl-dev gettext \
  zlib1g-dev swig unzip time rsync python3 python3-setuptools python3-yaml
```

```bash
# 1. 取本分支
git clone -b er2 https://github.com/zt406600/wlan-ap.git wlan-ap-er2
cd wlan-ap-er2

# 2. 拉 OpenWrt 源码（config.yml 里固定到 openwrt-25.12 @ 3f081e25，并走 gh-proxy 镜像）
#    + 应用 patches-25.12（ER2 板级支持、NSS/PPE/ECM、fullcone、默认密码、smpackage feed…）
#    + 把本仓库的 profiles/ 链接进 openwrt/
./setup.py --setup

# 3. 国内网络建议先准备一个"只对这条命令生效"的 git 镜像，
#    否则下一步扫描 smpackage feed 时会在 feeds/smpackage/dockerd 上卡住（详见下面的注意事项）
printf '[url "https://gh-proxy.org/https://github.com/"]\n\tinsteadOf = https://github.com/\n' > /tmp/git-mirror

# 4. 准备 feed：gen_config.py 按 profile 生成 feeds.conf（含 qca feed 与 ipq53xx target）、
#    克隆 6 个 feed、生成索引；再用 install -a 把 feed 里的包挂到 package/feeds，
#    否则第 5 步里 .config 中的 feed 包会因"找不到包"被静默丢掉
cd openwrt
GIT_CONFIG_GLOBAL=/tmp/git-mirror ./scripts/gen_config.py jdcloud_er2
./scripts/feeds install -a

# 5. 用作者的完整快照替换 profile 生成的精简 .config，并归一化
cp ../defconfig/jdc-er2.config .config
GIT_CONFIG_GLOBAL=/tmp/git-mirror make defconfig

# 6. 编译
make -j$(nproc) V=s
```

产物在 `openwrt/bin/targets/ipq53xx/generic/`：
`openwrt-ipq53xx-jdcloud_er2-squashfs-sysupgrade.bin`（刷机用）、`…-squashfs-factory.bin`、
`…-initramfs-kernel.bin`（救援/救砖用），以及 `config.buildinfo`、`feeds.buildinfo`、
`…manifest`、`sha256sums`。其中 `feeds.buildinfo` 记录了本次实际用到的 feed revision，
可以直接和作者对比（作者本机是 packages `a3bd79d`、luci `650a6ca3`、routing `b2097c8`、
telephony `2618106`、video `094bf58`、smpackage `896b3f36`）。

刷完后网线接 **LAN 口**（`02_network` 里是 `ucidef_set_interfaces_lan_wan "eth0" "eth1.1"`：
`eth0` 为 LAN，下面交换口 2/3/4 是 LAN、1 是 WAN），浏览器打开 `http://192.168.1.1`，
用户名 `root`，默认密码 **`123456`**（`patches-25.12/0014-*`，刷完请第一时间改掉）。

`make defconfig`/`feeds update` 会把所有 feed 的软件包元数据扫一遍（几分钟）。其中
`feeds/smpackage/dockerd`、`feeds/smpackage/rblibtorrent` 的 Makefile 会联网访问
`github.com`（`git ls-remote`）和 `codeload.github.com`（tar 包），国内直连常常卡住。三种处理：

```bash
# a) 有代理：直接让它对所有下载生效（最稳妥，git 和 tar 包都覆盖）
export http_proxy=http://127.0.0.1:7890 https_proxy=http://127.0.0.1:7890

# b) 只用 git 镜像（即上面第 3 步的 /tmp/git-mirror）：能覆盖 github.com 的 git 操作，
#    实测可以顺利扫完整个 smpackage（含 dockerd/rblibtorrent）；
#    如果你这边仍卡住，说明卡在 codeload.github.com 的 tar 包下载，退回用 a) 的代理

# c) 干脆不要 smpackage feed：
sed -i '/small-package/d' feeds.conf.default
rm -rf feeds/smpackage feeds/smpackage.tmp
#    再按下一节把快照里来自该 feed 的符号关掉
```

### 自用包（作者私货，别人按需取舍）

快照里下面这些只是作者个人偏好，**和 ER2 的板级支持、NSS/PPE/ECM 加速无关**：

| 符号 | 说明 | 别人怎么办 |
| --- | --- | --- |
| `luci-app-wolultra` + `luci-i18n-wolultra-zh-cn` | 网络唤醒（WOL）界面 | 源码不在本仓库、也不在任何 feed 里，见下面两种做法 |
| `luci-app-mini-diskmanager` + `luci-i18n-mini-diskmanager-zh-cn` | 磁盘管理 | 同上 |
| `luci-theme-aurora` | 主题 | 同上 |
| `luci-app-daede_daed`、`luci-app-ssr-plus_Nftables_Transparent_Proxy`、`luci-app-ssr-plus_INCLUDE_{Shadowsocks_NONE_Client,Shadowsocks_NONE_Server,NONE_V2RAY}` | 来自第三方 feed `smpackage` 的代理插件 | 源是公开的，但该 feed **没锁版本**（见下）；不需要就在 menuconfig 里关掉 |
| `cloudflared`、`natmap`、`easytier`（正好是快照比作者本机 `.config` 少的 8 个符号：`cloudflared`/`luci-app-cloudflared`/`luci-i18n-cloudflared-zh-cn`、`natmap`/`luci-app-natmap`/`luci-i18n-natmap-zh-cn`、`luci-app-easytier`/`luci-i18n-easytier-zh-cn`） | 作者机器上没启用 | 快照里本来就是 `# … is not set`，想要自己在 menuconfig 打开 |

想要那 3 个"本仓库里没有源码"的包，自己 clone 到 `openwrt/package/` 后重新识别即可：

```bash
cd openwrt/package
git clone --depth 1 https://gh-proxy.org/https://github.com/VIKINGYFY/packages VIKINGYFY
git clone --depth 1 https://gh-proxy.org/https://github.com/4IceG/luci-app-mini-diskmanager
git clone --depth 1 https://gh-proxy.org/https://github.com/eamonxg/luci-theme-aurora
cd .. && make defconfig
```

不想要它们（推荐）：`make defconfig` 会因为"符号不存在"直接忽略这几行，不影响编译；想干净一点就：

```bash
cd openwrt
sed -i -E 's/^(CONFIG_PACKAGE_(luci-app-wolultra|luci-i18n-wolultra-zh-cn|luci-app-mini-diskmanager|luci-i18n-mini-diskmanager-zh-cn|luci-theme-aurora))=y/# \1 is not set/' .config
make defconfig
```

### 与作者不完全一致的地方

1. **`smpackage` feed 没有锁版本**：`patches-25.12/0127-feeds-add-smpackage-feed.patch` 加的是
   `src-git smpackage https://github.com/kenzok8/small-package`（没有 `^<sha>`；作者本机是 `896b3f36`）。
   来自这个 feed 的符号是：`luci-theme-argon`、`autocore-arm`、`kmod-nft-fullcone`
   （fullcone 的**内核模块**，来自 `feeds/smpackage/fullconenat-nft`）、`luci-app-daede_daed`、
   `luci-app-ssr-plus_*`。想完全对齐就自己 pin 一下：
   ```bash
   sed -i 's#kenzok8/small-package$#kenzok8/small-package^896b3f36#' openwrt/feeds.conf.default
   ./scripts/feeds update smpackage
   ```
2. **`luci-app-nss-overview` 不是 feed 包**，而是由 `patches-25.12/0125-*` 直接放进 `package/` 的，
   所以任何克隆都能拿到（含作者加的温度/NSS 行）。
3. **Indio UM-325BE 的 DTS 本地改动没有进本分支**，只影响那块板子，ER2 不受影响。
4. `defconfig/jdc-er2.config` 是"作者的 `.config` 快照"，不是 profile。你自己微调后
   （`make menuconfig`）可以 `cp openwrt/.config defconfig/jdc-er2.config` 一起提交，方便下次复刻。
5. 只想要最小系统、不需要 LuCI：`./build.sh jdcloud_er2` 就够了。若想走 profile 流程又带基础 Web 界面，
   往 `profiles/jdcloud_er2.yml` 的 `packages:` 里加：

```yaml
  # 基础 LuCI（可选）
  - luci
  - luci-ssl
  - luci-mod-admin-full
  - luci-app-firewall
  - luci-app-package-manager
  - luci-proto-ppp
  - luci-proto-ipv6
  - luci-theme-bootstrap
  - luci-i18n-base-zh-cn
  - luci-i18n-firewall-zh-cn
```

### 校验编出来的包是否和快照一致

```bash
grep -oP '^CONFIG_PACKAGE_\K[^=]+(?==y)' defconfig/jdc-er2.config | sort > /tmp/snap.txt
grep -oP '^CONFIG_PACKAGE_\K[^=]+(?==y)' openwrt/.config         | sort > /tmp/mine.txt
comm -3 /tmp/snap.txt /tmp/mine.txt   # 左列=快照有、你没有；右列=你有、快照没有
```

差异只应该出现在上面「自用包」那几项、以及 smpackage 的版本差异上。

## Developer Notes

### Branching model

- `main` - Stable dev branch
- `next` - Integration branch
- `staging-*` - Feature/bug branches
- `release/v#.#.#` - Release branches (*major.minor.patch*)

### Repository structure

Build files:
- `Makefile` - Calls Docker environment per target
- `dock-run.sh` - Dockerized build environment
- `docker/Dockerfile` - Dockerfile for build image
- `build.sh` - Build script
- `setup.py` - Clone and set up the tree
- `config.yml` - Specifies OpenWrt version and patches to apply

Directories:
- `feeds/` - OpenWiFi feeds
- `patches/` - OpenWiFi patches applied during builds
- `profiles/` - Per-target kernel configs, packages, and feeds
    - [wifi-ax](profiles/wifi-ax.yml): Wi-Fi AX packages
    - [ucentral-ap](profiles/ucentral-ap.yml): uCentral packages
    - [x64_vm](profiles/x64_vm.yml): x86-64 VM image

### uCentral packages

AP-NOS packages implementing the uCentral protocol include the following
repositories (refer to the [ucentral](feeds/ucentral/) feed for a full list):
- ucentral-client: https://github.com/Telecominfraproject/wlan-ucentral-client
- ucentral-schema: https://github.com/Telecominfraproject/wlan-ucentral-schema
- ucentral-wifi: https://github.com/blogic/ucentral-wifi
