include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=qca-nss-ppe-rule
PKG_SOURCE_PROTO:=git
PKG_BRANCH:=master
PKG_RELEASE:=3
PKG_SOURCE_URL:=https://git.codelinaro.org/clo/qsdk/oss/lklm/nss-ppe
PKG_MIRROR_HASH:=cef37e2661f27bc4726e27808c9cdbb5cc4f709680e625358da5fdaff7036c32
PKG_SOURCE_VERSION:=5efb960b0872d57e39426dac538d7056e63b26c5
PKG_VERSION:=1.0

include $(INCLUDE_DIR)/package.mk

ifeq ($(CONFIG_TARGET_ipq),y)
subtarget:=$(SUBTARGET)
else
subtarget:=$(CONFIG_TARGET_BOARD)
endif

define KernelPackage/qca-nss-ppe-rule
  SECTION:=kernel
  CATEGORY:=Kernel modules
  SUBMENU:=Network Devices
  TITLE:=Kernel driver for PPE rule module
  DEPENDS:=@TARGET_ipq95xx||TARGET_ipq53xx||TARGET_ipq54xx||TARGET_ipq96xx||TARGET_ipq52xx \
		+kmod-qca-nss-ppe +kmod-qca-nss-ppe-vp
  FILES:=$(PKG_BUILD_DIR)/drv/ppe_rule/qca-nss-ppe-rule.ko
  AUTOLOAD:=$(call AutoLoad,52,qca-nss-ppe-rule)
endef

define KernelPackage/qca-nss-ppe-rule/Description
   Kernel module for PPE rule driver
endef

define KernelPackage/qca-nss-ppe-rule/install
	$(INSTALL_DIR) $(1)/usr/bin
	$(INSTALL_BIN) ./files/acl_dump.sh $(1)/usr/bin/
	$(INSTALL_BIN) ./files/mcast_hammer.sh $(1)/usr/bin/
	$(INSTALL_BIN) ./files/json_mcast_hammer.sh $(1)/usr/bin/
endef

define Build/InstallDev/qca-nss-ppe-rule
	$(INSTALL_DIR) $(1)/usr/include/qca-nss-ppe
	$(CP) $(PKG_BUILD_DIR)/drv/exports/* $(1)/usr/include/qca-nss-ppe/
endef

define Build/InstallDev
	$(call Build/InstallDev/qca-nss-ppe-rule,$(1))
endef

# ppe_rule.c and ppe_acl.c include the PPE core headers (<ppe_drv.h>,
# <ppe_acl.h>, <ppe_drv_public.h>), which qca-nss-ppe only installs into staging
# during its own install step; use this package's own copy to stay clear of
# that ordering, exactly as qca-nss-ppe-ds does.
EXTRA_CFLAGS+= \
	-I$(PKG_BUILD_DIR)/exports \
	-I$(PKG_BUILD_DIR)/drv/exports \
	-I$(STAGING_DIR)/usr/include/qca-ssdk \
	-I$(STAGING_DIR)/usr/include/qca-ssdk/fal \
	-I$(STAGING_DIR)/usr/include/qca-ssdk/init \

ifeq ($(CONFIG_LOWMEM_FLASH),y)
MAKE_OPTS+=PPE_LOWMEM_PROFILE_16M=y
EXTRA_CFLAGS+= -DPPE_LOWMEM_PROFILE_16M
endif

ifeq ($(CONFIG_KERNEL_IPQ_MEM_PROFILE),256)
MAKE_OPTS+=PPE_LOWMEM_PROFILE_256M=y
EXTRA_CFLAGS+= -DPPE_MEM_PROFILE_LOW_256M
endif

MAKE_OPTS+=ppe-rule=y
MAKE_OPTS+=PPE_RFS_ENABLED=y

ifneq ($(CONFIG_LOWMEM_FLASH),y)
MAKE_OPTS+=PPE_ACL_ENABLED=y
MAKE_OPTS+=PPE_POLICER_ENABLED=y
MAKE_OPTS+=PPE_QOS_ENABLED=y
endif

ifneq ($(CONFIG_KERNEL_IPQ_MEM_PROFILE),256)
ifneq ($(CONFIG_LOWMEM_FLASH),y)
MAKE_OPTS+=PPE_PRIORITY_ENABLED=y
endif
endif

ifneq ($(CONFIG_TARGET_ipq53xx),)
MAKE_OPTS+=PPE_RULE_IPQ53XX=y
endif

ifneq ($(CONFIG_TARGET_ipq54xx),)
MAKE_OPTS+=PPE_RULE_IPQ54XX=y
endif

ifneq ($(CONFIG_KERNEL_IPQ_MEM_PROFILE),256)
ifneq ($(CONFIG_LOWMEM_FLASH),y)
MAKE_OPTS+=PPE_MIRROR_ENABLED=y
endif
endif

define Build/Compile
	$(KERNEL_MAKE) $(strip $(MAKE_OPTS)) \
		M="$(PKG_BUILD_DIR)" \
		EXTRA_CFLAGS="$(EXTRA_CFLAGS)" \
		SoC="$(subtarget)" \
		modules
endef

$(eval $(call KernelPackage,qca-nss-ppe-rule))
