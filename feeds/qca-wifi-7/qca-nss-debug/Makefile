include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

# The NSS debug infrastructure (nss_log() and the per-module debug level /
# category controls) lives in the nss-ppe source tree but is built as its own
# module: the PPE core driver includes <nss_debug.h> and calls
# nss_debug_enable_get() on every packet, so the debug module has to be loaded
# before it.
#
# QSDK 14.0 builds it from the same source tree (qca/src/nss/base/qca-nss-ppe/
# nss_debug) as a separate feed package; here the tree is fetched from the
# upstream repository and only that subdirectory is compiled.
PKG_NAME:=qca-nss-debug
PKG_SOURCE_PROTO:=git
PKG_BRANCH:=master
PKG_RELEASE:=1
PKG_SOURCE_URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-ppe
PKG_MIRROR_HASH:=9dc2620b59952d1d7146169b9824fba8431fc7b7e63f37a0a3bf285182f8c45c
PKG_SOURCE_VERSION:=03a79c9564c62db1ad22615a8fb3007de37c41f1
PKG_VERSION:=1.0

include $(INCLUDE_DIR)/package.mk

ifeq ($(CONFIG_TARGET_ipq),y)
subtarget:=$(SUBTARGET)
else
subtarget:=$(CONFIG_TARGET_BOARD)
endif

define KernelPackage/qca-nss-debug
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for the NSS debug infrastructure
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx
  FILES:=$(PKG_BUILD_DIR)/nss_debug/qca-nss-debug.ko
  AUTOLOAD:=$(call AutoLoad,30,qca-nss-debug)
endef

define KernelPackage/qca-nss-debug/Description
   Kernel module providing nss_log() and the per-module debug level and
   category controls used by the NSS/PPE drivers.
endef

define Build/InstallDev/qca-nss-debug
	$(INSTALL_DIR) $(1)/usr/include/qca-nss-debug
	$(CP) $(PKG_BUILD_DIR)/nss_debug/exports/* $(1)/usr/include/qca-nss-debug/
endef

define Build/InstallDev
	$(call Build/InstallDev/qca-nss-debug,$(1))
endef

define Build/Compile
	$(KERNEL_MAKE) $(strip $(MAKE_OPTS)) \
		M="$(PKG_BUILD_DIR)" \
		EXTRA_CFLAGS="$(EXTRA_CFLAGS)" \
		SoC="$(subtarget)" \
		modules
endef

$(eval $(call KernelPackage,qca-nss-debug))
