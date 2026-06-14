# Addendum — Board-specific DDR & installing Armbian to eMMC (RK3518/RK3528 TV box)

This addendum documents two things that were *not* obvious during the original
bring-up and cost a long debugging session:

1. **The stock (public `rkbin`) DDR blobs do not work reliably on this board** —
   the only DDR that is stable is the **factory build** baked into the original
   eMMC loader.
2. **How to install Armbian onto the eMMC** without re-breaking the DDR.

---

## 1. The DDR blob is board-specific — use the factory one

### Symptoms with the public rkbin blobs

| DDR blob (rkbin `bin/rk35/`) | Result on this board |
|---|---|
| `rk3528_ddr_1056MHz_4BIT_PCB_v1.11.bin` (build *Feb 2025*) | DDR **write-training fails** (`wrtrn err`), boot hangs |
| `rk3528_ddr_1056MHz_4BIT_PCB_v1.13.bin` (build *Dec 2025*) | Same — 4BIT still hangs |
| `rk3528_ddr_1056MHz_2L_PCB_v1.11 / v1.13` | **Trains, but marginal** — boots, then **random `Synchronous Abort` in U-Boot** and unreliable MMC transfers |

The tell-tale sign of the marginal **2L** DDR is a U-Boot crash whose faulting
address is **ASCII text**, e.g.:

```
"Synchronous Abort" handler, esr 0x96000004, far 0x6566782d316d3170
```

`0x65 66 78 2d 31 6d 31 70` = `"efx-1m1p"` — a pointer overwritten by adjacent
string data (bit flips). That is the classic signature of **DDR corrupting RAM**.
It is intermittent (one boot crashes, the next doesn't) and can also masquerade
as MMC read failures (the IDMAC descriptor lives in the corrupted RAM).

### The DDR that actually works

It lives in the **original eMMC loader** (extracted from a backup of the stock
firmware). Its banner:

```
DDR 56f70fd2ad huan.he 25/06/11-11:04.04, fwver: v1.11 4bit PCB
```

Same *version label* (v1.11, 4bit PCB) as the public rkbin blob, but a **different
build (June 2025)**, tuned for this board's specific DRAM die. The public rkbin
never shipped this exact build.

> **Lesson:** before experimenting on a Rockchip box, take a full `dd` image of the
> **original eMMC** (`dd if=/dev/mmcblkX of=backup.img`). It captures the factory
> idbloader at sector 64, which is your only reliable source of the tuned DDR.

---

## 2. Working boot chain

| eMMC offset | Content |
|---|---|
| sector **64** | **factory idbloader** = factory 4BIT DDR (June build) + Rockchip SPL `v1.06` |
| sector **16384** | **mainline `u-boot.itb`** = BL31 `v1.21` + U-Boot 2026.07 (+ the SD fix below) |
| partition(s) | OS (rootfs with `/boot`) — on SD or eMMC |

Key finding: the **Rockchip SPL v1.06 loads the mainline `u-boot.itb` FIT just
fine** (it reaches mainline U-Boot, runs distro_boot). So there is **no need to
carve the DDR blob out** of the factory loader — just use the **whole factory
idbloader** (DDR + SPL) and keep the mainline `u-boot.itb` at sector 16384.

---

## 3. Recovering & deploying the factory idbloader

**Verify the factory DDR build in your backup:**
```bash
dd if=backup.img bs=1M count=4 2>/dev/null | strings | grep -i 'huan\|fwver'
# expect: DDR ... huan.he 25/06/11 ... fwver: v1.11 4bit PCB
```

**Extract the factory idbloader (DDR + SPL), from sector 64:**
```bash
dd if=backup.img of=factory_idbloader.bin bs=512 skip=64 count=4096   # ~2 MB, covers DDR+SPL
```

**Write it to the eMMC at sector 64.** Two ways:

*A) From a U-Boot prompt (mmc copy, with verify — the marginal DDR can corrupt the
transfer, so always verify and retry):*
```bash
# put it on the SD at a free raw sector first (on the build host):
sudo dd if=factory_idbloader.bin of=/dev/sdX seek=24576 conv=notrunc; sync
```
```
# at the U-Boot '=>' prompt:
mmc dev 1
mmc read 0x10000000 0x6000 0x1000      # stash from SD (2 MB)
mmc dev 0
mmc write 0x10000000 0x40 0x1000       # -> eMMC sector 64
mmc read 0x12000000 0x40 0x1000        # read back
cmp.b 0x10000000 0x12000000 0x200000   # must say "2097152 byte(s) were the same"
reset
```

*B) RKDevTool (Windows):* tab **Download Image**, set **Loader** to a working
maskrom loader (a `boot_merger` build that trains, e.g. the 2L one), add a row with
**Address `0x40`** and **Path `factory_idbloader.bin`**, then **Run**.

Keep the mainline `u-boot.itb` at sector 16384 — the factory SPL loads it.

---

## 4. SD read fix in U-Boot — `CONFIG_SYS_MMC_MAX_BLK_COUNT`

**Symptom:** `mmc fail to send stop cmd` when distro_boot loads the kernel/initrd
(large reads). Small reads (a few MB) work, large single reads fail.

**Cause:** U-Boot issues one giant multi-block read. The default
`CONFIG_SYS_MMC_MAX_BLK_COUNT` is **65535**, so a ~15–30 MB image is **not split**,
and this board's SD path can't sustain a single transfer that big (the `CMD12`
stop fails and wedges the controller — subsequent reads then report
`No partition table`).

**Fix** (in `configs/generic-rk3528_defconfig`):
```
CONFIG_SYS_MMC_MAX_BLK_COUNT=2048
```
This caps each transfer at **2048 blocks (1 MB)**; U-Boot then splits large reads
into reliable 1 MB chunks (`drivers/mmc/dw_mmc.c` sets `cfg->b_max` from this, and
`drivers/mmc/mmc.c` splits the read loop on `b_max`).

> This is **independent of clock**. Lowering `&sdmmc` `max-frequency` is *not*
> required (the problem is transfer *size*, not speed). Note this is also separate
> from the marginal-DDR corruption in §1 — fix the DDR with the factory blob, then
> the chunking fix holds.

---

## 5. Installing Armbian to the eMMC (without breaking the DDR)

The eMMC already holds the **working bootloader** (factory idbloader @64 + mainline
`u-boot.itb` @16384). The trap: **any standard clone or `armbian-install` rewrites
the idbloader @64 with a stock (marginal-DDR) one.** Golden rule:

> **Never let an install overwrite sector 64 with a non-factory idbloader.**
> If it does, re-write `factory_idbloader.bin` to sector 64 afterwards.

### Recommended: install the OS only, keep the eMMC bootloader

Boot Armbian from the SD (working), then:

```bash
lsblk        # eMMC = the ~7.3 GB device that has mmcblkXboot0 ; SD = the other
EMMC=/dev/mmcblk?     # set to your eMMC
```

1. **Partition the eMMC**, rootfs starting at sector 32768 (leaves the bootloader
   area 64–32767 untouched):
   ```bash
   sgdisk -og $EMMC
   sgdisk -n 1:32768:0 -t 1:8300 -c 1:rootfs $EMMC
   partprobe $EMMC
   mkfs.ext4 -L rootfs ${EMMC}p1
   ```
2. **Copy the running system:**
   ```bash
   mount ${EMMC}p1 /mnt
   rsync -axHAX --info=progress2 / /mnt/
   ```
3. **Point the new system at the eMMC rootfs:**
   ```bash
   blkid ${EMMC}p1          # note the UUID
   # edit /mnt/etc/fstab          -> rootfs UUID
   # edit /mnt/boot/armbianEnv.txt -> rootdev=UUID=<that UUID>
   ```
4. The bootloader on the eMMC is already correct — **leave sectors 64 and 16384
   alone.**
5. `umount /mnt`, power off, **remove the SD**, boot from eMMC.

### Alternative: a full `dd` clone of a prepared SD

A full `dd` of the SD onto the eMMC *is* valid **if you first make the SD
self-contained and correct**, i.e. write the **factory** idbloader onto the SD's
own sector 64 (not the marginal one). Then the clone carries the right idbloader to
the eMMC and there's nothing to fix afterwards:

```bash
# prepare the SD (on the build host):
sudo dd if=factory_idbloader.bin of=/dev/sdX seek=64    conv=notrunc
sudo dd if=u-boot.itb            of=/dev/sdX seek=16384 conv=notrunc
sync
# ...then write a small OS image / leave the OS partition in place...
# finally clone SD -> eMMC (from a booted system, SD=source, eMMC=target)
sudo dd if=/dev/mmcblkSD of=/dev/mmcblkEMMC bs=4M conv=fsync
```

**Cautions for the full-`dd` route:**

1. **Size.** The clone only works if the SD image's partitions **end before the
   eMMC size (~7.3 GB)**. If the rootfs was already expanded to fill a large SD
   card, `dd` writes only the first 7.3 GB and leaves a **GPT that claims the disk
   is larger** → `Invalid GPT / last_usable_lba incorrect` (the exact error from the
   original brick). Use a **fresh, un-expanded image** (rootfs ≤ 7.3 GB), or shrink
   the rootfs first.
2. **Fix the GPT if needed.** If `Invalid GPT` shows up on the eMMC after the clone,
   move the backup GPT header to the real end of the device:
   ```bash
   sgdisk -e /dev/mmcblkEMMC
   partprobe /dev/mmcblkEMMC
   ```
3. **UUID clash.** After cloning, the eMMC and SD share the same partition UUIDs.
   Test the eMMC boot with the **SD removed**, or the kernel may mount the SD's
   rootfs instead. To keep both, give the eMMC rootfs a new UUID afterwards:
   ```bash
   tune2fs -U random /dev/mmcblkEMMCp1      # then update armbianEnv.txt / fstab
   ```

> If you used the older "clobbering" route (plain clone with the *marginal*
> idbloader on the SD, or `armbian-install`), restore the working bootloader after:
> ```bash
> dd if=factory_idbloader.bin of=$EMMC seek=64    conv=notrunc
> dd if=u-boot.itb           of=$EMMC seek=16384 conv=notrunc
> sync
> ```

---

## 6. Replicating onto another (identical) box

The factory idbloader is the **manufacturer's universal blob for this model** —
every unit ships with the same one — so you can reuse the same
`factory_idbloader.bin` across identical boxes (still: back up each unit's eMMC
before touching it; and beware a different production batch may use a different
DRAM die, in which case re-test).

The one thing people miss: **on this board the eMMC boots first.** A brand-new box
boots its factory eMMC loader and **ignores the SD**. So preparing only an SD is not
enough — you must handle the eMMC. Two options:

- **Prepare the eMMC (recommended).** Put `factory_idbloader.bin`@64 +
  `u-boot.itb`@16384 + OS on the **eMMC** (e.g. the full-`dd` clone above, or §5's
  partition+rsync). The box then boots self-contained, no SD needed.
- **SD-boot.** **Erase the new box's eMMC first** so the boot ROM falls through to
  the SD, then run an SD that has the **factory** idbloader@64. (With the factory
  DDR on the SD, "empty eMMC → hang" no longer happens — that earlier failure was
  the *marginal* DDR on the SD.)

---

### One-line takeaways

- Public rkbin DDR blobs don't fit this board; the **factory June-2025 4BIT build**
  (in the original eMMC) is the only stable one — back up the eMMC first.
- Factory **idbloader (DDR + SPL v1.06) + mainline `u-boot.itb`** is a valid combo;
  no need to carve the DDR blob.
- `CONFIG_SYS_MMC_MAX_BLK_COUNT=2048` fixes the `mmc fail to send stop cmd` on
  kernel/initrd loads.
- On eMMC installs, **protect sector 64** — restore the factory idbloader if any
  tool overwrites it.
- For a full-`dd` clone: put the **factory** idbloader on the SD@64 first, keep the
  image ≤ eMMC size, fix the backup GPT (`sgdisk -e`) and the UUID clash if they
  appear. Remember the **eMMC boots first** when replicating to a new box.
