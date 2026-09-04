include $(TOPDIR)/rules.mk

PKG_NAME:=aic8800-usb
PKG_VERSION:=5.0
PKG_RELEASE:=1
PKG_SOURCE_PROTO:=git
PKG_SOURCE_URL:=https://github.com/radxa-pkg/aic8800.git
PKG_SOURCE_VERSION:=bd11969265809a0fc948f1107c8256bbb2c1aa60
PKG_MIRROR_HASH:=9c84f790e0561f3ed3a3c1b0f23a62196a5f67784dad1215b11fb0dcabd92d19
PKG_LICENSE:=GPL-3.0-only
PKG_LICENSE_FILES:=LICENSE

include $(INCLUDE_DIR)/kernel.mk
include $(INCLUDE_DIR)/package.mk
PKG_BUILD_DIR:=$(KERNEL_BUILD_DIR)/$(PKG_NAME)-$(PKG_VERSION)

define KernelPackage/aic8800-usb
  SUBMENU:=Wireless Drivers
  TITLE:=AIC8800D80 USB wireless driver
  DEPENDS:=+kmod-usb-core +kmod-mac80211
  FILES:=$(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic_load_fw/aic_load_fw.ko \
    $(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800/aic8800_fdrv/aic8800_fdrv.ko
  AUTOLOAD:=$(call AutoLoad,70,aic_load_fw aic8800_fdrv)
endef

define Package/aic8800-usb-firmware
  SECTION:=firmware
  CATEGORY:=Firmware
  TITLE:=AIC8800D80 USB firmware
  DEPENDS:=+kmod-aic8800-usb
endef

define Package/aic8800-usb-firmware/install
	$(INSTALL_DIR) $(1)/lib/firmware/aic8800_fw/USB/aic8800D80
	$(CP) $(PKG_BUILD_DIR)/src/USB/driver_fw/fw/aic8800D80/* \
		$(1)/lib/firmware/aic8800_fw/USB/aic8800D80/
	$(LN) aic8800_fw/USB/aic8800D80 $(1)/lib/firmware/aic8800D80
endef

NOSTDINC_FLAGS:= \
	$(KERNEL_NOSTDINC_FLAGS) \
	-I$(STAGING_DIR)/usr/include/mac80211-backport/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211-backport \
	-I$(STAGING_DIR)/usr/include/mac80211/uapi \
	-I$(STAGING_DIR)/usr/include/mac80211 \
	-include backport/autoconf.h \
	-include backport/backport.h

define Build/Prepare
	$(call Build/Prepare/Default)
	$(FIND) $(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800 -type f \( -name '*.c' -o -name '*.h' \) -exec $(SED) 's/\<LINUX_VERSION_CODE\>/397850/g' {} +
	$(FIND) $(PKG_BUILD_DIR)/src/USB/driver_fw/drivers -type f \( -name '*.c' -o -name '*.h' -o -name 'Makefile' \) -exec $(SED) 's|/vendor/etc/firmware|/lib/firmware/aic8800_fw/USB|g' {} +
endef

define Build/Compile
	$(MAKE) -C $(PKG_BUILD_DIR)/src/USB/driver_fw/drivers/aic8800 \
		KDIR="$(LINUX_DIR)" ARCH="$(LINUX_KARCH)" CROSS_COMPILE="$(TARGET_CROSS)" \
		KBUILD_EXTRA_SYMBOLS="$(KERNEL_BUILD_DIR)/symvers/mac80211.symvers" \
		NOSTDINC_FLAGS="$(NOSTDINC_FLAGS)" modules
endef

$(eval $(call KernelPackage,aic8800-usb))
$(eval $(call BuildPackage,aic8800-usb-firmware))
