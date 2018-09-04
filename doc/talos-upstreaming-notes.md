# Talos II Firmware Upstreaming Notes
## Alexander Lent <ppc64@xanderlent.com>

__WARNING:__ This build may use proprietary firmware blobs! (We try not to, but since they are the default - and currently necessary - in upstream, they may sneak in.)

Talos II ("Talos") the the OpenPOWER POWER9 Nimbus (Sforza module) mainboard by [Raptor Computing Systems](https://raptorcs.com) ("Raptor" or "RCS"). There are currently two mainboards in this family, but both use the same firmware. (The Talos II [TL2MB1] is the full dual-socket version, the Talos II Lite [TL1MB1] has only one socket.)

The [RCS Wiki](https://wiki.raptorcs.com/) is one source of useful information about Talos II and related topics.

Since Talos II was based on Romulus, changes in Romulus code were used to understand how upstream has changed since Talos diverged.

The [open-power/op-build](https://github.com/open-power/op-build) master at __v2.0.9__ was used as the base.

The [talos-op-build](https://git.raptorcs.com/git/talos-op-build/) tag __raptor-v1.05__ was used as the source of downstream code. This is on the __06-17-2018__ branch, which forks from op-build master a long time ago (__TODO__ about when). The __06-18-2018__ branch appears to be Raptor's patches rebased on a recent (__TODO__ about when, IIRC post-v2.1?) upstream version, Raptor has 67 commits that diverge in the latter branch.

Analyzed or changed files and directories follow:

## doc/talos\_upstreaming\_notes.md
(Underscores are escaped in the heading above for Markdown.)

This file was created to track the analysis and changes done in this upstreaming effort, in case it needs to be redone when RCS or upstream make more changes.

## openpower/configs
Two files need to be added here, one for configuring buildroot to create the Talos II firmware properly, and the second for configuring hostboot for the Talos II.

### openpower/configs/hostboot/talos.config
Raptor's `talos.config` seemed to be slightly nicer than upstream Romulus, though almost identical. Copied without change from __raptor-v1.05__.

__TODO:__ Do we need to change `set DISABLE_HOSTBOOT_RUNTIME` to `unset DISABLE_HOSTBOOT_RUNTIME` as Romulus did?

The changes between Romulus in __raptor-v1.05__ and Romulus upstream were compared to understand what changed in upstream.

### openpower/configs/talos_defconfig

These changes were applied to the Talos II configuration to bring it in line with upstream:

- The addition of `--disable-libsanitizer` to `BR2_EXTRA_GCC_CONFIG_OPTIONS`
- The kernel uprev to `4.18.3`. The Raptor kernel was removed, it was no different than an older upstream OpenPOWER kernel.
- `BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE` changed from `skiroot_p9_defconfig` to `skiroot_defconfig`, as per upstream
	- Changes to those files by Raptor and bdragon are not currently upstreamed
- `BR2_OCC_BIN_FILENAME="occ.bin"` is deleted
- `BR2_HCODE_INCLUDE_IONV=n` is added to prevent inclusion of NVIDIA-supporting microcode (Romulus adds`# BR2_HCODE_INCLUDE_IONV is not set` which gives the default of `y`, so we want to explicitly opt out.)
- `BR2_CAPP_UCODE_BIN_FILENAME="cappucode.bin"` is copied from Romulus. Unlike in Raptor's branches, upstream cannot remove CAPP-ucode at this time, see below.

These Raptor-specific changes were then carried forward, some adapted:

- `BR2_GLOBAL_PATCH_DIR="$(BR2_EXTERNAL_OP_BUILD_PATH)/patches/talos-patches"` was added to allow Talos-only patches (after examining how vesnin did it) 
- `BR2_PACKAGE_HOST_LINUX_HEADERS_CUSTOM_4_16=y` was specified since Raptor included it for 4.15
- `BR2_PACKAGE_LINUX_FIRMWARE=n` (etc.) was specified explicitly, rather than being not included as Raptor had done
	- This seems to successfully prevent linux-firmware binary blobs from being included without bdragon's edits to the firmware-whitelist script.
- The Talos II-specific names & XML are preserved from Raptor
- The machine XML was updated to use upstream's methods for referring to third-party repos.
	- Added:
		- `BR2_OPENPOWER_MACHINE_XML_CUSTOM_GIT=y`
		- `BR2_OPENPOWER_MACHINE_XML_CUSTOM_GIT_VALUE="https://scm.raptorcs.com/scm/git/talos-xml"`
	- Removed:
		- `BR2_OPENPOWER_MACHINE_XML_GITHUB_PROJECT_VALUE="talos-xml"`
- Added `BR2_OCC_GPU_BIN_BUILD=n` to disable GPU GPE binaries in the occ build, as Raptor does with patches to other projects, which seem to predate the addition of this option in upstream.

Changes not carried over from Raptor:

- `BR2_PACKAGE_PNV_LPC=y` and `PNV_PACKAGE_DEVMEM_ASPEED=y` are Talos-specific changes and will be addressed later in the upstreaming process.
- Until upstream allows us to build without binaries, we cannot use the following to disable binaries and explicitly include the binaries:
	- `BR2_PACKAGE_HOSTBOOT_BINARIES=n`
		- Currently `BR2_PACKAGE_HOSTBOOT_BINARIES=y`
	- `BR2_PACKAGE_CAPP_UCODE=n`
		- Currently `BR2_PACKAGE_CAPP_UCODE=y`
- The running of the `openpower/scripts/talos-extra-cleanup` script was removed. The issue that bdragon was addressing is not relevant with our current level of minimal difference to upstream.

__TODO:__ Would it be helpful to rename the target to `talosii` or `talos_ii`? I don't think so, as Raptor uses `talos` but...

## openpower/linux
Some of the patches here were upstream's and some were Raptor-specific. The Raptor-specific patches will not be inculded in all upstream kernels, so nothing in this folder changes in upstream.

Talos II-specific patches to Linux should be placed in `openpower/patches/talos-patches/linux`. See that section for details.

## openpower/overlay
__TODO:__ Raptor changes here were not carried forward but may be necessary for proper functionality.

__TODO:__ Resolve potential license issues with upstream where these changes are GPLv3 from Raptor. There are also issues of where these changes should be integrated. 

## openpower/package
__TODO:__ Raptor changes here were not carried forward but may be necessary for proper functionality.

__TODO:__ Determine the function of Raptor's two new packages and if they can be upstreamed as-is.

### openpower/package/capp-ucode
Binaries of the CAPP microcode to allow the system to use CAPI, apparently. Not included by Raptor, (_deleted_ by Raptor,) but part of upstream.

Ideally `BR2_PACKAGE_CAPP_UCODE=n` would avoid building or inclduing the CAPP microcode, but that's currently ignored.

__TODO:__ Alter PNOR build step to honor `BR2_PACKAGE_CAPP_UCODE=n`.

### openpower/package/common-p8-xml
This is apparently for POWER8, so likely has no impact on Talos II (which is POWER9-based).

Strangely enough, upstream uses _older_ commit "e02b6f6ddd5f225ddb70c286a10685df5b9267db" while Raptor uses "dc15d115bffca490e96a440195fca2c90920b3ca", the latest from open-power/common-p8-xml. 
 
__TODO:__ Forward this oddity to upstream for investigation.

### openpower/package/Config.in
Because changes from Raptor are not currently integrated, we do not include Raptor's changes here.

(Raptor just added their new packages.)

### openpower/package/devmem-aspeed
Because changes from Raptor are not currently integrated, we do not include Raptor's changes here.

Added by Raptor, uses Raptor's git repository.
It's a GPL-2.0-or-later (GPL-2.0+) program that is apparently used to read and write directly into the ASpeed BMC's memory?

It would be compiled in by default, because it depends on `BR2_BUSYBOX_SHOW_OTHERS=y` which `talos_defconfig` sets.
We might want to either make it a custom talos-only package if possible, or default it to n and set in the defconfig `BR2_PACKAGE_DEVMEM_ASPEED=y`.

### openpower/package/hcode
Raptor uses `9d40d9e1434ee0306cb10f21273617f2b5d8361c` of their special hcode version, which is on top of `000298f14d160fdd466c5a28a589bafbdf2a363a`. Neither commmit is upstream, they diverge after `699005f149f1f1eec45b20c06dc776684e4d6b0f`.

__TODO:__ What did Raptor do to diverge from upstream and why?

Use upstream (stable, the default) version.

Additionally, Raptor had the default of `BR2_HCODE_INCLUDE_IONV` set to `n` but upstream uses `y`, so `BR2_HCODE_INCLUDE_IONV=n` was set in `openpower/configs/talos_defconfig`, see appropriate section above.

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

__TODO:__ Raptor's code seems to include the ring files from `hostboot-binaries`, get them to clarify this.

### openpower/package/hostboot
Hostboot for POWER9 was split from the POWER8 version upstream, making things simpler.

Raptor uses `884b60b16009061ab84db88f918902a8c8098a4b`, which diverges from upstream after `3c2fdb8f668c9b192c59656f88b37312e1a335be`.

Upstream stable uses `d0332131ea0c06bc4f165ccdeac0307034cb981c`.

Use upstream (stable, the default) version.

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

__TODO:__ What did Raptor do to diverge from upstream and why?

### openpower/package/hostboot-binaries
Not included by Raptor, (_deleted_ by Raptor,) but part of upstream.

Ideally `BR2_PACKAGE_HOSTBOOT_BINARIES=n` would avoid building or inclduing the hostboot binary blobs, but that's currently ignored.

__TODO:__ Alter PNOR build step to honor `BR2_PACKAGE_HOSTBOOT_BINARIES=n`.

### openpower/package/hostboot-p8
This is hostboot for POWER8, because upstream split it from the current POWER9 hostboot. It isn't relevant for Talos.

### openpower/package/ima-catalog
Raptor uses upstream at `01b26a136da16a87c0b6b3c4d9f27555dca104dc`, upstream is at the newer `90237254664cadab529a397965083e38806d92e6`.

This is apparently not the newest availible from upstream's ima-catalog, but that's probably OK?

Use upstream.

### openpower/package/libflash
Uses Raptor skiboot at "2d1519ef16b773401ddcc532592505f1251c82e4" instead of upstream which uses skiboot at __v5.10.1__.

Use upstream.

__TODO:__ What did Raptor do to diverge from upstream and why?

### openpower/package/machine-xml
This one is a big change, it uses [talos-xml](https://scm.raptorcs.com/scm/git/talos-xml) directly.

The defconfig uses an older version `221192a4c3389192bd770754132215570e96610f` rather than the most current `2bf1f60bddcb664f058e0915f72803ae9e74db25` but using an older version is probably fine.

__TODO:__ Investigate how this relates to upstream `romulus-xml`, which seems very closely related.

But! Upstream has a system in place for custom machine XML now, so the `openpower/configs/talos_defconfig` was changed to make it work, rather than this packages, see above.

__TODO:__ The `raptor-aggressive` branch has a program which is licensed under the AGPLv3, and is likely (like other GPLv3 code) not suitable for upstream. However, this branch is not used by this build, nor should it be considering the unsupported nature of overclocking.

__TODO:__ Triple-check that this is OK for upstream to accept. There is probably a happy-laywer-making solution in splitting off the offending program entirely into its own repo, but that's something Raptor would have to look into.

### openpower/package/occ
OCC for POWER9 was split from the POWER8 version upstream, making things simpler.

It is no longer necessary to set `BR2_OCC_BIN_FILENAME`, but Talos never set it. `BR2_OCC_GPU_BIN_BUILD` controls building of the GPU GPE binaries with the same value as `BR2_PACKAGE_HOSTBOOT_BINARIES`, see above for more on that and why we explicitly set it to `n` anyway.

__TODO:__ Does that actually remove OCC GPU GPE binaries? Raptor changed `OPOCC_GPU_SUPPORT=1` to `OPOCC_GPU_SUPPORT=0` in `occ.mk`, and it seems as though if hostboot-binaries are present it may be included anyway?

Raptor uses `a8d07676985b31d9e0631d2529a05ce3f68f07b7`, which is on top of upstream's `77bb5e602b4aa3421e38af2b8fadb55bb2e9496b`.

Raptor's commit is described by its log "Remove GPU1 binary-only file". As stated, upstream now has a system to handle this removal.

Upstream stable is `084756c397c4cf2dc94836797b4242a997892480`.

Use upstream stable for OCC.

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

__TODO:__ Double-check this is safe. The OCC is absolutely critical.

### openpower/package/occ-p8
This is OCC for POWER8, because upstream split it from the current POWER9 OCC. It isn't relevant for Talos.

### openpower/package/openpower-ffs
__TODO:__

### openpower/package/openpower-mrw
Raptor _deleted_ this package, it is likely not relevant.

__TODO:__ Solve the mystery of this one.

### openpower/package/openpower-pnor
Raptor uses `14d3bf9c02207b2638f2565616ebaa6972da0e5c`, diverges from upstream at `864bec404e8d5a5ee19e3d5478b290cdf6ef7cab`. Upstream uses `f6d970c6a600a7e248fa5d604eb471db4482760b`.

Raptor's patches seem to allow removal of binary blobs, change parition layout, and change the size of the PNOR.

Use upstream here for now.

__TODO:__ What did Raptor do to diverge from upstream and why?

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

### openpower/package/openpower-vpnor
Raptor uses `0e30f86cb44f3ab0aa4080faf32eb4bf2a9b12b2` from upstream, upstream uses `643e730e3b9818bdd878eebee209b268c234fc65`.

Use upstream here, it's a newer version.

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

### openpower/package/p8-pore-binutils
Raptor uses upstream, didn't even change the repository.
Probably only useful on POWER8, irrelevant since Talos II uses POWER9.

### openpower/package/petitboot
Raptor uses `v1.7.1` from their Petitboot, but that just tracks upstream. Upstream uses petitboot `v1.7.2`.

Use upstream petitboot.

### openpower/package/pnv-lpc
Because changes from Raptor are not currently integrated, we do not include Raptor's changes here.

Added by Raptor. Based on something from their skiboot at `2d1519ef16b773401ddcc532592505f1251c82e4`.

__TODO:__ Why did Raptor add this package?

### openpower/package/ppe42-binutils
Raptor uses the same version of this project as upstream.

### openpower/package/ppe42-gcc
Raptor uses the same version of this project as upstream.

### openpower/package/sb-signing-framework
Raptor uses the same version of this project as upstream.

### openpower/package/sb-signing-utils
Raptor uses an old version (`v0.3`) of upstream, just use (`v0.5`) upstream.

### openpower/package/sbe
Upstream defaults to stable SBE version `55d6eb23ddd2837109ec2afa547be0b0e85bb74a`, but Raptor uses the older (but upstream) `a389a5d98c2ab38292ce7451c210b3cb0293938c` - which __v2.0.9__ lists as "latest".

__TODO:__ Reference public discussions were Raptor was told to use newer firmware to avoid bugs.

Use upstream stable version (the default) since it is newer.

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

### openpower/package/skiboot
Raptor uses `bc106a09b0ab298ec71b3ef8337fb5f820a7c454` from their skiboot repo, upstream uses `v6.0.8` of upstream skiboot.

__TODO:__ What did Raptor do to diverge from upstream and why?

Use upstream version.

__TODO:__ This needs to be tested on real hardware to make sure nothing is broken!

## openpower/patches
Talos II-specifc patches are in `openpower/patches/talos-patches`.

### openpower/patches/talos-patches/linux

Some of the Raptor-specific patches to Linux were copied without change from __raptor-v1.05__:

- `0004-drm-ast-Add-option-to-initialize-palette-on-driver-l.patch`
	- Moved from `openpower/linux` in Raptor's tree.
- `0005-Force-ASpeed-RAMDAC-palette-reset.patch`
	- Moved from `openpower/linux` in Raptor's tree.

The following patch seems to be related to Raptor's partiton support, which is not upstream, so it is currently excluded:

- `0006-mtd-powernv_flash-Enable-partition-support.patch`

__TODO:__ We should be able to rebase more of Raptor's patches to upstream firmware (hostboot, skiboot, occ, etc.) and put them here.

__TODO:__ The partition support is something that should be upstreamed if possible, according to Stewart Smith [here](https://lists.ozlabs.org/pipermail/openpower-firmware/2018-August/000253.html).
> I believe Raptor has an out of tree change that adds another partition
> on flash for optional firmware components such as the binary only blobs
> from the linux-firmware package. I'd love to see that go upstream.

## openpower/scripts
No changes currently necessary from upstream.

- Raptor's changes to `firmware-whitelist` to disallow firmware are not needed in upstream as long as linux-firmware is not built.
- Bdragon's addition here (`talos-extra-cleanup`) is currently not useful in usptream.