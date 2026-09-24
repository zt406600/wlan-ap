# OpenWiFi AP NOS

OpenWrt-based access point network operating system (AP NOS) for TIP OpenWiFi.
Read more at [openwifi.tip.build](https://openwifi.tip.build/).

> **京东云 ER2 / JDCloud ER2 用户**：本分支（`er2`）已包含该板子的完整板级支持，并附带一份
> 可直接使用的完整 `.config`（`defconfig/jdc-er2.config`，含 LuCI 网页界面）。
> 构建方法见下方 [JDCloud ER2 固件构建](#jdcloud-er2-固件构建) 一节。

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

京东云 ER2（DTS compatible `jdcloud,er2`，SoC ipq5332，target `ipq53xx/generic`）的板级支持、
QCA NSS/PPE/ECM 硬件加速补丁和默认配置都在本分支里。有两种构建方式，**二选一即可**：

| 方式 | 命令 | 结果 |
| --- | --- | --- |
| A. 脚本构建 | `./build.sh jdcloud_er2` | 按 `profiles/jdcloud_er2.yml` 生成 `.config`（base system + 内核/NSS 相关包，约 80 个包），**不含 LuCI 网页界面** |
| B. 手动构建 | 见下文「手动构建」 | 使用仓库自带的完整配置 `defconfig/jdc-er2.config`（351 个 `CONFIG_PACKAGE_*=y`），含完整 LuCI 网页界面 |

> 两种方式不要混用：`./build.sh` 会用 `gen_config.py` 重新生成 `.config`，把已有配置覆盖掉。
> 想要自由裁剪软件包，用手动构建（方式 B）。

### 手动构建

构建依赖见上方 [Building](#building) 一节，Debian/Ubuntu 上：

```bash
sudo apt install build-essential libncurses5-dev gawk git libssl-dev gettext \
  zlib1g-dev swig unzip time rsync python3 python3-setuptools python3-yaml
```

```bash
# 1. 取本分支（er2）
git clone -b er2 https://github.com/zt406600/wlan-ap.git wlan-ap-er2
cd wlan-ap-er2

# 2. 拉 OpenWrt 源码（config.yml 里固定到 openwrt-25.12 @ 3f081e25，默认走 gh-proxy 镜像）
#    + 应用 patches-25.12（ER2 板级支持、NSS/PPE/ECM、fullcone、默认密码、extra feeds…）
#    + 把本仓库的 profiles/ 链接进 openwrt/
./setup.py --setup

# 3. 国内网络建议先准备一个"只对本终端生效"的 git 镜像，
#    否则下一步扫描 feed 时可能在 feeds/smpackage/dockerd 上卡住（详见下面的注意事项）
printf '[url "https://gh-proxy.org/https://github.com/"]\n\tinsteadOf = https://github.com/\n' > /tmp/git-mirror

# 4. 准备 feed：gen_config.py 按 profile 生成 feeds.conf（含 qca feed 与 ipq53xx target）、
#    克隆各 feed、生成索引；再用 install -a 把 feed 里的包挂到 package/feeds，
#    否则第 5 步里 .config 中的 feed 包会因"找不到包"被静默丢掉
cd openwrt
GIT_CONFIG_GLOBAL=/tmp/git-mirror ./scripts/gen_config.py jdcloud_er2
GIT_CONFIG_GLOBAL=/tmp/git-mirror ./scripts/feeds install -a

# 5. 用仓库自带的完整配置替换上一步生成的最小系统配置，并归一化
cp ../defconfig/jdc-er2.config .config
GIT_CONFIG_GLOBAL=/tmp/git-mirror make defconfig

# 6. 编译（-j 后接并行任务数）
make -j$(nproc) V=s
```

`defconfig/jdc-er2.config` 是一份可直接使用的 `.config`，其中包含（挑重点）：

- **完整 LuCI 网页界面**：`luci`、`luci-ssl`、`luci-mod-admin-full`、`luci-mod-{network,status,system}`、
  `luci-app-firewall`、`luci-app-package-manager`、`luci-app-upnp`、`luci-app-nss-overview`、
  `luci-proto-{ppp,ipv6,relay}`、`luci-theme-argon` + `luci-theme-bootstrap`、简体中文语言包；
- **QCA 硬件加速**：`kmod-qca-nss-dp-qca`、`kmod-qca-nss-phy`、`kmod-qca-nss-ppe`
  (+`-bridge-mgr`/`-vlan-mgr`/`-netlink`/`-pppoe-mgr`/`-ds`/`-rule`/`-vp`)、`kmod-qca-nss-sfe`、
  `kmod-qca-nss-ecm-standard`、`kmod-qca-ssdk-qca-nohnat`、`qca-ssdk-shell`、`kmod-qca8084`、
  `kmod-qca8k-cc`、`kmod-phylink`、`kmod-phy-aquantia`；
- **常用工具/子系统**：`curl`、`wget-ssl`、`jq`、`htop`、`tcpdump`、`ethtool-full`、`iperf3`、
  `smartmontools`、`lm-sensors`、`nvme-cli`、`mmc-utils`、`parted`/`e2fsprogs`/`f2fs` 系列、
  `wireguard` + `kmod-tcp-bbr`、`kmod-nft-fullcone`（fullcone NAT）等。

### 增删软件包

**编译前加进固件**（推荐）：

```bash
cd openwrt

# 1) 要加的包如果来自 feed 但还没挂进 package/feeds，先在菜单里看不到/选不上，需要先 install
./scripts/feeds update -a
./scripts/feeds install luci-app-ddns          # 换成你要的包名

# 2) 交互式勾选：LuCI 相关在 LuCI 菜单，其它在 Network / Utilities / Kernel modules 等
make menuconfig

# 3) 归一化配置后重新编译
make defconfig
make -j$(nproc) V=s
```

不开 menuconfig（脚本化/无人值守）也可以，直接改 `.config` 再归一化，效果一样：

```bash
echo 'CONFIG_PACKAGE_luci-app-ddns=y' >> .config    # 关掉则写 '# CONFIG_PACKAGE_xxx is not set'
make defconfig
make -j$(nproc) V=s
```

如果包不在任何 feed 里（例如本仓库不含源码的 `luci-app-mini-diskmanager`、`luci-theme-aurora`），
把源码 clone 到 `openwrt/package/` 后，菜单里就会出现对应条目：

```bash
cd openwrt/package
git clone --depth 1 https://github.com/4IceG/luci-app-mini-diskmanager
git clone --depth 1 https://github.com/eamonxg/luci-theme-aurora
cd .. && make menuconfig
```

**编译后给已刷机的设备加包**（不重编整机）：

```bash
make package/luci-app-ddns/compile     # 只编这一个包，依赖会一起编
```

产物在 `openwrt/bin/packages/<arch>/<feed>/`（ER2 的 arch 是 `aarch64_cortex-a53`，
feed 目录为 `base`/`luci`/`packages`/`qca`/`routing`/`smpackage`/`telephony`/`video`）。
本固件用的是 OpenWrt 25.12 的 `apk` 包管理（`CONFIG_USE_APK=y`，安装包后缀是 `*.apk`），
拷到设备上 `apk add --allow-untrusted xxx.apk` 安装即可；内核模块在
`openwrt/bin/targets/ipq53xx/generic/packages/`。也可以在 LuCI 的
`系统 → 软件包`（`luci-app-package-manager`）里直接上传安装。

**保存自己的配置**：调好后存回仓库，下次直接复用：

```bash
cd openwrt && cp .config ../defconfig/jdc-er2.config
```

### 产物与刷机

产物在 `openwrt/bin/targets/ipq53xx/generic/`：
`openwrt-ipq53xx-jdcloud_er2-squashfs-sysupgrade.bin`（刷机用）、`…-squashfs-factory.bin`、
`…-initramfs-kernel.bin`（救援/救砖用），以及 `config.buildinfo`、`feeds.buildinfo`、
`…manifest`、`sha256sums`（`feeds.buildinfo` 记录了本次编译实际用到的 feed revision）。

刷完后网线接 **LAN 口**（`eth0` 与下面交换口 2/3/4 是 LAN、1 是 WAN，见
`feeds/qca-wifi-7/ipq53xx/base-files/etc/board.d/02_network`），浏览器打开 `http://192.168.1.1`，
用户名 `root`，默认密码 **`123456`**（`patches-25.12/0014-*`，刷完请第一时间改掉）。

默认网络配置由 `feeds/qca-wifi-7/ipq53xx/base-files/etc/uci-defaults/99-jdcloud-er2-network`
在首次启动时写入：`lan` 是桥 `br-lan`（`eth0` + 交换 VLAN 2 的 CPU 口 `eth1.2`）静态
`192.168.1.1/24`，`wan` 是 `eth1.1` 走 DHCP（`wan6` 是 `eth1.1` 上的 DHCPv6）。
它不是在 `02_network` 里“顺带”生成的：`patches-25.12/0021-*` 注释掉了 `/bin/config_generate`
里 `generate_network()` 的调用循环，`02_network` 只写 `/etc/board.json`，接口得在这里补。

### 注意事项

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
#    再按下一节把该 feed 带来的符号在 menuconfig 里关掉
```

### 关于第三方 feed `smpackage`

`patches-25.12/0127-feeds-add-smpackage-feed.patch` 会加一个指向
`https://github.com/kenzok8/small-package` 的 `src-git smpackage` feed（**没有锁定 commit**）。
`.config` 里 `luci-theme-argon`、`autocore-arm`、`kmod-nft-fullcone`（fullcone 的内核模块，
来自该 feed 的 `fullconenat-nft`）、`luci-app-daede_daed`、`luci-app-ssr-plus_*` 这些符号都来自它，
上游更新后编出来的版本会跟着变。要固定就自己 pin 一个 commit：

```bash
sed -i 's#kenzok8/small-package$#kenzok8/small-package^<commit>#' openwrt/feeds.conf.default
./scripts/feeds update smpackage
```

不需要这个 feed 就去掉（顺手在 menuconfig 里把上面那几个符号关掉）：

```bash
sed -i '/small-package/d' openwrt/feeds.conf.default
rm -rf openwrt/feeds/smpackage openwrt/feeds/smpackage.tmp
```

`defconfig/jdc-er2.config` 里还有几个包本仓库不含源码（`luci-app-wolultra`、
`luci-app-mini-diskmanager`、`luci-theme-aurora`），`make defconfig` 会因找不到对应符号
自动忽略这几行，不影响编译；需要就把源码放进 `openwrt/package/`（见「增删软件包」）。

### 用方式 A（`./build.sh`）加包

方式 A 的包集合就是 `profiles/jdcloud_er2.yml` 里的 `packages:`，想加包往这里加，例如基础 LuCI：

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

改完重新跑一次 `./build.sh jdcloud_er2`（它会重新生成 `.config`），再 `make -j$(nproc)` 编译。

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
