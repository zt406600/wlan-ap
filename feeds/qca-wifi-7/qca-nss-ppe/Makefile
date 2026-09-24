include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=qca-nss-ppe
PKG_SOURCE_PROTO:=git
PKG_BRANCH:=master
PKG_RELEASE:=4
PKG_SOURCE_URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-ppe
PKG_MIRROR_HASH:=b338f378ce60e3732a01a6bcaa6cb707b5ee662ab971bf34ea568f7b300cc60a
PKG_SOURCE_VERSION:=5efb960b0872d57e39426dac538d7056e63b26c5
PKG_VERSION:=1.0

# r44 (5efb960b) upstream already carries the __maybe_unused markers that the
# local 0001/0002 compilation-fix patches added (drv/ppe_drv/tun/ppe_drv_tun.c
# and clients/pppoe/frontends/ppe/ppe_pppoe_mgr.c), so both patches were
# dropped.  0003/0004 still apply to r44 unchanged.

# Make the build dir (and hence the module set) depend on the client package
# selections: OpenWrt encodes PKG_CONFIG_DEPENDS into the .configured stamp, so
# ticking a new qca-nss-ppe client in menuconfig forces a re-prepare/recompile
# instead of reusing the previously built module set.
PKG_CONFIG_DEPENDS:= \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-tun \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-qdisc \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-pppoe-mgr \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-lag-mgr \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-vxlanmgr \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-mapt \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-tunipip6 \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-gre-mgr \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-l2tp-mgr \
	CONFIG_PACKAGE_kmod-qca-nss-ppe-netlink

include $(INCLUDE_DIR)/package.mk

ifeq ($(CONFIG_TARGET_ipq),y)
subtarget:=$(SUBTARGET)
else
subtarget:=$(CONFIG_TARGET_BOARD)
endif

define KernelPackage/qca-nss-ppe
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE (core driver)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx||TARGET_ipq96xx||TARGET_ipq52xx\
		+kmod-qca-ssdk-qca-nohnat \
		+kmod-nat46-qca
  FILES:=$(PKG_BUILD_DIR)/drv/ppe_drv/qca-nss-ppe.ko
  AUTOLOAD:=$(call AutoLoad,31,qca-nss-ppe)
endef

define KernelPackage/qca-nss-ppe/Description
Kernel module for PPE core driver
endef

define KernelPackage/qca-nss-ppe/install
        $(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) ./files/ppe_flow_dump $(1)/usr/bin/
	$(INSTALL_BIN) ./files/ppe_if_map $(1)/usr/bin/
	$(INSTALL_BIN) ./files/nss_perf_config.sh $(1)/lib/
endef

# Some of the clients below need a module that is built by a *different* source
# package (drv/ppe_vp -> qca-nss-ppe-vp, drv/ppe_rule -> qca-nss-ppe-rule and
# qca-nss-dp.ko -> qca-nss-dp-qca).  Those packages depend back on this one
# (+kmod-qca-nss-ppe) because they consume the headers and symbol versions that
# this package installs, hence writing the requirement as a "+kmod-..." package
# dependency makes scripts/package-metadata.pl emit the dependency edge in both
# directions:
#
#     qca-nss-ppe/compile      -> qca-nss-ppe-vp/compile   (from the +kmod- here)
#     qca-nss-ppe-vp/compile   -> qca-nss-ppe/compile      (from its own DEPENDS)
#
# GNU make resolves such a cycle by silently dropping one of the two edges, so
# the client may be compiled before the module that exports the symbols it links
# against.  Since KBUILD_EXTRA_SYMBOLS is built from the symvers files that
# already exist (include/kernel.mk), a clean build_dir then fails in modpost
# with "undefined!" errors for ppe_drv_*/nss_dp_vp_*.
#
# Use the config dependency form (@PACKAGE_...) for those modules instead: it
# keeps the requirement (the client can only be selected once the provider
# module is enabled) but does not create a compile edge, so the build order
# stays acyclic:
#
#     qca-nss-ppe -> {nat46-qca,qca-ssdk-qca} -> qca-nss-dp-qca
#                  -> {qca-nss-ppe-vp,qca-nss-ppe-rule} -> qca-nss-ppe-ds
#
# Only the direct child modules of drv/ are built by the packages below; the
# same-source clients (vlan-mgr, bridge-mgr, tun, ...) stay as "+kmod-..."
# dependencies because same-source dependencies are filtered out by
# package-metadata.pl and therefore cannot close a cycle.
#
# Dropping the "+kmod-..." prefix also removes the provider from the package
# metadata *and* from the packaging time dependency check of
# include/package-pack.mk (CheckDependencies): that check extracts the
# "depends=" entries modpost records in the .modinfo of each .ko of a package
# (scripts/gen-dependencies.sh) and looks them up in the .provides list, which is
# only filled from the (filtered) DEPENDS list.  As soon as the provider happens
# to be built first, so that its .symvers exists and is picked up through
# KBUILD_EXTRA_SYMBOLS, packaging fails with
#
#     Package kmod-qca-nss-ppe-vlan-mgr is missing dependencies for the
#     following libraries:
#     qca-nss-ppe-vp.ko
#
# while the very same build passes when the provider is compiled afterwards,
# i.e. the result depends on the module build order.
#
# Restore both halves of the requirement in the package metadata without putting
# it back into the build dependency graph:
#
#   EXTRA_DEPENDS  -> "Depends:" entry of the package.  package-pack.mk merges it
#                     into Package/<name>/DEPENDS only, it is never parsed by
#                     scripts/package-metadata.pl, hence there is no compile edge.
#   extra_provides -> the provider module files are added to the .provides list of
#                     the package (the same hook package/kernel uses for built-in
#                     modules), which is what CheckDependencies compares against.
#
# Every "@PACKAGE_kmod-..." requirement below therefore has a matching
# "EXTRA_DEPENDS += ..." and "Package/kmod-.../extra_provides" entry.  The
# leading comma is required because EXTRA_DEPENDS is a comma separated list whose
# first element ("kernel (=...)") is set by include/kernel.mk.  The version
# constraint is intentionally left open: kmods are already pinned to the same
# kernel ABI through that kernel dependency, symbols are resolved by name at load
# time (CONFIG_MODVERSIONS is off) and the AUTOLOAD priorities above guarantee
# that the providing modules are loaded first.
define KernelPackage/qca-nss-ppe-vlan-mgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE VLAN manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe \
	   @PACKAGE_kmod-qca-nss-ppe-vp @PACKAGE_kmod-qca-nss-dp-qca
  EXTRA_DEPENDS+=, kmod-qca-nss-ppe-vp (>=0), kmod-qca-nss-dp-qca (>=0)
  FILES:=$(PKG_BUILD_DIR)/clients/vlan/qca-nss-ppe-vlan.ko
  AUTOLOAD:=$(call AutoLoad,53,qca-nss-ppe-vlan)
endef

define KernelPackage/qca-nss-ppe-vlan-mgr/Description
   Kernel module for the PPE VLAN manager.
   Allocates a PPE interface and a PPE virtual port for every 802.1Q (VLAN)
   netdev and for the VLAN-as-VP case, so that VLAN traffic can be hardware
   accelerated by the PPE instead of being punted to the CPU.
endef

# The VLAN client links against the PPE virtual port API (ppe_vp_alloc()/
# ppe_vp_free() exported by qca-nss-ppe-vp) and the qca-nss-dp virtual port
# helpers; see the comment block above for why their module files have to be
# declared here as well.
define Package/kmod-qca-nss-ppe-vlan-mgr/extra_provides
	echo qca-nss-ppe-vp.ko; echo qca-nss-dp.ko
endef

define KernelPackage/qca-nss-ppe-bridge-mgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE bridge manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-vlan-mgr \
	   @PACKAGE_kmod-qca-nss-dp-qca
  EXTRA_DEPENDS+=, kmod-qca-nss-dp-qca (>=0)
  FILES:=$(PKG_BUILD_DIR)/clients/bridge/qca-nss-ppe-bridge-mgr.ko
  AUTOLOAD:=$(call AutoLoad,54,qca-nss-ppe-bridge-mgr)
endef

define KernelPackage/qca-nss-ppe-bridge-mgr/Description
   Kernel module for the PPE bridge manager.
   Registers a PPE bridge interface for every Linux bridge (and joins/leaves
   its members), so that bridged L2 traffic can be hardware accelerated.
   Loads after qca-nss-ppe-vlan, whose join/leave helpers it calls.
endef

# The bridge client needs the qca-nss-dp virtual port helpers, see the comment
# block above for why the module file has to be declared here as well.
define Package/kmod-qca-nss-ppe-bridge-mgr/extra_provides
	echo qca-nss-dp.ko
endef

define KernelPackage/qca-nss-ppe-tun
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE tunnel (core driver)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe \
	   @PACKAGE_kmod-qca-nss-ppe-vp @PACKAGE_kmod-qca-nss-ppe-rule
  EXTRA_DEPENDS+=, kmod-qca-nss-ppe-vp (>=0), kmod-qca-nss-ppe-rule (>=0)
  FILES:=$(PKG_BUILD_DIR)/drv/ppe_tun/qca-nss-ppe-tun.ko
  AUTOLOAD:=$(call AutoLoad,52,qca-nss-ppe-tun)
endef

define KernelPackage/qca-nss-ppe-tun/Description
   Kernel module for the PPE tunnel abstraction (drv/ppe_tun).
   Provides the tunnel netdev/virtual-port plumbing consumed by the
   VXLAN / MAP-T / TUNIPIP6 / GRE / L2TP managers below.
endef

# The tunnel plumbing calls into the PPE virtual port API (ppe_vp_alloc(),
# ppe_vp_free(), ppe_vp_mtu_*) of qca-nss-ppe-vp and the rule API
# (ppe_acl_rule_*) of qca-nss-ppe-rule, see the comment block above for why the
# module files have to be declared here as well.
define Package/kmod-qca-nss-ppe-tun/extra_provides
	echo qca-nss-ppe-vp.ko; echo qca-nss-ppe-rule.ko
endef

define KernelPackage/qca-nss-ppe-qdisc
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE queue discipline
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe
  FILES:=$(PKG_BUILD_DIR)/drv/ppe_qdisc/qca-nss-ppe-qdisc.ko
  AUTOLOAD:=$(call AutoLoad,51,qca-nss-ppe-qdisc)
endef

define KernelPackage/qca-nss-ppe-qdisc/Description
   Kernel module for the PPE queue discipline (shaping/scheduling offload).
endef

define KernelPackage/qca-nss-ppe-pppoe-mgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE PPPoE manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-ppp +kmod-pppoe +kmod-bonding
  FILES:=$(PKG_BUILD_DIR)/clients/pppoe/qca-nss-ppe-pppoe-mgr.ko
  AUTOLOAD:=$(call AutoLoad,55,qca-nss-ppe-pppoe-mgr)
endef

define KernelPackage/qca-nss-ppe-pppoe-mgr/Description
   Kernel module for the PPE PPPoE manager.
   Creates a PPE interface / virtual port for every PPPoE session so that
   PPPoE WAN traffic can be accelerated by the PPE instead of being punted
   to the CPU (needs CONFIG_PPP + CONFIG_PPPOE in the kernel config).
endef

define KernelPackage/qca-nss-ppe-lag-mgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE LAG manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-vlan-mgr
  FILES:=$(PKG_BUILD_DIR)/clients/lag/qca-nss-ppe-lag.ko
  AUTOLOAD:=$(call AutoLoad,55,qca-nss-ppe-lag)
endef

define KernelPackage/qca-nss-ppe-lag-mgr/Description
   Kernel module for the PPE link aggregation (bonding) manager.
   Accelerates LAG/bonding member traffic. Builds with -DBONDING_SUPPORT
   when CONFIG_BONDING is enabled in the kernel config.
endef

define KernelPackage/qca-nss-ppe-vxlanmgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE VXLAN manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-tun +kmod-qca-nss-ppe-bridge-mgr \
	   +kmod-vxlan
  FILES:=$(PKG_BUILD_DIR)/clients/vxlanmgr/qca-nss-ppe-vxlanmgr.ko
  AUTOLOAD:=$(call AutoLoad,56,qca-nss-ppe-vxlanmgr)
endef

define KernelPackage/qca-nss-ppe-vxlanmgr/Description
   Kernel module for the PPE VXLAN manager.
   Offloads VXLAN (and VXLAN-GPE) tunnel decap/encap to the PPE
   (needs CONFIG_VXLAN in the kernel config).
endef

define KernelPackage/qca-nss-ppe-mapt
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE MAP-T client
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-tun +kmod-nat46-qca
  FILES:=$(PKG_BUILD_DIR)/clients/mapt/qca-nss-ppe-mapt.ko
  AUTOLOAD:=$(call AutoLoad,56,qca-nss-ppe-mapt)
endef

define KernelPackage/qca-nss-ppe-mapt/Description
   Kernel module for the PPE MAP-T client (RFC 7599).
endef

define KernelPackage/qca-nss-ppe-tunipip6
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE TUNIPIP6 manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-tun +kmod-sit +kmod-ip6-tunnel
  FILES:=$(PKG_BUILD_DIR)/clients/tunipip6/qca-nss-ppe-tunipip6.ko
  AUTOLOAD:=$(call AutoLoad,56,qca-nss-ppe-tunipip6)
endef

define KernelPackage/qca-nss-ppe-tunipip6/Description
   Kernel module for the PPE TUNIPIP6 manager (6rd / DS-Lite style tunnels).
   Requires CONFIG_IPV6_SIT in the kernel config.
endef

define KernelPackage/qca-nss-ppe-gre-mgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE GRE manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-tun +kmod-gre +kmod-gre6
  FILES:=$(PKG_BUILD_DIR)/clients/gre/qca-nss-ppe-gre.ko
  AUTOLOAD:=$(call AutoLoad,56,qca-nss-ppe-gre)
endef

define KernelPackage/qca-nss-ppe-gre-mgr/Description
   Kernel module for the PPE GRE manager.
   Requires CONFIG_NET_IPGRE_DEMUX / CONFIG_NET_IPGRE in the kernel config.
endef

define KernelPackage/qca-nss-ppe-l2tp-mgr
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE L2TP manager (client)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe +kmod-qca-nss-ppe-tun +kmod-ppp +kmod-l2tp +kmod-pppol2tp
  FILES:=$(PKG_BUILD_DIR)/clients/l2tp/qca-nss-ppe-l2tp.ko
  AUTOLOAD:=$(call AutoLoad,56,qca-nss-ppe-l2tp)
endef

define KernelPackage/qca-nss-ppe-l2tp-mgr/Description
   Kernel module for the PPE L2TP manager.
   Requires CONFIG_L2TP in the kernel config.
endef

define KernelPackage/qca-nss-ppe-netlink
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE netlink (ACL/policer/QoS/exception)
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx \
	   +kmod-qca-nss-ppe @PACKAGE_kmod-qca-nss-ppe-rule
  EXTRA_DEPENDS+=, kmod-qca-nss-ppe-rule (>=0)
  FILES:=$(PKG_BUILD_DIR)/netlink/qca-nss-ppe-netlink.ko
  AUTOLOAD:=$(call AutoLoad,57,qca-nss-ppe-netlink)
endef

define KernelPackage/qca-nss-ppe-netlink/Description
   Kernel module providing the PPE netlink control interface for ACL,
   policer, QoS and exception-rule programming.
endef

# The netlink client programs the rules that are owned by qca-nss-ppe-rule
# (ppe_acl_rule_*, ppe_policer_*, ppe_qos_*), see the comment block above for why
# the module file has to be declared here as well.
define Package/kmod-qca-nss-ppe-netlink/extra_provides
	echo qca-nss-ppe-rule.ko
endef

define Build/InstallDev/qca-nss-ppe
	$(INSTALL_DIR) $(1)/usr/include/qca-nss-ppe
	$(CP) $(PKG_BUILD_DIR)/exports/* $(1)/usr/include/qca-nss-ppe/
	$(CP) $(PKG_BUILD_DIR)/drv/exports/* $(1)/usr/include/qca-nss-ppe/
endef

define Build/InstallDev
	$(call Build/InstallDev/qca-nss-ppe,$(1))
endef

ifdef CONFIG_TARGET_PROFILE
TARGET_PROFILE=$(subst ",,$(CONFIG_TARGET_PROFILE))
PATCH_PROFILE_NAME=patches-$(subst DEVICE_,,$(TARGET_PROFILE))
FILES_PROFILE_NAME=files-$(subst DEVICE_,,$(TARGET_PROFILE))
endif

# -I$(PKG_BUILD_DIR)/drv/exports:
#	netlink/include/nss_ppenl_acl_if.h includes <ppe_acl.h> (and the
#	policer/QoS/priority headers), which live in the source tree's
#	drv/exports/ directory.  Reference the build dir directly so a fresh
#	build does not depend on the headers having been staged by an earlier
#	install of the qca-nss-ppe-rule package.
# -I$(LINUX_DIR)/net/l2tp:
#	clients/l2tp/nss_ppe_l2tp.c includes the kernel private header
#	<l2tp_core.h>, which a kernel built with CONFIG_L2TP=m keeps in
#	net/l2tp/ (it is not part of the exported/uapi header set).  The
#	symbols it uses (l2tp_tunnel_get(), l2tp_tunnel_get_session(),
#	l2tp_tunnel_dec_refcount(), l2tp_session_dec_refcount(),
#	l2tp_stats_update()) are all EXPORT_SYMBOL_GPL.
EXTRA_CFLAGS+= \
	-I$(STAGING_DIR)/usr/include/qca-ssdk \
	-I$(STAGING_DIR)/usr/include/qca-ssdk/fal \
	-I$(STAGING_DIR)/usr/include/qca-ssdk/init \
	-I$(STAGING_DIR)/usr/include/nat46 \
	-I$(PKG_BUILD_DIR)/drv/exports \
	-I$(LINUX_DIR)/net/l2tp

MAKE_OPTS+=ppe-drv=y
MAKE_OPTS+=vlan-mgr=y
MAKE_OPTS+=bridge-mgr=y

# The clients built here reference symbols exported by sibling packages that are
# built from this very source tree (clients/vlan -> ppe_vp_alloc()/ppe_vp_free()
# from qca-nss-ppe-vp, netlink/ -> ppe_acl_*/ppe_policer_*/ppe_qos_* from
# qca-nss-ppe-rule).  Those packages depend back on this package for the headers
# and symbol versions it installs, so on a clean build_dir their symvers files do
# not exist yet when this package is compiled and modpost fails with
# "undefined!" errors (KBUILD_EXTRA_SYMBOLS is only the symvers files that have
# already been created).  In the QCA SDK all of it is one single module build, so
# the situation does not exist there.
#
# CONFIG_MODVERSIONS is disabled in this kernel, hence the symbol version (CRC)
# recorded by modpost has no effect at load time: symbols are resolved by name
# from the modules that are already loaded, and the AUTOLOAD priorities below
# guarantee that the providing modules load first (ppe=31, vp=52, rule=52,
# vlan=53, bridge-mgr=54, netlink=57).  Let modpost warn instead of failing so
# that a clean, single-pass build of the whole feed produces a working module
# set; the warnings stay visible in the build log.
MAKE_OPTS+=KBUILD_MODPOST_WARN=1

# The clients below are only built when their package is selected.  This keeps a
# default image compiling exactly the same module set as before, while selecting
# any of them (menuconfig -> Kernel modules -> Network Devices) pulls in the PPE
# and kernel counterparts the client needs through the package DEPENDS.
#
# The gates must accept both "y" and "m": kmod packages are tristate, so a client
# pulled in as an install dependency ends up as "y" while one ticked as "M" in
# menuconfig stays "m" in .config.  Testing for "y" alone silently skips the
# MAKE_OPTS for the "m" case, which leaves the module out of the build and makes
# the image build fail while packaging the (empty) kmod package.
ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-tun),)
MAKE_OPTS+=ppe-tun=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-qdisc),)
MAKE_OPTS+=ppe-qdisc=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-pppoe-mgr),)
MAKE_OPTS+=pppoe-mgr=y
MAKE_OPTS+=PPPOE_MGR_FE_PPE_ENABLE=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-lag-mgr),)
MAKE_OPTS+=lag-mgr=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-vxlanmgr),)
MAKE_OPTS+=vxlanmgr=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-mapt),)
MAKE_OPTS+=mapt-mgr=y
endif

# tunipip6, gre and l2tp also link against kernel symbols that only exist once
# the matching kernel feature is enabled (kmod-sit / kmod-ip6-tunnel,
# kmod-gre / kmod-gre6 and kmod-l2tp / kmod-pppol2tp, all pulled in through
# DEPENDS), hence they stay off until the package is selected.
ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-tunipip6),)
MAKE_OPTS+=tunipip6-mgr=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-gre-mgr),)
MAKE_OPTS+=gre-mgr=y
MAKE_OPTS+=GRE_MGR_FE_PPE_ENABLE=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-l2tp-mgr),)
MAKE_OPTS+=l2tp-mgr=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-netlink),)
MAKE_OPTS+=netlink=y
endif

ifeq ($(CONFIG_LOWMEM_FLASH),y)
MAKE_OPTS+=PPE_16M_PROFILE=y
endif

ifeq ($(CONFIG_KERNEL_IPQ_MEM_PROFILE),256)
MAKE_OPTS+=PPE_256M_PROFILE=y
endif

ifneq ($(CONFIG_KERNEL_IPQ_MEM_PROFILE),256)
ifneq ($(CONFIG_LOWMEM_FLASH),y)
MAKE_OPTS+=PPE_IPSEC_ENABLE=y
MAKE_OPTS+=PPE_TUN_ENABLE=y
endif
endif

ifeq ($(CONFIG_TARGET_ipq95xx_generic_QSDK_MinEnt), y)
EXTRA_CFLAGS+= -DPPE_V4_BAD_LEN_EXCEP_DIS
endif

define Build/Compile
	$(KERNEL_MAKE) $(strip $(MAKE_OPTS)) \
		M="$(PKG_BUILD_DIR)" \
		EXTRA_CFLAGS="$(EXTRA_CFLAGS)" \
		SoC="$(subtarget)" \
		modules
endef

define patch_profile
	$(call PatchDir/Default,$(PKG_BUILD_DIR),./$(PATCH_PROFILE_NAME))
endef

define files_profile

	if [ -d "./$(FILES_PROFILE_NAME)" ]; then \
		$(CP) ./$(FILES_PROFILE_NAME)/* ./files/ ; \
	fi
endef

Hooks/Prepare/Post += patch_profile
Hooks/Prepare/Post += files_profile

$(eval $(call KernelPackage,qca-nss-ppe))
$(eval $(call KernelPackage,qca-nss-ppe-vlan-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-bridge-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-tun))
$(eval $(call KernelPackage,qca-nss-ppe-qdisc))
$(eval $(call KernelPackage,qca-nss-ppe-pppoe-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-lag-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-vxlanmgr))
$(eval $(call KernelPackage,qca-nss-ppe-mapt))
$(eval $(call KernelPackage,qca-nss-ppe-tunipip6))
$(eval $(call KernelPackage,qca-nss-ppe-gre-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-l2tp-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-netlink))
