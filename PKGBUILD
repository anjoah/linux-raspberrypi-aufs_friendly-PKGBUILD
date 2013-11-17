# Maintainer: Petros Angelatos <petrosagg@resin.io>

buildarch=18

pkgbase=linux-raspberrypi-aufs_friendly
pkgname=('linux-raspberrypi-aufs_friendly' 'linux-headers-raspberrypi-aufs_friendly')
# pkgname=linux-custom       # Build kernel with a different name
_kernelname=${pkgname#linux}
_basekernel=3.10
pkgver=${_basekernel}.19
commit=b49aafd02fa7572d387acd34550beea5b4c3d239
pkgrel=1

arch=('arm armv6h')
url="http://www.kernel.org/"
license=('GPL2')
makedepends=('xmlto' 'docbook-xsl' 'uboot-mkimage' 'git' 'python2' 'bc')
options=('!strip')
source=("linux-$pkgver.tar.gz::https://codeload.github.com/raspberrypi/linux/tar.gz/${commit}"
		'aufs3.10.tar.gz::https://codeload.github.com/morfoh/aufs3-standalone/tar.gz/aufs3.10'
		'config'
        'change-default-console-loglevel.patch'
        'usb-add-reset-resume-quirk-for-several-webcams.patch'
        'args-uncompressed.txt'
        'boot-uncompressed.txt'
        'imagetool-uncompressed.py')

md5sums=('4237274a251abbe779d1254f65d65456'
         '26c027e99e0d1e79eb90888e2f152ccf'
         'dcad931c44abc6f858b951eecb4bd7be'
         '9d3c56a4b999c8bfbd4018089a62f662'
         'd00814b57448895e65fbbc800e8a58ba'
         '9335d1263fd426215db69841a380ea26'
         'a00e424e2fbb8c5a5f77ba2c4871bed4'
         '2f82dbe5752af65ff409d737caf11954')

build() {
  cd "${srcdir}/linux-${commit}"

msg "Patches:"
  #msg2 "Add upstream patch"
  #patch -p1 -i "${srcdir}/patch-${pkgver}"

  msg2 "Add the USB_QUIRK_RESET_RESUME for several webcams"
  # FS#26528
  patch -Np1 -i "${srcdir}/usb-add-reset-resume-quirk-for-several-webcams.patch"

  # add latest fixes from stable queue, if needed
  # http://git.kernel.org/?p=linux/kernel/git/stable/stable-queue.git

  msg2 "Set DEFAULT_CONSOLE_LOGLEVEL to 4 (same value as the 'quiet' kernel param)"
  # remove this when a Kconfig knob is made available by upstream
  # (relevant patch sent upstream: https://lkml.org/lkml/2011/7/26/227)
  patch -Np1 -i "${srcdir}/change-default-console-loglevel.patch"
  cp ${srcdir}/args-uncompressed.txt arch/arm/boot/
  cp ${srcdir}/boot-uncompressed.txt arch/arm/boot/
  cp ${srcdir}/imagetool-uncompressed.py arch/arm/boot/

  #make bcmrpi_defconfig
  #sed -ri "s|^(CONFIG_LOCALVERSION=\").*|\1\-ARCH\"|" .config
  cat "${srcdir}/config" > ./.config

  msg2 "Add AUFS patches"
  cd "${srcdir}/aufs3-standalone-aufs3.10"
  cp -rp *.patch "${srcdir}/linux-${commit}"
  cp -rp fs "${srcdir}/linux-${commit}"
  cp -rp Documentation/ "${srcdir}/linux-${commit}"
  cp -rp include/ "${srcdir}/linux-${commit}"

  cd "${srcdir}/linux-${commit}"
  patch -p1 < aufs3-kbuild.patch || true # ignore error
  patch -p1 < aufs3-base.patch
  patch -p1 < aufs3-proc_map.patch
  patch -p1 < aufs3-standalone.patch


msg "Prepare to build"
  # set extraversion to pkgrel
  sed -ri "s|^(EXTRAVERSION =).*|\1 -${pkgrel}|" Makefile

  # don't run depmod on 'make install'. We'll do this ourselves in packaging
  sed -i '2iexit 0' scripts/depmod.sh

  # get kernel version
  make prepare

  # load configuration
  # Configure the kernel. Replace the line below with one of your choice.
  #make menuconfig # CLI menu for configuration
  #make nconfig # new CLI menu for configuration
  #make xconfig # X-based configuration
  #make oldconfig # using old config from previous kernel version
  # ... or manually edit .config

  # Copy back our configuration (use with new kernel version)
  #cp ./.config ../${pkgver}.config

  ####################
  # stop here
  # this is useful to configure the kernel
  # msg "Stopping build"
  # return 1
  ####################

  #yes "" | make config

 msg "Building!"
  make ${MAKEFLAGS} modules uImage
}

package_linux-raspberrypi-aufs_friendly() {
  pkgdesc="The AUFS compatible Linux Kernel and modules for Raspberry Pi"
  depends=('coreutils' 'linux-firmware' 'module-init-tools>=3.16')
  optdepends=('crda: to set the correct wireless channels of your country')
  provides=('kernel26' "linux=${pkgver}")
  conflicts=('kernel26' 'linux')
  install=${pkgname}.install

  cd "${srcdir}/linux-${commit}"

  KARCH=arm

  # get kernel version
  _kernver="$(make kernelrelease)"

  mkdir -p "${pkgdir}"/{lib/modules,lib/firmware,boot}
  make INSTALL_MOD_PATH="${pkgdir}" modules_install
  cd arch/$KARCH/boot/
  /usr/bin/python2 imagetool-uncompressed.py
  cd "${srcdir}/linux-${commit}"
  cp arch/$KARCH/boot/kernel.img ${pkgdir}/boot/kernel.img
  #cp arch/$KARCH/boot/uImage "${pkgdir}/boot/uImage"

  # set correct depmod command for install
  sed \
    -e  "s/KERNEL_NAME=.*/KERNEL_NAME=${_kernelname}/g" \
    -e  "s/KERNEL_VERSION=.*/KERNEL_VERSION=${_kernver}/g" \
    -i "${startdir}/${pkgname}.install"

  # remove build and source links
  rm -f "${pkgdir}"/lib/modules/${_kernver}/{source,build}
  # remove the firmware
  rm -rf "${pkgdir}/lib/firmware"
  # gzip -9 all modules to save 100MB of space
  find "${pkgdir}" -name '*.ko' |xargs -P 2 -n 1 gzip -9
  # make room for external modules
  ln -s "../extramodules-${pkgver}-${_kernelname:-ARCH}" "${pkgdir}/lib/modules/${_kernver}/extramodules"
  # add real version for building modules and running depmod from post_install/upgrade
  mkdir -p "${pkgdir}/lib/modules/extramodules-${pkgver}-${_kernelname:-ARCH}"
  echo "${_kernver}" > "${pkgdir}/lib/modules/extramodules-${pkgver}-${_kernelname:-ARCH}/version"

  # Now we call depmod...
  depmod -b "$pkgdir" -F System.map "$_kernver"

  # move module tree /lib -> /usr/lib
  mkdir -p "${pkgdir}/usr"
  mv "$pkgdir/lib" "$pkgdir/usr"
}

package_linux-headers-raspberrypi-aufs_friendly() {
  pkgdesc="Header files and scripts for building modules for AUFS compatible linux kernel for Raspberry Pi"
  provides=('kernel26-headers' "linux-headers=${pkgver}")
  conflicts=('kernel26-headers')

  install -dm755 "${pkgdir}/usr/lib/modules/${_kernver}"

  cd "${pkgdir}/usr/lib/modules/${_kernver}"
  ln -sf ../../../src/linux-${_kernver} build

  cd "${srcdir}/linux-${commit}"
  install -D -m644 Makefile \
    "${pkgdir}/usr/src/linux-${_kernver}/Makefile"
  install -D -m644 kernel/Makefile \
    "${pkgdir}/usr/src/linux-${_kernver}/kernel/Makefile"
  install -D -m644 .config \
    "${pkgdir}/usr/src/linux-${_kernver}/.config"

  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/include"

  for i in acpi asm-generic config crypto drm generated linux math-emu \
    media net pcmcia scsi sound trace uapi video xen; do
    cp -a include/${i} "${pkgdir}/usr/src/linux-${_kernver}/include/"
  done

   # copy arch includes for external modules
  mkdir -p ${pkgdir}/usr/src/linux-${_kernver}/arch/$KARCH
  cp -a arch/$KARCH/include ${pkgdir}/usr/src/linux-${_kernver}/arch/$KARCH/
  mkdir -p ${pkgdir}/usr/src/linux-${_kernver}/arch/$KARCH/mach-bcm2708   
  cp -a arch/$KARCH/mach-bcm2708/include ${pkgdir}/usr/src/linux-${_kernver}/arch/$KARCH/mach-bcm2708/

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
#  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/v4l2-core"

 # cp drivers/media/v4l2-core/*.h  "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/v4l2-core/"

  # pci
  for i in bt8xx cx88 saa7134; do
mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/pci/${i}"
    cp -a drivers/media/pci/${i}/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/pci/${i}"
  done
  # usb
  for i in cpia2 em28xx pwc sn9c102; do
mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/usb/${i}"
    cp -a drivers/media/usb/${i}/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/usb/${i}"
  done
  # i2c
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/i2c"
  cp drivers/media/i2c/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/i2c/"
  for i in cx25840; do
mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/i2c/${i}"
    cp -a drivers/media/i2c/${i}/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/i2c/${i}"
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
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb-core"
  cp drivers/media/dvb-core/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb-core/"
  # and...
  # http://bugs.archlinux.org/task/11194
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/include/config/dvb/"
  cp include/config/dvb/*.h "${pkgdir}/usr/src/linux-${_kernver}/include/config/dvb/"

  # add dvb headers for http://mcentral.de/hg/~mrec/em28xx-new
  # in reference to:
  # http://bugs.archlinux.org/task/13146
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb-frontends/"
  cp drivers/media/dvb-frontends/lgdt330x.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb-frontends/"
  #cp drivers/media/i2c/msp3400-driver.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/i2c/"

  # add dvb headers
  # in reference to:
  # http://bugs.archlinux.org/task/20402
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/usb/dvb-usb"
  cp drivers/media/usb/dvb-usb/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/usb/dvb-usb/"
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb-frontends"
  cp drivers/media/dvb-frontends/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/dvb-frontends/"
  mkdir -p "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/tuners"
  cp drivers/media/tuners/*.h "${pkgdir}/usr/src/linux-${_kernver}/drivers/media/tuners/"

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
  rm -rf "${pkgdir}"/usr/src/linux-${_kernver}/arch/{alpha,arm26,avr32,blackfin,cris,frv,h8300,ia64,m32r,m68k,m68knommu,mips,microblaze,mn10300,parisc,powerpc,ppc,s390,sh,sh64,sparc,sparc64,um,v850,x86,xtensa}
}
