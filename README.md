# xr1710g-builds

Builds OpenWrt firmware for the Gemtek XR1710G (Brightspeed XR1710G) from
[`genshanxinli/xr1710g-openwrt`](https://github.com/genshanxinli/xr1710g-openwrt) branch `xr1710g`.

## Download

Prebuilt images are attached to [Releases](../../releases).

## Build

Actions → *build* → *Run workflow*. Or locally on a case-sensitive filesystem:

```sh
git clone -b xr1710g https://github.com/genshanxinli/xr1710g-openwrt.git openwrt
cd openwrt
cp ../configs/xr1710g.config .config
./scripts/feeds update -a && ./scripts/feeds install -a
make defconfig && make -j$(nproc)
```

Output: `bin/targets/airoha/an7581/`.

## Flashing

`xr1710g-ubi-initramfs-recovery.itb` goes to the U-Boot recovery console.
`xr1710g-ubi-sysupgrade.itb` is for `sysupgrade` from a running system.
See the source repo's [`docs/flashing.md`](https://github.com/genshanxinli/xr1710g-openwrt/blob/xr1710g/docs/flashing.md).
