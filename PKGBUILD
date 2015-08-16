# Contributor: brenix <brenix@gmail.com>
# Contributor: graysky <graysky AT archlinux DOT us>
# Contributor: Tobias Powalowski <tpowa@archlinux.org>
# Contributor: Thomas Baechler <thomas@archlinux.org>

### PATCH AND BUILD OPTIONS
_makenconfig="n"	# tweak kernel options prior to a build via nconfig
_localmodcfg="n"	# compile ONLY probed modules
_use_current="n"	# use the current kernel's .config file
_BFQ_enable_="y"	# enable BFQ as the default I/O scheduler

### DOCS
# DETAILS FOR _localmodcfg="y"
# As of mainline 2.6.32, running with this option will only build the modules that you currently have
# probed in your system VASTLY reducing the number of modules built and the build time to do it.
#
# WARNING - make CERTAIN that all modules are modprobed BEFORE you begin making the pkg!
#
# To keep track of which modules are needed for your specific system/hardware, give my module_db script
# a try: http://aur.archlinux.org/packages.php?ID=41689  Note that if you use my script, this PKGBUILD 
# will auto run the 'sudo modprobed_db reload' for you to probe all the modules you have logged!
#
# More at this wiki page ---> https://wiki.archlinux.org/index.php/Modprobed_db

# DETAILS FOR _use_current="y"
# Enabling this option will use the .config of the RUNNING kernel rather than the ARCH defaults.
# Useful when the package gets updated and you already went through the trouble of customizing your
# config options.  NOT recommended when a new kernel is released, but again, convenient for package bumps.

# DETAILS FOR _BFQ_enable="y"
# Alternative I/O scheduler by Paolo.  For more, see: http://algo.ing.unimo.it/people/paolo/disk_sched/

pkgname=linux-rifs
true && pkgname=(linux-rifs linux-rifs-headers)
_kernelname=-rifs
_srcname=linux-3.6
pkgver=3.6.9
pkgrel=1
arch=('i686' 'x86_64')
url="http://code.google.com/p/rifs-scheduler"
license=('GPL2')
options=('!strip')
_bfqpath="http://algo.ing.unimo.it/people/paolo/disk_sched/patches/3.6.0-v5"
source=("http://www.kernel.org/pub/linux/kernel/v3.x/${_srcname}.tar.xz"
			  "http://www.kernel.org/pub/linux/kernel/v3.x/patch-${pkgver}.xz"
				'change-default-console-loglevel.patch'
				'module-symbol-waiting-3.6.patch'
				'module-init-wait-3.6.patch'
				'irq_cfg_pointer-3.6.6.patch'
				'fat-3.6.x.patch'
				'rifs-3.6-unofficial.patch'
				'nohz_fix.patch'
				'linux-rifs.preset'
				'config' 'config.x86_64'
				"${_bfqpath}/0001-block-cgroups-kconfig-build-bits-for-BFQ-v5-3.6.patch"
				"${_bfqpath}/0002-block-introduce-the-BFQ-v5-I-O-sched-for-3.6.patch")

build() {
	cd "${srcdir}/${_srcname}"

	#add upstream patch
	patch -p1 -i "${srcdir}/patch-${pkgver}"

	# set DEFAULT_CONSOLE_LOGLEVEL to 4 (same value as the 'quiet' kernel param)
	# remove this when a Kconfig knob is made available by upstream
	# (relevant patch sent upstream: https://lkml.org/lkml/2011/7/26/227)
	patch -Np1 -i "${srcdir}/change-default-console-loglevel.patch"

	# fix module initialisation
	# https://bugs.archlinux.org/task/32122
	patch -Np1 -i "${srcdir}/module-symbol-waiting-3.6.patch"
	patch -Np1 -i "${srcdir}/module-init-wait-3.6.patch"

	# fix FS#32615 - Check for valid irq_cfg pointer in smp_irq_move_cleanup_interrupt
	patch -Np1 -i "${srcdir}/irq_cfg_pointer-3.6.6.patch"

	# fix cosmetic fat issue
	# https://bugs.archlinux.org/task/32916
	patch -Np1 -i "${srcdir}/fat-3.6.x.patch"

	### Patch source with rifs patch
	# Fix double name in EXTRAVERSION
	msg "Patching source with RIFS Scheduler"
	patch -Np1 -i "${srcdir}/rifs-3.6-unofficial.patch"
	patch -Np1 -i "${srcdir}/nohz_fix.patch"

	### Patch with BFQ IO Scheduler
	msg "Patching source with BFQ patches"
	for p in $(ls ${srcdir}/000{1,2}-block*.patch); do
		patch -Np1 -i $p
	done

	### Clean tree and copy ARCH config over
	msg "Running make mrproper to clean source tree"
	make mrproper

	if [ "${CARCH}" = "x86_64" ]; then
		cat "${srcdir}/config.x86_64" > ./.config
	else
		cat "${srcdir}/config" > ./.config
	fi

	### Optionally use running kernel's config
	# code originally by nous; http://aur.archlinux.org/packages.php?ID=40191
	if [ $_use_current = "y" ]; then
		if [[ -s /proc/config.gz ]]; then
			msg "Extracting config from /proc/config.gz..."
			# modprobe configs
			zcat /proc/config.gz > ./.config
		else
			warning "You kernel was not compiled with IKCONFIG_PROC!"
			warning "You cannot read the current config!"
			warning "Aborting!"
			exit
		fi
	fi

	if [ "${_kernelname}" != "" ]; then
		sed -i "s|CONFIG_LOCALVERSION=.*|CONFIG_LOCALVERSION=\"${_kernelname}\"|g" ./.config
	 sed -i "s|CONFIG_LOCALVERSION_AUTO=.*|CONFIG_LOCALVERSION_AUTO=n|" ./.config
	fi

	### BFQ to be compiled in but not enabled
	sed -i -e s'/CONFIG_CFQ_GROUP_IOSCHED=y/CONFIG_CFQ_GROUP_IOSCHED=y\nCONFIG_IOSCHED_BFQ=y\nCONFIG_CGROUP_BFQIO=y/' \
			-i -e s'/CONFIG_DEFAULT_CFQ=y/CONFIG_DEFAULT_CFQ=y\n# CONFIG_DEFAULT_BFQ is not set/' ./.config

	### Optionally enable BFQ as the default io scheduler
	if [ $_BFQ_enable_ = "y" ]; then
		sed -i -e '/CONFIG_DEFAULT_IOSCHED/ s,cfq,bfq,' \
				-i -e s'/CONFIG_DEFAULT_CFQ=y/# CONFIG_DEFAULT_CFQ is not set\nCONFIG_DEFAULT_BFQ=y/' ./.config
	fi

	# set extraversion to pkgrel
	sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile

	# don't run depmod on 'make install'. We'll do this ourselves in packaging
	sed -i '2iexit 0' scripts/depmod.sh

	# get kernel version
	msg "Running make prepare for you to enable patched options of your choosing"
	make prepare

	### Optionally load needed modules for the make localmodconfig
	# See http://aur.archlinux.org/packages.php?ID=41689
	if [ $_localmodcfg = "y" ]; then
		msg "If you have modprobe_db installed, running it in recall mode now"
		if [ -e /usr/bin/modprobed_db ]; then
			[[ ! -x /usr/bin/sudo ]] && echo "Cannot call modprobe with sudo.  Install via pacman -S sudo and configure to work with this user." && exit 1
			sudo /usr/bin/modprobed_db recall
		fi
		msg "Running Steven Rostedt's make localmodconfig now"
		make localmodconfig
	fi

	if [ $_makenconfig = "y" ]; then
		msg "Running make nconfig"
		make nconfig
	fi

	msg "Running make bzImage and modules"
	make ${MAKEFLAGS} LOCALVERSION= bzImage modules
}

package_linux-rifs() {
_Kpkgdesc='Linux Kernel and modules with the RIFS scheduler and BFQ patches'
pkgdesc="${_Kpkgdesc}"
depends=('coreutils' 'linux-firmware' 'mkinitcpio>=0.7')
optdepends=('dkms-nvidia-beta: DKMS nvidia drivers')
provides=("linux-rifs=${pkgver}")
backup=("etc/mkinitcpio.d/linux-rifs.preset")
install=linux-rifs.install

cd "${srcdir}/${_srcname}"

KARCH=x86

# get kernel version
_kernver="$(make LOCALVERSION= kernelrelease)"
_basekernel=${_kernver%%-*}
_basekernel=${_basekernel%.*}

mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
make LOCALVERSION= INSTALL_MOD_PATH="${pkgdir}" modules_install
cp arch/$KARCH/boot/bzImage "${pkgdir}/boot/vmlinuz-linux-rifs"

# add vmlinux
install -D -m644 vmlinux "${pkgdir}/usr/src/linux-${_kernver}/vmlinux"

# install fallback mkinitcpio.conf file and preset file for kernel
install -D -m644 "${srcdir}/linux-rifs.preset" "${pkgdir}/etc/mkinitcpio.d/linux-rifs.preset"

# set correct depmod command for install
sed \
	-e  "s/KERNEL_NAME=.*/KERNEL_NAME=-rifs/g" \
	-e  "s/KERNEL_VERSION=.*/KERNEL_VERSION=${_kernver}/g" \
	-i "${startdir}/linux-rifs.install"
sed \
	-e "1s|'linux.*'|'linux-rifs'|" \
	-e "s|ALL_kver=.*|ALL_kver=\"/boot/vmlinuz-linux-rifs\"|" \
	-e "s|default_image=.*|default_image=\"/boot/initramfs-linux-rifs.img\"|" \
	-e "s|fallback_image=.*|fallback_image=\"/boot/initramfs-linux-rifs-fallback.img\"|" \
	-i "${pkgdir}/etc/mkinitcpio.d/linux-rifs.preset"

# remove build and source links
rm -f "${pkgdir}"/lib/modules/${_kernver}/{source,build}
# remove the firmware
rm -rf "${pkgdir}/lib/firmware"
# gzip -9 all modules to save 100MB of space
find "${pkgdir}" -name '*.ko' -exec gzip -9 {} \;
# make room for external modules
ln -s "../extramodules-${_basekernel}${_kernelname:rifs}" "${pkgdir}/lib/modules/${_kernver}/extramodules"
# add real version for building modules and running depmod from post_install/upgrade
mkdir -p "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname:rifs}"
echo "${_kernver}" > "${pkgdir}/lib/modules/extramodules-${_basekernel}${_kernelname:rifs}/version"

# Now we call depmod...
depmod -b "$pkgdir" -F System.map "$_kernver"

# move module tree /lib -> /usr/lib
mv "$pkgdir/lib" "$pkgdir/usr"
}

package_linux-rifs-headers() {
_Hpkgdesc='Header files and scripts to build modules for linux-rifs.'
pkgdesc="${_Hpkgdesc}"
provides=("linux-rifs-headers=${pkgver}" "linux-headers=${pkgver}")

install -dm755 "${pkgdir}/usr/lib/modules/${_kernver}"

cd "${pkgdir}/usr/lib/modules/${_kernver}"
ln -sf ../../../src/linux-${_kernver} build

cd "${srcdir}/${_srcname}"
install -D -m644 Makefile \
	"${pkgdir}/usr/src/linux-${_kernver}/Makefile"
install -D -m644 kernel/Makefile \
	"${pkgdir}/usr/src/linux-${_kernver}/kernel/Makefile"
install -D -m644 .config \
	"${pkgdir}/usr/src/linux-${_kernver}/.config"

mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/include"

for i in acpi asm-generic config crypto drm generated linux math-emu \
	media net pcmcia scsi sound trace video xen; do
cp -a include/${i} "${pkgdir}/usr/src/linux-${_kernver}/include/"
	done

	# copy arch includes for external modules
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/arch/x86"
	cp -a arch/x86/include "${pkgdir}/usr/src/linux-${_kernver}/arch/x86/"

	# copy files necessary for later builds, like nvidia and vmware
	cp Module.symvers "${pkgdir}/usr/src/linux-${_kernver}"
	cp -a scripts "${pkgdir}/usr/src/linux-${_kernver}"

	# fix permissions on scripts dir
	chmod og-w -R "${pkgdir}/usr/src/linux-${_kernver}/scripts"
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/.tmp_versions"

	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/arch/${KARCH}/kernel"

	cp arch/${KARCH}/Makefile "${pkgdir}/usr/src/linux-${_kernver}/arch/${KARCH}/"

	if [ "${CARCH}" = "i686" ]; then
		cp arch/${KARCH}/Makefile_32.cpu "${pkgdir}/usr/src/linux-${_kernver}/arch/${KARCH}/"
	fi

	cp arch/${KARCH}/kernel/asm-offsets.s "${pkgdir}/usr/src/linux-${_kernver}/arch/${KARCH}/kernel/"

	# add headers for lirc package
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/video"

	cp drivers/media/video/*.h  "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/video/"

	for i in bt8xx cpia2 cx25840 cx88 em28xx pwc saa7134 sn9c102; do
		mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/video/${i}"
		cp -a drivers/media/video/${i}/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/video/${i}"
	done

	# add docbook makefile
	install -D -m644 Documentation/DocBook/Makefile \
		"${pkgdir}/usr/src/linux-${_kernver}/Documentation/DocBook/Makefile"

	# add dm headers
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/md"
	cp drivers/md/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/md"

	# add inotify.h
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/include/linux"
	cp include/linux/inotify.h "${pkgdir}/usr/src/linux-${_kernver}/include/linux/"

	# add wireless headers
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/net/mac80211/"
	cp net/mac80211/*.h "${pkgdir}/usr/src/linux-${_kernver}/net/mac80211/"

	# add dvb headers for external modules
	# in reference to:
	# http://bugs.archlinux.org/task/9912
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/dvb-core"
	cp drivers/media/dvb/dvb-core/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/dvb-core/"
	# and...
	# http://bugs.archlinux.org/task/11194
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/include/config/dvb/"
	[[ -e include/config/dvb/ ]] && cp include/config/dvb/*.h "${pkgdir}/usr/src/linux-${_kernver}/include/config/dvb/" 

	# add dvb headers for http://mcentral.de/hg/~mrec/em28xx-new
	# in reference to:
	# http://bugs.archlinux.org/task/13146
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/frontends/"
	cp drivers/media/dvb/frontends/lgdt330x.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/frontends/"
	cp drivers/media/video/msp3400-driver.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/frontends/"

	# add dvb headers
	# in reference to:
	# http://bugs.archlinux.org/task/20402
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/dvb-usb"
	cp drivers/media/dvb/dvb-usb/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/dvb-usb/"
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/frontends"
	cp drivers/media/dvb/frontends/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb/frontends/"
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/common/tuners"
	cp drivers/media/common/tuners/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/common/tuners/"

	# add xfs and shmem for aufs building
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/fs/xfs"
	mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/mm"
	cp fs/xfs/xfs_sb.h "${pkgdir}/usr/src/linux-${_kernver}/fs/xfs/xfs_sb.h"

	# copy in Kconfig files
	for i in `find . -name "Kconfig*"`; do
		mkdir -p "${pkgdir}"/usr/src/linux-${_kernver}/`echo ${i} | sed 's|/Kconfig.*||'`
		cp ${i} "${pkgdir}/usr/src/linux-${_kernver}/${i}"
	done

	chown -R root.root "${pkgdir}/usr/src/linux-${_kernver}"
	find "${pkgdir}/usr/src/linux-${_kernver}" -type d -exec chmod 755 {} \;

	# strip scripts directory
	find "${pkgdir}/usr/src/linux-${_kernver}/scripts" -type f -perm -u+w 2>/dev/null | while read binary ; do
	case "$(file -bi "${binary}")" in
		*application/x-sharedlib*) # Libraries (.so)
			/usr/bin/strip ${STRIP_SHARED} "${binary}";;
		*application/x-archive*) # Libraries (.a)
			/usr/bin/strip ${STRIP_STATIC} "${binary}";;
		*application/x-executable*) # Binaries
			/usr/bin/strip ${STRIP_BINARIES} "${binary}";;
	esac
done

# remove unneeded architectures
rm -rf "${pkgdir}"/usr/src/linux-${_kernver}/arch/{alpha,arm,arm26,avr32,blackfin,cris,frv,h8300,ia64,m32r,m68k,m68knommu,mips,microblaze,mn10300,parisc,powerpc,ppc,s390,sh,sh64,sparc,sparc64,um,v850,xtensa}
}
# Global pkgdesc and depends are here so that they will be picked up by AUR
pkgdesc='Linux Kernel and modules with the RIFS scheduler and BFQ patches'
sha1sums=('e0bffe68a714cf3cb8548aff39c5311b2135b18c'
          '624e8b35e998a064d0b6988c66490be73f7deeac'
          'cd1f6e8d2209c4a53fc27a6c1e9f5f86e74e0c9f'
          '293ec8aba1db50ff7fb43ba1e01fe34ee3a232b2'
          '4cbd6a6c94eef9fea74e3e670a77e78f1243202b'
          '04c534d70353718679ec0e052df1f4e1173b9f8d'
          '5c83f8aa9e76fb0a74861275622b0f759c2625c4'
          '76abefb7eac82b54a410f2ec4b7c7b1534d66727'
          '194eb72966f4bfa14a1740f130cf3210b2f182c8'
          '4cf0f81d9c38b797b3df6aedae6a21bc32a2e162'
          '99461ed17918baa4900ff938c1e89bda9021789d'
          '2d5bcf344f083bca1e4a877b02c00a8f04589e9c'
          '4cff75df3020aeb9f234fdc9bcac950c3e67d163'
          '2ac74c14a5731a8b60b1803645b948364a667d5c')
