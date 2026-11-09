include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

# The PPE client managers (bridge/vlan/pppoe/lag/vxlan/mapt/tunipip6/gre/l2tp)
# used to live in the nss-ppe repository next to the core driver.  Upstream
# moved them into their own repository (nss-ppe commit 7932f247, 2026-05-06),
# so from QSDK 14.0 on they are a separate source tree again.
#
# They stay in one package here (as they were in nss-ppe) because they all
# come from the same tree and share the same build options; the QSDK tree
# splits them into one feed package per client, which only makes the
# dependency bookkeeping harder.
PKG_NAME:=qca-nss-ae-clients
PKG_SOURCE_PROTO:=git
PKG_BRANCH:=master
PKG_RELEASE:=1
PKG_SOURCE_URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-ae-clients
PKG_MIRROR_HASH:=d42b82ad7e318c45899a40bf5518a3c07066faeae65ed789df3932d23e9d3631
PKG_SOURCE_VERSION:=e5336c2b08d7d51f8c2d6b94fc684025dc5992c2
PKG_VERSION:=1.0

include $(INCLUDE_DIR)/package.mk

ifeq ($(CONFIG_TARGET_ipq),y)
subtarget:=$(SUBTARGET)
else
subtarget:=$(CONFIG_TARGET_BOARD)
endif

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
   PPPoE WAN traffic can be accelerated by the PPE instead of being punted to
   the CPU (needs CONFIG_PPP + CONFIG_PPPOE in the kernel config).
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

# The clients need the PPE core headers and the qca-nss-dp virtual port
# helpers.  clients/l2tp/nss_ppe_l2tp.c includes the kernel private header
# <l2tp_core.h>, which a kernel built with CONFIG_L2TP=m keeps in net/l2tp/
# (it is not part of the exported/uapi header set).  The symbols it uses
# (l2tp_tunnel_get(), l2tp_tunnel_get_session(), l2tp_tunnel_dec_refcount(),
# l2tp_session_dec_refcount(), l2tp_stats_update()) are all
# EXPORT_SYMBOL_GPL.
EXTRA_CFLAGS+= \
	-I$(STAGING_DIR)/usr/include/qca-ssdk \
	-I$(STAGING_DIR)/usr/include/qca-ssdk/fal \
	-I$(STAGING_DIR)/usr/include/qca-ssdk/init \
	-I$(STAGING_DIR)/usr/include/nat46 \
	-I$(STAGING_DIR)/usr/include/qca-nss-ppe \
	-I$(STAGING_DIR)/usr/include/qca-nss-dp \
	-I$(LINUX_DIR)/net/l2tp

# The clients below are only built when their package is selected.  This keeps
# a default image compiling exactly the same module set as before, while
# selecting any of them (menuconfig -> Kernel modules -> Network Devices)
# pulls in the PPE and kernel counterparts the client needs through the
# package DEPENDS.
#
# The gates must accept both "y" and "m": kmod packages are tristate, so a client
# pulled in as an install dependency ends up as "y" while one ticked as "M" in
# menuconfig stays "m" in .config.  Testing for "y" alone silently skips the
# MAKE_OPTS for the "m" case, which leaves the module out of the build and makes
# the image build fail while packaging the (empty) kmod package.
ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-vlan-mgr),)
MAKE_OPTS+=vlan-mgr=y
endif

ifneq ($(CONFIG_PACKAGE_kmod-qca-nss-ppe-bridge-mgr),)
MAKE_OPTS+=bridge-mgr=y
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

# The clients reference symbols exported by sibling packages that are built
# from other source trees (ppe_vp_alloc()/ppe_vp_free() from qca-nss-ppe-vp,
# qca-nss-dp.ko from qca-nss-dp-qca).  Those packages depend back on
# qca-nss-ppe for the headers and symbol versions it installs, so a plain
# "+kmod-..." requirement would make package-metadata.pl emit the dependency
# edge in both directions and GNU make would silently drop one of them, so the
# client could be compiled before the module exporting its symbols.
#
# Use the config dependency form (@PACKAGE_...) for those instead: it keeps the
# requirement (the client can only be selected once the provider module is
# enabled) but does not create a compile edge, so the build order stays
# acyclic.  EXTRA_DEPENDS puts the requirement back into the package metadata
# and "extra_provides" puts the provider module files into the .provides list,
# which is what the packaging time dependency check compares against.
#
# CONFIG_MODVERSIONS is disabled in this kernel, hence the symbol version (CRC)
# recorded by modpost has no effect at load time: symbols are resolved by name
# from the modules that are already loaded, and the AUTOLOAD priorities above
# guarantee that the providing modules load first (ppe=31, vp=52, rule=52,
# vlan=53, bridge-mgr=54, netlink=57).
MAKE_OPTS+=KBUILD_MODPOST_WARN=1

define Build/Compile
	$(KERNEL_MAKE) $(strip $(MAKE_OPTS)) \
		M="$(PKG_BUILD_DIR)" \
		EXTRA_CFLAGS="$(EXTRA_CFLAGS)" \
		SoC="$(subtarget)" \
		modules
endef

$(eval $(call KernelPackage,qca-nss-ppe-vlan-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-bridge-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-pppoe-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-lag-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-vxlanmgr))
$(eval $(call KernelPackage,qca-nss-ppe-mapt))
$(eval $(call KernelPackage,qca-nss-ppe-tunipip6))
$(eval $(call KernelPackage,qca-nss-ppe-gre-mgr))
$(eval $(call KernelPackage,qca-nss-ppe-l2tp-mgr))
