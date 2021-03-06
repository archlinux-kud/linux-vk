# Based on PKGBUILD created for Arch Linux by:
# Jan Alexander Steffens (heftig) <jan.steffens@gmail.com>
# Tobias Powalowski <tpowa@archlinux.org>
# Thomas Baechler <thomas@archlinux.org>

# Author: Albert I <kras@raphielgang.org>

pkgbase=linux-vk
pkgver=5.2.1
pkgrel=1
arch=(x86_64)
url="https://github.com/krasCGQ/linux-vk"
license=(GPL2)
makedepends=(kmod inetutils bc libelf git)
options=('!strip')
_srcname=${pkgbase/-*}
source=(
  "$_srcname::git+$url"
  60-linux.hook    # pacman hook for depmod
  90-linux.hook    # pacman hook for initramfs regeneration
  config.compilers # configuration for custom compiler paths
  linux.install    # standard config file for post (un)install
  linux.preset     # standard config file for mkinitcpio ramdisk
  sign_modules.sh  # script to sign out-of-tree kernel modules
  x509.genkey      # preset for generating module signing key
  # for Clang asm goto hacks
  https://raw.githubusercontent.com/nathanchance/patches/master/linux/stable/0001-DO-NOT-UPSTREAM-x86-Revert-two-commits-that-break-th.patch
  https://raw.githubusercontent.com/nathanchance/patches/master/linux/stable/0002-DO-NOT-UPSTREAM-x86-Avoid-warnings-errors-due-to-lac.patch
)
sha384sums=('SKIP'
            'f7c95513e185393a526043eb0f5ecf1f800840ab3b2ed223532bb9d40ddcce44c5fab5f4b528cfd2a89bf67ad764751d'
            '01a9570c0907fa9a11ee1c384248fdf9b83de4fc2fe65cbc53446d9711aee9b148faa29a2e6449ca1a9d7b7f4cbe6c7c'
            'SKIP'
            '5b9cfec7a4e1829bd89e32c5ac3aa4f983c77dffbdedde848d305068b70ae2fee51a9d9351c6b4cb917f556db0b0f622'
            'd11ffe6e88adbcf59ca02c9481af7f7897e494eddc2345cb08ae093bf48f32ef48932fcd85224e1bfaa3db42042a6afb'
            'd5dcc15254f4ff2ac545aabf6971bd19389f89d18130bed08177721fc799ebc7d39d395366743e0d93202fc29afe7a6d'
            '4399cc1b697b95bb92e0c10e7dbd5fa6c52612aafeb8d6fb829d20bbc341fc1a6f6ef8a0c57a9509ca9f319eb34c80de'
            'SKIP'
            'SKIP')

_kernelname=${pkgbase#linux-}
_codename=TheElegant
_defconfig=$_srcname/arch/x86/configs/${_kernelname}_defconfig
_threads=$(nproc --all)

# custom binutils, clang and gcc paths
binutils_path="$(grep binutils config.compilers 2> /dev/null | cut -d '=' -f 2)"
clang_path="$(grep clang config.compilers 2> /dev/null | cut -d '=' -f 2)"
gcc_path="$(grep gcc config.compilers 2> /dev/null | cut -d '=' -f 2)"

prepare() {
  local hash

  if [ -n "$(grep MODULE_SIG=y $_defconfig)" ]; then
    msg2 "Module signing status: enabled"

    # parse hash used to sign modules from defconfig
    hash=$(grep SIG_SHA $_defconfig | sed -e s/.*._// -e s/=y//)
    # defaults to SHA1 if not found in defconfig
    [ -z "$hash" ] && hash=SHA1
    msg2 "Module signature hash algorithm used: ${hash,,}"

    if [ ! -f "../${pkgbase}.pem" ]; then
      msg2 "Generating an $hash public/private key pair..."
      openssl req -new -nodes -utf8 -${hash,,} -days 3650 -batch -x509 \
        -config x509.genkey -outform PEM -out ../${pkgbase}.pem \
        -keyout ../${pkgbase}.pem 2> /dev/null
    fi

    msg2 "Copying key pair into source folder..."
    cp -f ../${pkgbase}.pem $_srcname/${pkgbase}.pem
  else
    msg2 "Module signing status: disabled"
  fi

  cd $_srcname

  msg2 "Setting version..."
  scripts/setlocalversion --save-scmversion
  echo "-$pkgrel" > localversion.10-pkgrel
  echo "-${_kernelname^^}.$_codename" > localversion.20-pkgname

  msg2 "Generating defconfig..."
  make -s ${_kernelname}_defconfig -j${_threads}

  make -s kernelrelease > ../version
  msg2 "Prepared %s version %s" "$pkgbase" "$(<../version)"
}

build() {
  # mark variables as local
  local amdgpu_dc_enabled CC cc_temp ccache_exist compiler CROSS_COMPILE

  # is ccache exist?
  ccache_exist="$(which ccache &> /dev/null && echo true || echo false)"

  # enabled features determine how kernel and package will be treated
  # DRM_AMD_DC defaults to true in Kconfig
  amdgpu_dc_enabled=$(test -n "$(grep DRM_AMD_DC $_defconfig)" && echo false || echo true)
  msg2 "AMDGPU DC enabled: $amdgpu_dc_enabled"
  export r8168_enabled=$(test -n "$(grep R8168 $_defconfig)" && echo true || echo false)
  msg2 "R8168 enabled: $r8168_enabled"

  if $amdgpu_dc_enabled; then
    warning "Incompatible configuration detected! NOT using Clang."
  else
    # custom clang
    if find "$clang_path/bin/clang" &> /dev/null; then
      msg2 "Custom Clang detected! Building with Clang..."
      export PATH="$clang_path/bin:$PATH"
      export LD_LIBRARY_PATH="$clang_path/lib${LD_LIBRARY_PATH:+:${LD_LIBRARY_PATH}}"
      clang_exist=true
      clang_custom=true
    # clang installed on system
    elif which clang &> /dev/null; then
      msg2 "Clang detected! Building with Clang..."
      clang_exist=true
    fi
    if [ -n "$clang_exist" ]; then
      # use ccache if exist
      $ccache_exist && CC+="ccache "
      CC+=clang
      # clang < 9 doesn't support asm goto
      [ "$(clang -dumpversion | cut -d '.' -f 1)" -lt 9 ] && export no_asmgoto=true
    fi
  fi

  # custom gcc if exist
  gcc_path="$(find "$gcc_path/bin" -name '*gcc' 2> /dev/null | sort -n | head -1)"
  if [ -n "$gcc_path" ]; then
    msg2 "Custom GCC detected!"
    export PATH="$(dirname "$gcc_path"):$PATH"
    # meant for checking whether the custom gcc has prefix or not
    cc_temp="$(basename "$gcc_path" | sed -e s/gcc//)"
  fi

  # the custom gcc has prefix
  if [ -n "$cc_temp" ]; then
    # use ccache if exist and clang is absent
    if [ -z "$clang_exist" ] && $ccache_exist; then
      CROSS_COMPILE+="ccache "
    fi
    CROSS_COMPILE+="$cc_temp"
  # or... it doesn't
  else
    # opt in for possibility of building with ccache
    if [ -z "$clang_exist" ] && $ccache_exist; then
      CC="ccache gcc"
    fi
  fi

  if [ -n "$clang_exist" ]; then
    if [ -n "$clang_custom" ] && find "$binutils_path/bin/ld" &> /dev/null; then
      export PATH="$binutils_path/bin:$PATH"
      # required for LTO
      export LD_LIBRARY_PATH="$binutils_path/lib:${LD_LIBRARY_PATH:+:LD_LIBRARY_PATH}"
      binutils_exist=true
    fi
    # custom compiler string for clang
    # todo: gcc?
    # from github.com/nathanchance/scripts, slightly edited
    clang_version="$($CC --version | head -1 | cut -d \( -f 1 | sed 's/[[:space:]]*$//')"
    binutils_version="$(${cc_temp}ld --version | head -1 | cut -d ' ' -f 3-5 | sed -e 's/[\(|\)]//g')"
    msg2 "Using $clang_version, $binutils_version..."
  fi

  # set CROSS_COMPILE and CC if declared via compiler array
  if [ -n "$CROSS_COMPILE" ] && [ -z "$binutils_exist" ]; then
    compiler+=( "CROSS_COMPILE=$CROSS_COMPILE" "LD=${CROSS_COMPILE}ld.gold" )
  else
    compiler+=( "LD=ld.gold" )
  fi
  [ -n "$CC" ] && compiler+=( "CC=$CC" )
  # this is harmless on unaffected compilers
  [ -n "$clang_exist" ] && compiler+=( "CLANG_TRIPLE=$($CC -dumpmachine)" )

  cd $_srcname

  msg2 "Applying compiler sanitization features..."
  if [ -n "$clang_exist" ]; then
    # apply init stack sanitizer for clang 8+
    [ "$(echo __clang_major__ | clang -E -x c - | tail -1)" -ge 8 ] && scripts/config -e INIT_STACK_ALL

    if [ -n "$no_asmgoto" ]; then
      # build hax due to lack of asm goto support
      msg2 "Applying asm goto build hacks..."
      for i in ../*.patch; do
         patch -sNp1 < $i
      done
    fi
  else
    # apply certain sanitizer features
    scripts/config -e GCC_PLUGIN_STRUCTLEAK_BYREF_ALL \
                   -d GCC_PLUGIN_STRUCTLEAK_VERBOSE \
                   -e GCC_PLUGIN_STACKLEAK \
                   --set-val STACKLEAK_TRACK_MIN_SIZE 100 \
                   -d STACKLEAK_METRICS \
                   -d STACKLEAK_RUNTIME_DISABLE
  fi
  # enable JUMP_LABEL for supported toolchains
  # we need to keep it disabled for Clang as asm goto support is in early stage
  [ -z "$clang_exist" ] && scripts/config -e JUMP_LABEL -d STATIC_KEYS_SELFTEST
  # refresh the config just in case
  make "${compiler[@]}" oldconfig -j${_threads}

  msg2 "Building kernel and modules..."
  make "${compiler[@]}" bzImage modules -j${_threads} > /dev/null
  export image_name=$(make "${compiler[@]}" -s image_name)
}

_package() {
  pkgdesc="The Linux-VK kernel and modules"
  depends=(coreutils linux-firmware kmod mkinitcpio)
  optdepends=('crda: to set the correct wireless channels of your country')
  backup=("etc/mkinitcpio.d/$pkgbase.preset")
  install=linux.install
  # conflicts with r8168-dkms
  $r8168_enabled && conflicts+=(r8168-dkms)

  local kernver="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernver"

  # copy signing_key.x509 to PKGBUILD location
  cp -f ${_srcname}/certs/signing_key.x509 ../linux-vk.x509

  cd $_srcname

  msg2 "Installing boot image..."
  # systemd expects to find the kernel here to allow hibernation
  # https://github.com/systemd/systemd/commit/edda44605f06a41fb86b7ab8128dcf99161d2344
  install -Dm644 "$image_name" "$modulesdir/vmlinuz"
  install -Dm644 "$modulesdir/vmlinuz" "$pkgdir/boot/vmlinuz-$pkgbase"

  msg2 "Installing modules..."
  make INSTALL_MOD_PATH="$pkgdir/usr" modules_install > /dev/null

  # a place for external modules,
  # with version file for building modules and running depmod from hook
  local extramodules="extramodules$_kernelname"
  local extradir="$pkgdir/usr/lib/modules/$extramodules"
  install -Dt "$extradir" -m644 ../version
  ln -sr "$extradir" "$modulesdir/extramodules"

  # remove build and source links
  rm "$modulesdir"/{source,build}

  msg2 "Installing hooks..."
  # sed expression for following substitutions
  local subst="
    s|%PKGBASE%|$pkgbase|g
    s|%KERNVER%|$kernver|g
    s|%EXTRAMODULES%|$extramodules|g
  "

  # hack to allow specifying an initially nonexisting install file
  sed "$subst" "$startdir/$install" > "$startdir/$install.pkg"
  true && install=$install.pkg

  # fill in mkinitcpio preset and pacman hooks
  sed "$subst" ../linux.preset | install -Dm644 /dev/stdin \
    "$pkgdir/etc/mkinitcpio.d/$pkgbase.preset"
  sed "$subst" ../60-linux.hook | install -Dm644 /dev/stdin \
    "$pkgdir/usr/share/libalpm/hooks/60-$pkgbase.hook"
  sed "$subst" ../90-linux.hook | install -Dm644 /dev/stdin \
    "$pkgdir/usr/share/libalpm/hooks/90-$pkgbase.hook"

  msg2 "Fixing permissions..."
  chmod -Rc u=rwX,go=rX "$pkgdir"
}

_package-headers() {
  pkgdesc="Header files and scripts for building modules for Linux-VK kernel"

  local builddir="$pkgdir/usr/lib/modules/$(<version)/build"

  cd $_srcname

  msg2 "Installing build files..."
  install -Dt "$builddir" -m644 Makefile .config Module.symvers System.map vmlinux
  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  install -Dt "$builddir/arch/x86" -m644 arch/x86/Makefile
  cp -t "$builddir" -a scripts

  # add objtool for external module building and enabled VALIDATION_STACK option
  install -Dt "$builddir/tools/objtool" tools/objtool/objtool

  # add xfs and shmem for aufs building
  mkdir -p "$builddir"/{fs/xfs,mm}

  # ???
  mkdir "$builddir/.tmp_versions"

  msg2 "Installing headers..."
  cp -t "$builddir" -a include
  cp -t "$builddir/arch/x86" -a arch/x86/include
  install -Dt "$builddir/arch/x86/kernel" -m644 arch/x86/kernel/asm-offsets.s

  install -Dt "$builddir/drivers/md" -m644 drivers/md/*.h
  install -Dt "$builddir/net/mac80211" -m644 net/mac80211/*.h

  # http://bugs.archlinux.org/task/13146
  install -Dt "$builddir/drivers/media/i2c" -m644 drivers/media/i2c/msp3400-driver.h

  # http://bugs.archlinux.org/task/20402
  install -Dt "$builddir/drivers/media/usb/dvb-usb" -m644 drivers/media/usb/dvb-usb/*.h
  install -Dt "$builddir/drivers/media/dvb-frontends" -m644 drivers/media/dvb-frontends/*.h
  install -Dt "$builddir/drivers/media/tuners" -m644 drivers/media/tuners/*.h

  if [ -n "$no_asmgoto" ]; then
    # make the condition absolute for DKMS
    msg2 "Workarounding lack of asm goto support for Clang..."
    sed -i '144s/.*/#if 1/' "$builddir"/arch/x86/include/asm/cpufeature.h
    sed -i '14s/.*/#if 0/' "$builddir"/arch/x86/include/asm/rmwcc.h
  fi

  msg2 "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  msg2 "Removing unneeded architectures..."
  local arch
  for arch in "$builddir"/arch/*/; do
    [[ $arch = */x86/ ]] && continue
    rm -r "$arch"
  done

  msg2 "Removing documentation..."
  rm -r "$builddir/Documentation"

  msg2 "Removing broken symlinks..."
  find -L "$builddir" -type l -delete

  msg2 "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -delete

  msg2 "Stripping build tools..."
  local file
  while read -rd '' file; do
    case "$(file -bi "$file")" in
      application/x-sharedlib\;*)      # Libraries (.so)
        strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*)        # Libraries (.a)
        strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*)     # Binaries
        strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) # Relocatable binaries
        strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux)

  msg2 "Adding symlink..."
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase-$pkgver"

  msg2 "Fixing permissions..."
  chmod -Rc u=rwX,go=rX "$pkgdir"
}

pkgname=("$pkgbase" "$pkgbase-headers")
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
  }"
done

# vim:set ts=8 sts=2 sw=2 et:
