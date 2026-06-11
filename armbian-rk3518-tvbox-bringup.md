# Armbian on a generic RK3518 TV box (AIC8800D80 Wi-Fi) — full bring-up guide

> Getting mainline U-Boot + Armbian (vendor 6.1 kernel) running on a cheap RK3518
> TV box, including onboard Ethernet and **AIC8800D80 SDIO Wi-Fi**.
>
> I couldn't find a single RK3518 thread on the forum, so here is everything that
> actually worked, plus the non-obvious traps that cost me the most time. Hopefully
> it saves the next person a few days.

I'm sharing my progress here to give back to the community — there was no RK3518
documentation when I started, and I'd like the next person to have a head start.

> **Note:** I worked through this bring-up with the help of Claude (Anthropic),
> Opus 4.8 version, which assisted with the debugging and **summarized the entire
> process into this write-up**. The hardware, testing, and final results are mine;
> the AI helped me reason through the device-tree and SDIO issues and then condensed
> the whole back-and-forth into the structured guide you're reading.

**Box purchased here:** `https://pt.aliexpress.com/item/1005006148491225.html`

---

## TL;DR

- **RK3518 is part of the RK3528 family.** Use the `rk3528` rkbin blobs and the
  `rk3528` U-Boot defconfig. You need **BL31 v1.20** (it prints `RK3518 SoC`);
  older BL31 (v1.17) shipped in prebuilt images fails with *"SoC not recognized"*.
- **Device-tree base:** start from **Radxa ROCK 2F** (`rk3528-radxa-rock-2f`), not
  the Android dump. The Radxa DT boots to userspace; the round-tripped Android DT
  hangs at *"Starting kernel"*.
- **Disable PCIe** in the DT — RK3518 has no usable PCIe and the driver throws an
  external abort at boot.
- **Wi-Fi (AIC8800D80) had four separate problems**, all fixable:
  1. A conflicting `aic8800-usb-dkms` package (duplicate symbol).
  2. **The ROCK 2F status LED sits on `gpio1.3`, which on this box is SDIO data
     line D3.** The LED driver steals the pin → 4-bit SDIO writes fail → firmware
     download times out. **This was the killer.**
  3. Firmware filename mismatch (driver wants `fw_patch_table.bin`, package ships
     `fw_patch_table_8800d80_u02.bin`) → fixed with symlinks.
  4. (Earlier red herrings: SDIO clock cap and pin drive-strength — neither was the
     real cause; see the troubleshooting table.)

---

## Hardware

| Component | Detail |
|---|---|
| SoC | **RK3518A** — reports `SoC: 35181001`; boot ROM/logs say "RK3528". It is a variant inside the RK3528 family. |
| RAM | 1 GB LPDDR3, 4BIT PCB |
| eMMC | FN62MB, ~7.3 GB |
| Wi-Fi | **AIC8800D80**, SDIO, ID `c8a1:0082` (the chip is silk-screened "AIC8800D40", but it enumerates and loads as D80) |
| Ethernet (onboard) | integrated FEPHY via `gmac0` |
| Debug UART | `0xff9f0000`, **1500000** baud |

A cheap **USB-Ethernet dongle** (mine was a Davicom DM9601) is extremely useful for
SSH while you debug, since the onboard NIC and Wi-Fi come up late.

---

## Why the RK3518 is special

Rockchip never shipped a separate `rk3518` rkbin tree. RK3518 support was added
**inside the rk3528 binaries**: `bl31 >= v1.19` and `usbplug >= v1.04`. So:

- Build U-Boot with the **rk3528** defconfig.
- Use the **rk3528** DDR/BL31/SPL blobs from `rkbin`.
- The BL31 v1.20 ELF prints `RK3518 SoC` on the serial console — that's how you know
  the chip is recognized.

The stock Android loader on these boxes already uses the correct set
(DDR v1.11 4bit, SPL v1.06, BL31 v1.20, OP-TEE v1.06). Most **prebuilt Armbian
images fail** because they ship BL31 v1.17, which predates RK3518 support.

---

## 1. Build mainline U-Boot for rk3528

Build host: Ubuntu 22.04.

```bash
sudo apt update
sudo apt install -y gcc-aarch64-linux-gnu bison flex libssl-dev bc \
  device-tree-compiler swig python3-dev python3-setuptools \
  libgnutls28-dev python3-pyelftools

git clone https://github.com/rockchip-linux/rkbin
git clone https://github.com/u-boot/u-boot
cd u-boot

export CROSS_COMPILE=aarch64-linux-gnu-
export BL31=../rkbin/bin/rk35/rk3528_bl31_v1.20.elf
export ROCKCHIP_TPL=../rkbin/bin/rk35/rk3528_ddr_1056MHz_4BIT_PCB_v1.11.bin

make generic-rk3528_defconfig
make -j"$(nproc)"
```

You get `idbloader.img` (~172 KB) and `u-boot.itb` (~766 KB). Confirm BL31 is
embedded:

```bash
./tools/dumpimage -l u-boot.itb   # should list atf-1/2/3
```

> If your board uses a different DDR width/PCB, pick the matching
> `rk3528_ddr_*_v1.11.bin`. Mine is 4BIT PCB.

### Flash the loader to SD/eMMC

```bash
dd if=idbloader.img of=/dev/sdX seek=64    conv=notrunc
dd if=u-boot.itb   of=/dev/sdX seek=16384  conv=notrunc
sync
```

**Important:** writing a full Armbian image *afterwards* will overwrite the loader
with the broken v1.17 one. **Always re-flash these two files after writing the
Armbian image.**

---

## 2. Base image and device tree

Write an Armbian `rk35xx` (vendor 6.1 kernel) image, then re-flash the loader as
above.

- **DTB base:** `rk3528-radxa-rock-2f.dtb`. It boots to a login prompt. The Android
  `/proc` dump, after decompile/recompile, hangs at *"Starting kernel"* (damaged
  reserved-memory / GIC / timer nodes), so don't use it as a base — only as a
  reference for board-specific pins.

The system comes up as user/host from the Radxa image; SSH over the USB-Ethernet
dongle works immediately.

---

## 3. Serial console

The debug UART is at physical `0xff9f0000`. The Android DT numbers it `ttyS0`, the
Radxa DT numbers it `ttyS2`, so an `ttySx`-based console arg is fragile. Use an
**address-based earlycon** in `armbianEnv.txt`:

```
extraargs=earlycon=uart8250,mmio32,0xff9f0000
```

Baud is **1500000**.

---

## 4. Disable PCIe (prevents a boot crash)

RK3518 has no usable PCIe; the `rk_pcie` driver hits an external abort. In the DT,
set the PCIe controller node to disabled:

```dts
&pcie {
    status = "disabled";
};
```

(If you edit a decompiled DTB directly, find the `pcie`/`pcie@...` node and set
`status = "disabled";`.)

---

## 5. Onboard Ethernet (gmac0)

The integrated FEPHY works out of the box once enabled. In the Radxa DT, `gmac0`
(`ethernet@ffbd0000`) is already wired to the internal `phy@2`; just enable it:

```dts
&gmac0 {
    status = "okay";
};
```

Result: interface `end0`, PHY found at MDIO address `0:02` (Generic PHY driver).
Plug a cable and it gets carrier + DHCP. (The MAC may be random `7e:xx:...` if your
box has no MAC in OTP — cosmetic; you can set a static one later.)

A short `stmmac_mdio_reset` warning may appear during probe; it's benign as long as
the PHY is found and `end0` comes up.

---

## 6. Wi-Fi — AIC8800D80 over SDIO (the hard part)

The AIC8800 SDIO driver is **already built in the Armbian rk35xx vendor kernel**:

```
/lib/modules/<ver>/kernel/drivers/net/wireless/aic8800_sdio/
    aic8800_bsp/aic8800_bsp.ko      # base / firmware download
    aic8800_fdrv/aic8800_fdrv.ko    # the cfg80211/mac80211 driver
    aic8800_btlpm/aic8800_btlpm.ko  # bluetooth
```

No out-of-tree build needed. But four things must be right first.

### 6.1 Remove the conflicting USB DKMS package

Armbian ships `aic8800-usb-dkms`, whose base module exports the **same symbol**
(`aicwf_prealloc_txq_alloc`) as the in-tree `aic8800_bsp`. Loading the SDIO driver
then fails with `Exec format error` / *duplicate symbol*. Since this box uses **SDIO**
Wi-Fi, remove the USB variant:

```bash
sudo apt remove aic8800-usb-dkms
sudo depmod -a
```

### 6.2 Device-tree: SDIO host + power sequence + wlan platdata

Enable the SDIO controller and add the AIC8800 power/clock plumbing. Conceptually
(label form; if you edit a decompiled DTB use the numeric phandles):

```dts
/* SDIO host */
&sdio0 {                      /* mmc@ffc10000 */
    status            = "okay";
    bus-width         = <4>;
    non-removable;
    no-sd;
    no-mmc;
    supports-sdio;
    cap-sd-highspeed;
    cap-sdio-irq;
    keep-power-in-suspend;
    sd-uhs-sdr104;
    max-frequency     = <100000000>;
    mmc-pwrseq        = <&sdio_pwrseq>;
    /* pins: sdio0 bus4 + clk + cmd (see 6.4 for the drive-strength note) */
};

/* power sequence: WIFI_REG_ON + 32 kHz clock output */
sdio_pwrseq: sdio-pwrseq {
    compatible            = "mmc-pwrseq-simple";
    pinctrl-names         = "default";
    pinctrl-0             = <&wifi_32k>;        /* gpio1.19 func1, the 32k clk-out */
    post-power-on-delay-ms = <200>;
    reset-gpios           = <&gpio1 6 GPIO_ACTIVE_LOW>;   /* WIFI_REG_ON, gpio1.6 */
};

/* Rockchip wlan platform data (rfkill_wlan) */
wireless-wlan {
    compatible        = "wlan-platdata";
    rockchip,grf      = <&grf>;
    wifi_chip_type    = "AIC8800";
    WIFI,host_wake_irq = <&gpio1 7 GPIO_ACTIVE_HIGH>;     /* gpio1.7 */
    status            = "okay";
};
```

Notes:
- `wifi_32k` is the pinmux that routes the SoC 32.768 kHz clock out on **gpio1.19,
  function 1** (same pin the Android DT uses). On Rockchip you'll find it as
  `clkm1-32k-out` in the pinctrl/clk section.
- The `[WLAN_RFKILL] ... The ref_wifi_clk not found !` log line is **harmless** — the
  Android DT doesn't provide that optional clock either, and it still works.
- Keep the generic `mmc-pwrseq` model. I tried switching to the Rockchip
  `WIFI,poweren_gpio` model (removing the pwrseq) and it stopped the chip from
  enumerating — the pwrseq model gets furthest.

At this point the chip enumerates at boot:

```
mmc2: new ultra high speed SDR104 SDIO card at address 390b
```

### 6.3 ⚠️ THE BIG ONE: a status LED steals SDIO data line D3

This is the trap that wasted most of my time, and it's specific to using the
**Radxa ROCK 2F** DT on a **different board**.

The ROCK 2F DT has a `gpio-leds` node with a heartbeat status LED on **gpio1.3**:

```dts
gpio-leds {
    compatible = "gpio-leds";
    state-led {
        gpios = <&gpio1 3 GPIO_ACTIVE_HIGH>;   /* gpio1.3 */
        linux,default-trigger = "heartbeat";
    };
};
```

On **this TV box, gpio1.3 is SDIO D3** (one of the four data lines). The LED driver
grabs the pin and the kernel logs:

```
pin 35 already requested by ffc10000.mmc; switch mux 1 to GPIO
```

(`pin 35` = gpio1.3.) With D3 muxed to GPIO, the SDIO bus is effectively 2–3 bit:

- 1-bit commands (CMD52) still work → the card **enumerates** and the driver **reads**
  the chip vendor/device IDs fine.
- 4-bit block writes (CMD53) — the firmware download (~1.5 KB packets) — **fail with
  `-110` (timeout)**:

```
aicwf_sdio_send_pkt fail-110
send faild:0, 0,5dc
```

Because reads worked and only writes failed, I chased SDIO clock speed and pin
drive-strength for ages — all dead ends. **The real cause was a missing data line.**

**Fix:** remove that LED (the pin isn't an LED on this board). If you want to keep a
status LED, repoint it to the board's actual LED. On my box the real (blue) LED is on
**gpio4.11** (this is the `power` LED in the Android dump):

```dts
gpio-leds {
    compatible = "gpio-leds";
    power-led {
        label = "power";
        gpios = <&gpio4 11 GPIO_ACTIVE_HIGH>;   /* gpio4.11 */
        linux,default-trigger = "default-on";
        default-state = "on";
    };
};
```

After this, the `pin 35 ... switch mux 1 to GPIO` line disappears and 4-bit SDIO
writes work.

> General lesson: when reusing another board's DT, audit every `gpios`/`reset-gpios`
> consumer for pins that overlap your SDIO/eMMC data lines (and other muxed
> peripherals). A disabled node (e.g. PCIe `reset-gpios` on gpio1.2 here) is fine —
> it doesn't claim the pin — but an **active** consumer like an LED will.

### 6.4 Pin drive-strength (optional, match the Android DT)

The Radxa SDIO pins use `pcfg-pull-up-drv-level-2` (weak) and a pull-up on the clock.
The Android DT uses default drive and **no pull on the clock**. Matching Android is the
safe choice:

- data (bus4) + cmd → `pcfg_pull_up` (default drive)
- clk → `pcfg_pull_none`

In my case this was **not** the actual fix (the LED/D3 issue was), but it's worth
aligning with the known-good Android config. Don't over-crank drive strength — too
high causes overshoot/ringing and the writes fail *again*.

### 6.5 Firmware filename mismatch → symlinks

With D3 freed, the driver finally reaches firmware download — and then fails to open
the file:

```
aicbsp_driver_fw_init, chip rev: 199
rwnx_load_firmware: firmware path = /lib/firmware/aic8800/SDIO/aic8800D80//fw_patch_table.bin
rwnx_load_firmware: fw_patch_table.bin file failed to open
```

The vendor driver asks for **un-suffixed** names, but the Armbian `aic8800-firmware`
package installs **suffixed** ones (`*_8800d80_u02.bin`). Create symlinks inside the
D80 directory (pointing at the real D80 blobs — do **not** use the files from the
sibling `aic8800/` dir, those are for a different chip):

```bash
cd /lib/firmware/aic8800/SDIO/aic8800D80/
sudo ln -sf fw_patch_table_8800d80_u02.bin fw_patch_table.bin
sudo ln -sf fw_patch_8800d80_u02.bin       fw_patch.bin
sudo ln -sf fmacfw_8800d80_u02.bin         fmacfw.bin
sudo ln -sf fw_adid_8800d80_u02.bin        fw_adid.bin
sudo ln -sf lmacfw_rf_8800d80_u02.bin      lmacfw_rf.bin
sudo ln -sf aic_userconfig_8800d80.txt     aic_userconfig.txt
```

> If you later upgrade the `aic8800-firmware` package it may recreate the directory
> and drop the symlinks — keep a small script to re-create them.

### 6.6 Load the driver

Always reboot for a clean chip state before loading (repeated failed power-ons wedge
the chip; you'll see `aicbsp_dummy_sdmmc ... -123` on retries):

```bash
sudo reboot
# after boot:
sudo modprobe aic8800_fdrv      # pulls aic8800_bsp automatically
dmesg | grep -iE 'aic|rwnx|wlan'
ip -br link
```

Success looks like a `wlan0` interface with a **real MAC** (not a random
`7e:..`):

```
wlan0  DOWN  88:00:33:77:27:ef  <BROADCAST,MULTICAST>
```

### 6.7 Connect + make it persistent

```bash
sudo nmcli radio wifi on
nmcli device wifi list
sudo nmcli device wifi connect "YOUR_SSID" password "YOUR_PASS"

# load the module at boot (firmware symlinks already persist on rootfs)
echo "aic8800_fdrv" | sudo tee /etc/modules-load.d/aic8800.conf
```

---

## Final working state

- Boots mainline U-Boot (rk3528, BL31 v1.20) → Armbian vendor 6.1 kernel.
- Serial console on `0xff9f0000` @ 1500000.
- **Ethernet** `end0` (onboard FEPHY) — link + DHCP with a cable.
- **Wi-Fi** `wlan0` (AIC8800D80 SDIO) — scans, connects, real MAC.
- eMMC + SD readable; PCIe disabled (intentionally).

---

## Troubleshooting quick reference

| Symptom | Real cause | Fix |
|---|---|---|
| Prebuilt image: *"SoC not recognized"* | BL31 v1.17 predates RK3518 | Build loader with rk3528 **BL31 v1.20** |
| Hang at *"Starting kernel"* | Decompiled Android DT is damaged | Use Radxa ROCK 2F DT as base |
| External abort at boot | `rk_pcie` (no usable PCIe) | `status = "disabled"` on PCIe node |
| `aic8800_fdrv`: `Exec format error` / duplicate symbol | `aic8800-usb-dkms` clashes with in-tree `aic8800_bsp` | `apt remove aic8800-usb-dkms` + `depmod -a` |
| Chip enumerates, reads OK, **firmware TX `-110`**, `send_pkt fail` | An LED (`gpio1.3`) steals SDIO **D3** → broken 4-bit | Remove/repoint the `gpio-leds` entry on gpio1.3 |
| `fw_patch_table.bin file failed to open` | Driver wants un-suffixed names; package ships `*_8800d80_u02.bin` | Symlink un-suffixed → suffixed in `aic8800D80/` |
| `aicbsp_dummy_sdmmc ... -123` on 2nd+ attempt | Chip wedged by repeated failed power-ons | Reboot, then a **single** `modprobe` |
| `[WLAN_RFKILL] ref_wifi_clk not found` | Optional clock not referenced (Android lacks it too) | Ignore — harmless |

---

## Credits / notes

- DT base: Radxa ROCK 2F (`rk3528-radxa-rock-2f`).
- Reference for board-specific pins (Wi-Fi 32k, WIFI_REG_ON, host-wake, LED):
  the stock **Android device tree** dumped from the box.
- Chip family insight: RK3518 support lives inside Rockchip's **rk3528** rkbin blobs.

If you have a similar RK3518 box, the **LED-on-SDIO-D3** trap (6.3) and the
**firmware symlinks** (6.5) are the two things most likely to bite you. Everything
else is standard Rockchip bring-up.

*Posted to help the next person — there were zero RK3518 threads when I started.*
