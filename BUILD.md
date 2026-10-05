# Building the patched `sfp.ko`

This document describes how to build the patched SFP kernel module for the Banana Pi BPI-R4 running OpenWrt 25.12.5.

The resulting module was tested with:

* **Hardware:** Banana Pi BPI-R4
* **OpenWrt:** 25.12.5
* **Kernel:** 6.12.94
* **Target:** `mediatek/filogic`
* **Architecture:** `aarch64_cortex-a53`
* **SFP module:** OEM XGSPONST2001

The build uses the official OpenWrt 25.12.5 source tree and the following upstream Linux changes:

* `f53167e29b8e9178ce030f7de54634af1eb0bc0e` — allow prefix matching in SFP quirk lookup
* `03fa69146f2fe18742c0e12cfbf1d10c23d5b567` — add quirks for OEM XGSPONST2001 and FS XGS-SFP-ONT-MACI

## 1. Build environment

The build was performed under Ubuntu on WSL2.

OpenWrt must be built on a **case-sensitive Linux filesystem**. Do not place the source tree under `/mnt/c`, `/mnt/d`, etc.

Use a directory under the WSL home directory, for example:

```bash
cd ~
```

Install the required host packages:

```bash
sudo apt update

sudo apt install -y \
    build-essential \
    clang \
    flex \
    bison \
    gawk \
    gettext \
    git \
    libncurses-dev \
    libssl-dev \
    python3-setuptools \
    rsync \
    swig \
    unzip \
    zlib1g-dev \
    file \
    wget \
    zstd
```

## 2. Clone OpenWrt 25.12.5

Clone the exact OpenWrt release used by the router:

```bash
cd ~

git clone --branch v25.12.5 --depth 1 https://github.com/openwrt/openwrt.git

cd ~/openwrt
```

## 3. Use the official Filogic build configuration

Download the official OpenWrt 25.12.5 Filogic build configuration:

```bash
wget https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic/config.buildinfo -O .config
```

Generate the final configuration:

```bash
make defconfig
```

Verify that the SFP driver is configured as a kernel module:

```bash
grep '^CONFIG_PACKAGE_kmod-sfp=' .config
```

The expected result is:

```text
CONFIG_PACKAGE_kmod-sfp=m
```

## 4. Prepare the OpenWrt kernel source

Prepare the kernel source tree:

```bash
make target/linux/prepare V=s
```

Locate the prepared Linux source tree:

```bash
find build_dir -path '*/linux-6.12.94/drivers/net/phy/sfp.c'
```

On the BPI-R4/Filogic build used for testing, the source tree was:

```text
build_dir/target-aarch64_cortex-a53_musl/linux-mediatek_filogic/linux-6.12.94
```

Change into the kernel source directory:

```bash
cd build_dir/target-aarch64_cortex-a53_musl/linux-mediatek_filogic/linux-6.12.94
```

## 5. Download the upstream SFP patches

Download the two upstream Linux patches:

```bash
wget -O /tmp/0001-sfp-prefix.patch \
    https://github.com/torvalds/linux/commit/f53167e29b8e9178ce030f7de54634af1eb0bc0e.patch

wget -O /tmp/0002-sfp-xgsponst2001.patch \
    https://github.com/torvalds/linux/commit/03fa69146f2fe18742c0e12cfbf1d10c23d5b567.patch
```

## 6. Apply the patches

Apply the prefix-matching patch first:

```bash
patch -p1 < /tmp/0001-sfp-prefix.patch
```

Then apply the XGSPONST2001 quirk:

```bash
patch -p1 < /tmp/0002-sfp-xgsponst2001.patch
```

The patches should apply successfully.

Verify that the XGSPONST2001 quirk is present:

```bash
grep -n -A12 -B8 'XGSPONST2001' drivers/net/phy/sfp.c
```

The resulting source should contain:

```c
SFP_QUIRK_F_PREFIX("OEM", "XGSPONST2001", sfp_fixup_potron),
```

Also verify the prefix-matching support:

```bash
grep -n 'part_prefix_match' drivers/net/phy/sfp.c drivers/net/phy/sfp.h
```

## 7. Install the prebuilt OpenWrt LLVM-BPF toolchain

OpenWrt 25.12.5 can use a prebuilt LLVM-BPF toolchain instead of building LLVM-BPF from source.

Return to the OpenWrt root:

```bash
cd ~/openwrt
```

Download the official prebuilt toolchain:

```bash
wget https://downloads.openwrt.org/releases/25.12.5/targets/mediatek/filogic/llvm-bpf-21.1.6.Linux-x86_64.tar.zst
```

Extract it:

```bash
tar --use-compress-program=unzstd -xf llvm-bpf-21.1.6.Linux-x86_64.tar.zst
```

Verify the extraction:

```bash
ls -l llvm-bpf/bin/clang
cat llvm-bpf/.llvm-version
```

The LLVM version should be:

```text
21.1.6
```

Select the prebuilt BPF toolchain:

```bash
sed -i '/^CONFIG_BPF_TOOLCHAIN_BUILD_LLVM=/d' .config
sed -i '/^CONFIG_BPF_TOOLCHAIN_PREBUILT=/d' .config

echo 'CONFIG_BPF_TOOLCHAIN_PREBUILT=y' >> .config

make defconfig
```

Verify:

```bash
grep -E '^CONFIG_(BPF_TOOLCHAIN_|HAS_PREBUILT_LLVM_TOOLCHAIN|USE_LLVM_(PREBUILT|BUILD))' .config
```

The expected configuration is:

```text
CONFIG_BPF_TOOLCHAIN_PREBUILT=y
CONFIG_HAS_PREBUILT_LLVM_TOOLCHAIN=y
CONFIG_USE_LLVM_PREBUILT=y
```

## 8. Build the OpenWrt host tools

Build the OpenWrt host tools:

```bash
make tools/install -j$(nproc)
```

## 9. Build the OpenWrt AArch64 toolchain

Build the target cross-compiler:

```bash
make toolchain/install -j$(nproc)
```

Verify that the AArch64 compiler exists:

```bash
find staging_dir -name 'aarch64-openwrt-linux-musl-gcc'
```

The result should point to an OpenWrt toolchain similar to:

```text
staging_dir/toolchain-aarch64_cortex-a53_gcc-14.3.0_musl/bin/aarch64-openwrt-linux-musl-gcc
```

## 10. Compile the kernel and modules

Return to the OpenWrt root if necessary:

```bash
cd ~/openwrt
```

Compile the kernel and modules:

```bash
make target/linux/compile V=s -j$(nproc)
```

This builds much more than just `sfp.ko`, but it uses the exact OpenWrt kernel configuration and toolchain needed to produce a compatible module.

## 11. Locate the compiled module

Find the resulting SFP module:

```bash
find build_dir -name 'sfp.ko' -print
```

The expected location is similar to:

```text
build_dir/target-aarch64_cortex-a53_musl/linux-mediatek_filogic/linux-6.12.94/drivers/net/phy/sfp.ko
```

## 12. Verify the module

Confirm that the XGSPONST2001 quirk is present in the compiled module:

```bash
strings "$(find build_dir -name 'sfp.ko' | head -1)" | grep XGSPONST2001
```

Expected output:

```text
XGSPONST2001
```

Verify the kernel module ABI:

```bash
modinfo "$(find build_dir -name 'sfp.ko' | head -1)" | grep vermagic
```

Expected output:

```text
vermagic:       6.12.94 SMP mod_unload aarch64
```

## 13. Test the module

Copy the compiled module to the BPI-R4:

```bash
scp build_dir/target-aarch64_cortex-a53_musl/linux-mediatek_filogic/linux-6.12.94/drivers/net/phy/sfp.ko \
    root@<BPI-R4-IP>:/tmp/sfp-patched.ko
```

On the BPI-R4, back up the currently installed module:

```sh
cp /lib/modules/6.12.94/sfp.ko /root/sfp.ko.stock
```

Unload the stock module:

```sh
rmmod sfp
```

Load the patched module directly from `/tmp`:

```sh
insmod /tmp/sfp-patched.ko
```

Verify:

```sh
lsmod | grep -E 'sfp|mdio_i2c'
```

Then inspect the kernel log:

```sh
dmesg | tail -40
```

For the tested XGSPONST2001 module, the important result is that the SFP module remains operational and the BPI-R4 reports:

```text
mtk_soc_eth 15100000.ethernet sfp-wan: Link is Up - 10Gbps/Full
```

The stock driver produced:

```text
module transmit fault indicated
module transmit fault recovered
...
module persistently indicates fault, disabling
```

The patched driver does not enter that TX_FAULT shutdown loop.

## 14. Installing the patched module permanently

Once the patched module has been verified:

```sh
cp /lib/modules/6.12.94/sfp.ko /root/sfp.ko.stock
cp /tmp/sfp-patched.ko /lib/modules/6.12.94/sfp.ko
```

Reboot the BPI-R4:

```sh
reboot
```

After reboot, verify that the patched module is being loaded:

```sh
strings /lib/modules/6.12.94/sfp.ko | grep XGSPONST2001
```

```sh
modinfo sfp | grep -E 'filename|vermagic'
```

And check for TX_FAULT failures:

```sh
dmesg | grep 'transmit fault'
```

A successful boot should show the XGSPONST2001 module loading and `sfp-wan` reaching a 10-Gbps host-side link without the persistent TX_FAULT shutdown.

## 15. Compatibility warning

The supplied prebuilt module is intended specifically for:

```text
OpenWrt 25.12.5
Linux 6.12.94
mediatek/filogic
aarch64
Banana Pi BPI-R4
```

Do **not** assume that the `.ko` is compatible with a different kernel release.

For another OpenWrt release or kernel version, rebuild the module using the included upstream patches.

## 16. Source of the fix

The XGSPONST2001 quirk is based on upstream Linux changes:

* `f53167e29b8e9178ce030f7de54634af1eb0bc0e`
* `03fa69146f2fe18742c0e12cfbf1d10c23d5b567`

The underlying fix is an upstream Linux SFP quirk; this repository provides a backport/build for OpenWrt 25.12.5.
