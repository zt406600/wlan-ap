# 来源与维护说明

本目录是 **vendored**（拷进来的）第三方 LuCI 主题包，不是本仓库的代码。

| 项 | 值 |
|---|---|
| 上游 | https://github.com/jerrykuku/luci-theme-argon |
| 版本 | tag `v3.1.2`（commit `7b77b44`，`release: v3.1.2`） |
| 许可证 | Apache-2.0 / MIT（见本目录 `LICENSE`） |

## 为什么放在这里

`feeds.extras` 是仓库自带的本地 feed：`patches-25.12/0213-feeds-add-extras-feed.patch`
向 `feeds.conf.default` 追加一行

```
src-link extras ../../feeds/extras
```

与现有的 `src-link qca ../../feeds/qca-wifi-7` 完全同一套路。这样版本写死在仓库里、
构建不需要联网拉第三方 feed，也不会像未 pin 的第三方 feed 那样在某天悄悄改依赖把
构建打挂（2026-10 的 smpackage 就是这么炸的，见 commit `2a421838`）。

## 从上游复制时丢弃了什么

为了让仓库体积可控，只保留构建与运行需要的部分：

- 保留：`Makefile`、`LICENSE`、`htdocs/`（主题资源）、`po/`（翻译）、`root/`（默认配置、
  uci-defaults、ucode 辅助脚本、菜单/ACL）、`ucode/`（视图）
- 丢弃：`Screenshots/`（9.8M）、`docs/`、`tests/`、`less/`（css 源文件，发布版已带编译好的
  `htdocs/luci-static/argon/css/cascade.css`）、`compat/luci-app-argon-config/`
  （v3.1.2 已把配置界面整合进主题包 `root/usr/share/luci/menu.d/luci-app-argon-config.json`）

## 怎么升级

```bash
cd /tmp && git clone --branch v<新版本> --depth 1 https://github.com/jerrykuku/luci-theme-argon.git argon
cd argon && rm -rf Screenshots docs tests less compat README* RELEASE* .git
cp -a . /path/to/wlan-ap-er2/feeds/extras/luci-theme-argon/
# 更新本文件里的版本/commit，然后
cd /path/to/wlan-ap-er2/openwrt && ./scripts/feeds update extras && ./scripts/feeds install -f luci-theme-argon
make defconfig && make package/luci-theme-argon/compile V=s
```

升级后记得把 `defconfig/jdc-er2.config` 里的 `CONFIG_PACKAGE_luci-theme-argon=y` 保留，
并确认 `luci-theme-bootstrap` 仍在（漫游/回退用）。
