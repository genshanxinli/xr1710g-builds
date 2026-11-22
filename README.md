# xr1710g-builds

Builds OpenWrt firmware for the Gemtek XR1710G (Brightspeed XR1710G) from
[`genshanxinli/xr1710g-openwrt`](https://github.com/genshanxinli/xr1710g-openwrt) branch `xr1710g`.

## Download

Prebuilt images are attached to [Releases](../../releases).

## Installing packages (distfeeds)

Every release also carries `distfeeds-<short-sha>.tar.gz` (12,215,099 bytes for
`distfeeds-7c9d14d.tar.gz`, `sha256 b4a2e3f3…031f`) next to the images. It holds the
`.apk` files and the signed index for **that** build:

```
distfeeds.list                          the index paths below, relative to the feed base
MANIFEST.json                           counts + git_commit + image_sha256 + public_key_sha256
public-key.pem                          the key that signed these indexes
targets/airoha/an7581/packages/         core: base-files, kernel, the 67 kmod-* (81 packages)
packages/aarch64_cortex-a53/base/       82 packages
packages/aarch64_cortex-a53/luci/       18 packages
packages/aarch64_cortex-a53/packages/   5 packages
```

A release asset URL is **not** a feed: `…/download/<tag>/targets/airoha/an7581/packages/
packages.adb` answers `404` (checked with `curl -L` on 2026-11-22); only the flat asset
names exist. Extract the tarball and use the extracted directory as the base.

### 1. Put the feed where the router can reach it

```sh
sha256sum distfeeds-7c9d14d.tar.gz     # b4a2e3f355655d078cb59b7b8cbf2ee42e297e7ebc3359f6c7c26e65bfbb031f
tar -xzf distfeeds-7c9d14d.tar.gz      # -> distfeeds/
```

Serve it from a LAN host (a plain static server is enough — the paths are exact):

```sh
cd distfeeds && python3 -m http.server 8080     # base: http://192.168.1.212:8080
```

or keep it on the router; `/tmp` is a tmpfs, so it is gone after a reboot:

```sh
cat distfeeds-7c9d14d.tar.gz | ssh root@192.168.1.1 'cat > /tmp/distfeeds.tar.gz'
ssh root@192.168.1.1 'mkdir -p /tmp/feed && tar -xzf /tmp/distfeeds.tar.gz -C /tmp/feed'
# base: file:///tmp/feed/distfeeds
```

### 2. Add the four feed lines to `customfeeds.list`

```
https://FEED_BASE/packages/aarch64_cortex-a53/base/packages.adb
https://FEED_BASE/packages/aarch64_cortex-a53/luci/packages.adb
https://FEED_BASE/packages/aarch64_cortex-a53/packages/packages.adb
https://FEED_BASE/targets/airoha/an7581/packages/packages.adb
```

Write them to `/etc/apk/repositories.d/customfeeds.list`, **not** to
`distfeeds.list`. The latter is a build product, and it says so itself:

```
# This file is auto-generated and build-specific, any changes will be intentionally lost in sysupgrade.
# Add your custom feeds to /etc/apk/repositories.d/customfeeds.list
```

The four paths are the tarball's own `distfeeds.list` verbatim, with the base prepended —
`FEED_BASE` is the only part you change. All four are required: the kmods and the kernel
sit in the `targets/…` index, and the rest of their dependencies in `packages/…`.

### 3. The key is already there

The image and the feed from the same release share one key: the build writes
`public-key.pem` to `/etc/apk/keys/public-key.pem` inside the image and copies the same
file into the tarball, which is what `MANIFEST.json` records as `public_key_sha256`.
Nothing to copy, nothing to trust by hand.

Mixing builds is the only case that needs attention, and it fails loudly, not silently:

```
$ apk add phytool              # feed from build A, image from build B
WARNING: opening …/packages/aarch64_cortex-a53/base/packages.adb: UNTRUSTED signature
WARNING: opening …/packages/aarch64_cortex-a53/luci/packages.adb: UNTRUSTED signature
WARNING: opening …/packages/aarch64_cortex-a53/packages/packages.adb: UNTRUSTED signature
WARNING: opening …/targets/airoha/an7581/packages/packages.adb: UNTRUSTED signature
ERROR: unable to select packages:
  phytool (no such package):
    required by: world[phytool]
```

Check before you get there, and compare the numbers:

```sh
sha256sum /etc/apk/keys/public-key.pem
# must equal MANIFEST.json's public_key_sha256 (3f758bc4…612c for 7c9d14d)
```

If they differ and you still want that feed, point apk at the tarball's directory instead
of overwriting the image's key:

```sh
apk --keys-dir /path/to/distfeeds add iperf3
```

That flag is a real trust override, not just a quiet run — measured on the router (the four
identical warnings are abbreviated `×4`), same feed, image key `42387b8e…` that does not
match it:

```
$ apk --repositories-file /tmp/kr/repos.list add iperf3     # no --keys-dir
WARNING: opening …/packages/aarch64_cortex-a53/base/packages.adb: UNTRUSTED signature     (×4)
ERROR: unable to select packages:
  iperf3 (no such package):  required by: world[iperf3]                                  rc=1
$ apk --keys-dir /tmp/kr/feed/distfeeds add iperf3          # same dir as the feed
(1/3) Installing libatomic1 (14.4.0-r5) … (3/3) Installing iperf3 (3.22.1-r1)
OK: 24.8 MiB in 185 packages                                                             rc=0
```

Verified on 2026-11-22 in a throwaway root too (`apk --root <tmp> update`): no trusted key
→ `UNTRUSTED` ×4 and `OK: 0 distinct packages available`; `--keys-dir <distfeeds/>` →
`OK: 186 distinct packages available` (matches `MANIFEST.json`); `--keys-dir <dir holding
the wrong key>` → `UNTRUSTED` ×4 again. Read the warnings, not the exit code — `apk update`
exits 0 even when it trusts nothing. And `apk info -e` reads the installed database only;
it says nothing about signatures. `--keys-dir` is documented in `apk add --help`
(“Override the default system trusted keys directories”); bare `apk --help` lists no
global options, so it looks undocumented there.

Kernel modules also pin the kernel package they were built against — the feed's kmod
declares `depends: kernel=7.4_rc3~89a82a78…-r1` (`apk adbdump` prints it), so a kmod from
a build with a different kernel config does not resolve.

### 4. Install, and check that it landed

Pick a package the image does not ship — `apk info -e <pkg>` prints the name only when it
is installed, and on a freshly flashed image `iperf3`, `libiperf3` and `phytool` come back
empty. `iperf3` is the clean example: it is in `packages/aarch64_cortex-a53/packages/`, not
a kernel module, so it does not care about kernel version.

```sh
apk add iperf3                    # no `apk update` first: apk 3 fetches the index on demand
iperf3 --version
apk info -e iperf3                # prints `iperf3` when installed
```

Measured on the router against this release's feed:

```
$ apk --keys-dir /tmp/kr/feed/distfeeds add iperf3
(1/3) Installing libatomic1 (14.4.0-r5)
(2/3) Installing libiperf3 (3.22.1-r1)
(3/3) Installing iperf3 (3.22.1-r1)
OK: 24.8 MiB in 185 packages
$ iperf3 --version
iperf 3.22.1 (cJSON 1.7.15)
```

The `--keys-dir` in that transcript is there only because the test router runs another
build; on the image that matches this feed it is plain `apk add iperf3`. `apk del iperf3`
put the router back exactly as it was (182 packages, `diff` of the sorted `apk info -v`
list is empty).

Do not use `kmod-sched-flower` or `kmod-sched-act-police` as the example: both are in
`DEFAULT_PACKAGES`, so they are already on the flashed image and `apk add` is a no-op you
can mistake for a dead feed. For a kmod, the check is the module file, not the package
name:

```sh
apk manifest kmod-sched-act-police | grep '\.ko$'
# sha256-160:91f90757e04bc9a01968f0fb7d762d78413789aa  lib/modules/7.4-rc3/act_police.ko
```

`apk info -e <pkg>` plus that file is the "it installed" check — reinstall proof on the
router: `apk del kmod-sched-act-police` then `apk add kmod-sched-act-police` brought
`act_police.ko` back with md5 `4f40230761fef3e857638fcdb7a787b9`, identical.
Remove anything you added with `apk del <pkg>`.

`CONFIG_PACKAGE_owut` is off in this config (source tree `.config:7188`), so the feed is
for a manual `apk add` on the image you flashed, not for an online upgrade to an
untested image.

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

`xr1710g-ubi-initramfs-recovery.itb` goes to the U-Boot recovery console
(requires the third-party U-Boot; stock AXON 1.6 cannot load it).
`xr1710g-ubi-sysupgrade.itb` is for `sysupgrade` from a running system.

Do not cross-flash W1700K images: the hardware differs (RTL8261BE vs RTL8261N
10G PHY, MT7530-driven 1G LEDs) and the device stops working. See the source
repo's [README](https://github.com/genshanxinli/xr1710g-openwrt/blob/xr1710g/README.md).
