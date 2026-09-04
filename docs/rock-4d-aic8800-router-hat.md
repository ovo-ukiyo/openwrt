# ROCK 4D OpenWrt hardware enablement notes

This branch records the OpenWrt 25.12 work used on a Radxa ROCK 4D
(RK3576). It contains both generally useful board fixes and local image
customization. The latter is intentionally kept in separate commits so the
hardware fixes can later be submitted upstream without carrying personal
package choices.

## 1. Onboard AIC8800D80 Wi-Fi

### Symptom

The onboard AIC8800D80 works with Radxa's Armbian image, but it was not usable
with the official OpenWrt image. The otherwise unexplained internal USB device
`1a40:0101` is the board's Terminus USB hub; once the radio is powered and its
driver has loaded, the AIC USB function enumerates as `a69c:8d81`.

### Root causes

There were two independent missing pieces:

1. The ROCK 4D device tree described GPIO2_PD1 as an `rfkill-gpio`. That did
   not keep the onboard module enabled early and continuously enough for USB
   enumeration. The working description uses an always-on fixed 3.3 V
   regulator on GPIO2_PD1, including the existing `wifi_en_h` pinctrl state and
   a 5 ms startup delay.
2. OpenWrt did not contain the Radxa AIC8800 USB out-of-tree driver and the
   AIC8800D80 firmware files. A package was added from a pinned revision of
   `radxa-pkg/aic8800`. Kernel/mac80211 compatibility patches handle an unused
   cleanup label, the newer `cfg80211_ops.get_tx_power` callback signature and
   overlapping firmware-path formatting in the vendor source.

### Implemented device-tree change

The former rfkill node is replaced by a fixed regulator equivalent to:

```dts
wifi_chip_en: regulator-wifi-chip-en {
	compatible = "regulator-fixed";
	regulator-name = "wifi_chip_en";
	regulator-min-microvolt = <3300000>;
	regulator-max-microvolt = <3300000>;
	regulator-boot-on;
	regulator-always-on;
	enable-active-high;
	gpios = <&gpio2 RK_PD1 GPIO_ACTIVE_HIGH>;
	startup-delay-us = <5000>;
	pinctrl-names = "default";
	pinctrl-0 = <&wifi_en_h>;
};
```

### Validation

After a cold boot, the AIC USB function enumerated, both vendor modules loaded,
firmware was found below `/lib/firmware/aic8800_fw/USB`, and the 2.4 GHz access
point operated normally.

The AIC driver remains an out-of-tree vendor driver. Before an upstream OpenWrt
submission, its source/firmware licensing, long-term maintenance, mac80211
backport integration and removal of the forced `LINUX_VERSION_CODE` workaround
should be reviewed separately.

## 2. Dual 2.5G Router HAT FPC PCIe port

### Hardware and symptom

The tested topology is:

```text
ROCK 4D (RK3576)
└─ ASM2806 on Radxa Dual 2.5G Router HAT V1.5
   ├─ downstream port: local PCIe device
   ├─ downstream port 02:02.0: 16-pin FPC to ASM1182E switch
   ├─ downstream port: RTL8125
   └─ downstream port: RTL8125
```

On a cold boot, the two RTL8125 controllers and the other fixed endpoint were
present, but ASM2806 downstream port `02:02.0` was empty. The same ASM1182E
board and Intel Optane NVMe were stable at PCIe Gen2 x1 when connected directly
to the ROCK 4D, excluding the switch board and NVMe as the cause.

### Electrical evidence and root cause

ROCK 4D 40-pin header pin 16 is GPIO2_B6 (`gpiochip2` offset 14) and is wired to
Router HAT FPC pin 13, the downstream `POWER_EN` signal. With the stock device
tree it remained an input/low signal:

```text
40-pin pin 16:       0 V
Router HAT FPC 13:   0 V
GPIO2_B6:            input, inactive
```

The pin measured about 51.2 kOhm to ground while powered off, so it was not
shorted. Driving it high from userspace with
`gpioset -c gpiochip2 14=1` changed both points from 0 V to 3.3 V, illuminated
the ASM1182E board's power LED, and made bridge `04:00.0` appear immediately
after a PCI rescan. This established GPIO2_B6/FPC pin 13 as the missing power
enable.

Userspace enablement is too late. During the initial scan Linux assigned only
bus 04 to the empty branch. When ASM1182E was enabled later, its downstream bus
05 collided with a bus already assigned to another ASM2806 branch. The kernel
then reported:

```text
devices behind bridge are unusable because [bus 05] cannot be assigned for them
bridge has subordinate 04 but max busn 05
```

### Implemented device-tree change

A GPIO hog drives GPIO2_B6 high when the GPIO controller registers, before the
PCIe host performs its first enumeration:

```dts
&gpio2 {
	router_hat_pcie_enable_hog: router-hat-pcie-enable-hog {
		gpio-hog;
		gpios = <RK_PB6 GPIO_ACTIVE_HIGH>;
		output-high;
		line-name = "router-hat-pcie-enable";
	};
};
```

### Cold-boot validation

No userspace `gpioset`, PCI rescan or link retraining was used. On a complete
power cycle:

- the GPIO line was named `router-hat-pcie-enable` and was output-high;
- ASM1182E appeared at `04:00.0`, with downstream bridges at `05:03.0` and
  `05:07.0`;
- the Intel Optane NVMe appeared at `06:00.0` and mounted successfully at
  `/opt`;
- the RTL8125 branches were reassigned later bus numbers instead of colliding;
- the earlier bus-number error count was zero.

For upstreaming, an overlay or other HAT-specific description may be preferable
to forcing this GPIO in the base ROCK 4D device tree. The electrical function
and early-boot requirement are confirmed; the final binding/location should be
agreed with the OpenWrt and Linux Rockchip maintainers.

## 3. Local image customization

The repository also retains, in independent commits, the QModem feed, Docker,
5G modem support, filesystem/storage tools, USB/PCIe/networking packages,
RTL8125 and MT7922 support, transparent-proxy kernel modules, a 1 GiB rootfs
setting, and a local AIC8800/MT7922 startup-order workaround. These are useful
for reproducing this particular image but are not part of the proposed upstream
hardware fixes.

Apply the saved configuration seed after feeds are installed:

```sh
./scripts/feeds update -a
./scripts/feeds install -a
cp build-configs/rock-4d-custom.config .config
make defconfig
make download -j8
make -j"$(nproc)"
```

The seed contains no network credentials, passwords or device-specific UCI
configuration.
