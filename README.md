# JDCloud ER2(re-cs-08) openwrt

本仓库基于[Telecominfraproject/wlan-ap](https://github.com/Telecominfraproject/wlan-ap)适配   `JDCloud ER2(re-cs-08) ` 。

## 固件构建

### 环境搭建

需要 `linux` 环境，python ≥ 3.7， 或参见openwrt官方的构建环境引导页面:<br>
https://openwrt.org/docs/guide-developer/toolchain/beginners-build-guide

推荐使用 Debian/Ubuntu，在终端执行以下命令构建基础编译环境:
```
sudo apt install build-essential libncurses5-dev gawk git libssl-dev gettext zlib1g-dev swig unzip time rsync python3 python3-setuptools python3-yaml
```

### 本地构建

使用 `./build.sh <target>`全自动构建, 或按下面步骤手动构建(推荐):

#### 1. 克隆仓库：
* main 分支为上游sync fork分支（不要克隆）
* qca-nss-14.0 为基于qsdk 14.0的适配分支
*  er2 为紧跟上游的默认适配分支
*  **按需选择 `qca-nss-14.0` 或 `er2` 分支即可。**
```shell
git clone -b qca-nss-14.0 --single-branch https://github.com/zt406600/wlan-ap.git
```
#### 2. 初始化源码树，会自动创建 `openwrt/` 目录。
```shell
cd wlan-ap
./setup.py --setup    # 对于后续构建，请使用 --rebase 替代
```

#### 3. 根据配置文件初始化，将会安装feeds和其他包并生成 `.config` 文件。
* 你可以根据自己需求精心修改 `profiles/jdcloud_er2.yml` 文件，并一键生成 `.config` 文件。
* **注意！重复执行会覆盖已存在的.config文件。**
```shell
cd openwrt
./scripts/gen_config.py jdcloud_er2
```
* **上面命令执行完后，还可以使用 `make menuconfig` 按需求手选需要的包。**
* **或使用 `cp ../defconfig/jdc-er2.config .config` ，在预置 `.config` 文件基础上增删需要的包。**

#### 4. 执行编译命令，生成固件。
```shell
make -j$(nproc) download V=s  # 预下载相关源码包
make -j$(nproc) V=s
```
#### 5. 生成的固件在 `openwrt/bin/targets/ipq53xx/generic/` 目录下。


## 其他说明

### 关于luci-theme-argon主题
[luci-theme-argon](https://github.com/jerrykuku/luci-theme-argon) `v3.1.2`（`7b77b44`）  被放在了feeds/extras目录，并在使用 `src-link extras ../../feeds/extras` 引用。

### fullcone（SONiC 全锥形 NAT）
本仓库集成了[openwrt-sonic-fullcone](https://github.com/mufeng05/openwrt-sonic-fullcone)补丁，
**不要再装 `kmod-nft-fullcone`**（默认配置里已关掉）：<br>
它和 SONiC 的 `fullcone` 表达式同名，同时存在会互相抢占表达式注册，`nft_masq.ko`
甚至可能加载失败（连带 masquerade 一起挂掉）。<br>
fullcone 默认不开（ `menuconfig` 里也不需要额外勾选），刷机后按需在防火墙配置页面打开。


