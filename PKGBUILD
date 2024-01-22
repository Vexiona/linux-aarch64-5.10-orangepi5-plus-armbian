# Maintainer: Vexiona <88843406+Vexiona@users.noreply.github.com>

_desc="AArch64 armbian kernel for Orange Pi 5 Plus devices"
_pkgver_main=5.10.160
_armbian_repo='linux-rockchip'
_armbian_commit='c2e9a95ab59937a5f0aad0ac6e12fe81f26ea2e0'
_srcname="${_armbian_repo}-${_armbian_commit}"
pkgbase=linux-aarch64-armbian-orangepi5-plus
pkgname=(
  "${pkgbase}"
  "${pkgbase}-headers"
)
pkgver="${_pkgver_main}"
pkgrel=14
arch=('aarch64')
url="https://github.com/armbian/${_armbian_repo}"
license=('GPL2')
makedepends=( # Since we don't build the doc, most of the makedeps for other linux packages are not needed here
  'kmod' 'bc' 'dtc' 'uboot-tools' 'python'
)
_toolchain_url='https://releases.linaro.org/components/toolchain/binaries/latest-7/aarch64-linux-gnu'
_toolchain_name='gcc-linaro-7.5.0-2019.12-x86_64_aarch64-linux-gnu'
options=(!strip !distcc)
source=(
  "${_toolchain_url}/${_toolchain_name}.tar.xz"
  "${_srcname}.tar.gz::${url}/archive/${_armbian_commit}.tar.gz"
  'config'
  'linux.preset'
)
sha256sums=(
  '3b6465fb91564b54bbdf9578b4cc3aa198dd363f7a43820eab06ea2932c8e0bf'
  '6e627d80b80849347b57e9e6f6d681dfb988ba2ed509731a534d2f2dda554307'
  '89e223e49312c870a9d3a36b11fcc9bbb669f43adb64f9ffb66488f2a34ee9d6'
  'bdcd6cbf19284b60fac6d6772f1e0ec2e2fe03ce7fe3d7d16844dd6d2b5711f3'
)

prepare() {
  cd "${_srcname}"

  echo "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "${pkgbase#linux}" > localversion.20-pkgname

  # Prepare the configuration file
  cat "${srcdir}/config" > '.config'
}

build() {
  export ARCH="arm64"
  export CROSS_COMPILE=$(readlink -f ${_toolchain_name}/bin)/aarch64-linux-gnu-
  echo ${CROSS_COMPILE}
  cd "${_srcname}"

  # get kernel version, which will be used later for modules
  make prepare
  make -s kernelrelease > version

  # Host LDFLAGS or other LDFLAGS set by makepkg/user is not helpful for building kernel: it should links nothing outside of itself
  unset LDFLAGS
  # Only need normal Image, as most Rockchip devices does not need/support Image.gz
  # Image and modules are built in the same run to make sure they're compatible with each other
  # -@ enables symbols in dtbs, so overlay is possible
  make ${MAKEFLAGS} DTC_FLAGS="-@" Image modules dtbs
}

_package() {
  pkgdesc="The Linux Kernel and module - ${_desc}"
  depends=(
    'coreutils'
    'initramfs'
    'kmod'
  )
  optdepends=(
    'uboot-legacy-initrd-hooks: to generate uboot legacy initrd images'
    'linux-firmware: firmware images needed for some devices'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  backup=(
    "etc/mkinitcpio.d/${pkgbase}.preset"
  )

  cd "${_srcname}"

  # Install modules
  echo "Installing modules..."
  make INSTALL_MOD_PATH="${pkgdir}/usr" INSTALL_MOD_STRIP=1 modules_install

  # Install DTBs, not to target pkg, but in srcdir, so the later package() routine could use them
  make INSTALL_DTBS_PATH="${srcdir}/dtbs" dtbs_install

  # Install pkgbase
  local _dir_module="${pkgdir}/usr/lib/modules/$(<version)"
  echo "${pkgbase}" | install -D -m 644 /dev/stdin "${_dir_module}/pkgbase"

  # Install kernel image (this is technically not vmlinuz, but I name it this way to utilize mkinitcpio's existing hooks)
  install -Dm644 arch/arm64/boot/Image "${_dir_module}/vmlinuz"

  # Remove hbuild and source links, which points to folders used when building (i.e. dead links)
  rm -f "${_dir_module}/"{build,source}

  # install mkinitcpio preset file
  sed "s|%PKGBASE%|${pkgbase}|g" ../linux.preset |
    install -Dm644 /dev/stdin "${pkgdir}/etc/mkinitcpio.d/${pkgbase}.preset"

  # Install DTB
  echo 'Installing DTBs for Rockchip SoCs...'
  install -d -m 755 "${pkgdir}/boot/dtbs/${pkgbase}"
  cp -t "${pkgdir}/boot/dtbs/${pkgbase}" -a "${srcdir}/dtbs/rockchip"
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for linux kernel - ${_desc}"
  # Compiling modules against the tree needs gcc-wrapper.py clang-wrapper.py,
  # which both need Python.
  # Discussed in https://github.com/7Ji/archrepo/issues/5
  # About why depends instead of optdepends, this is a similar decision to
  # https://bugs.archlinux.org/task/69654
  depends=('python')
  
  # Mostly copied from alarm's linux-aarch64 and modified
  cd "${_srcname}"
  local _builddir="${pkgdir}/usr/lib/modules/$(<version)/build"

  echo "Installing build files..."
  install -Dt "${_builddir}" -m644 .config Makefile Module.symvers System.map \
    localversion.* version vmlinux
  install -Dt "${_builddir}/kernel" -m644 kernel/Makefile
  install -Dt "${_builddir}/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "${_builddir}" -a scripts

  echo "Installing headers..."
  cp -t "${_builddir}" -a include
  cp -t "${_builddir}/arch/arm64" -a arch/arm64/include
  install -Dt "${_builddir}/arch/arm64/kernel" -m644 arch/arm64/kernel/asm-offsets.s


  install -Dt "${_builddir}/drivers/md" -m644 drivers/md/*.h
  install -Dt "${_builddir}/net/mac80211" -m644 net/mac80211/*.h

  # https://bugs.archlinux.org/task/13146
  install -Dt "${_builddir}/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # https://bugs.archlinux.org/task/20402
  install -Dt "${_builddir}/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "${_builddir}/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "${_builddir}/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  # https://bugs.archlinux.org/task/71392
  install -Dt "${_builddir}/drivers/iio/common/hid-sensors" -m644 drivers/iio/common/hid-sensors/*.h

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "${_builddir}/{}" \;

  echo "Removing unneeded architectures..."
  local _arch
  for _arch in "${_builddir}"/arch/*/; do
    [[ ${_arch} = */arm64/ ]] && continue
    echo "Removing $(basename "${_arch}")"
    rm -r "${_arch}"
  done

  echo "Removing documentation..."
  rm -r "${_builddir}/Documentation"

  echo "Removing broken symlinks..."
  find -L "${_builddir}" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "${_builddir}" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v ${STRIP_SHARED} "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v ${STRIP_STATIC} "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v ${STRIP_BINARIES} "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v ${STRIP_SHARED} "$file" ;;
    esac
  done < <(find "${_builddir}" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  ${CROSS_COMPILE}strip -v $STRIP_STATIC "${_builddir}/vmlinux"

  echo "Adding symlink..."
  mkdir -p "${pkgdir}/usr/src"
  ln -sr "${_builddir}" "$pkgdir/usr/src/$pkgbase"
}

for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done