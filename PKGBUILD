# Maintainer: Noel Ejemyr <noelejemyr@protonmail.com>
# Contributor: Based on Artix/Arch Linux kernel PKGBUILD

_ver=6.1.44
_rel=1
_artix=artix${_rel}

pkgbase=linux-cix
pkgname=('linux-cix' 'linux-cix-headers')
pkgver=${_ver}.${_artix}
pkgrel=1
pkgdesc='Linux-cix custom kernel for CIX CD8180 SoC (Local Build)'
url='https://www.kernel.org/' # Or your specific source URL
arch=('aarch64')
license=(GPL-2.0-only)
makedepends=(
  bc
  cpio
  gettext
  libelf
  pahole
  perl
  python
  # Add rust dependencies only if CONFIG_RUST is enabled in your .config
  # rust
  # rust-bindgen
  # rust-src
  tar
  xz
  # No git needed now
)
options=(
  !debug # Disable debug symbols unless needed
  strip  # Strip binaries/modules
)

# --- MODIFIED SOURCE ARRAY ---
# Only list files that need to be copied into the build environment.
# The kernel source directory itself is handled via symlink in prepare().
source=('config'             # Your custom .config file
        'linux-cix.preset'   # Your mkinitcpio preset file
       )

# Run 'sha256sum config' and 'sha256sum linux-cix.preset'
# The order MUST match the source=() array above.
sha256sums=('SKIP'
            'SKIP'
           )

# Set build host/user variables (optional cosmetic change)
export KBUILD_BUILD_HOST=artixlinux
export KBUILD_BUILD_USER=$pkgbase
export KBUILD_BUILD_TIMESTAMP="$(date -Ru${SOURCE_DATE_EPOCH:+d @$SOURCE_DATE_EPOCH})"


# --- MODIFIED PREPARE FUNCTION ---
prepare() {
  # $srcdir: makepkg's working directory (usually ./src/)
  # $startdir: Directory containing the PKGBUILD (usually ./)
  # $pkgbase: 'linux-cix'

  # Create a symlink inside $srcdir pointing to the *actual* source code directory
  # Ensure the path to your source code is correct!
  ln -s "$startdir/$pkgbase" "$srcdir/$pkgbase"

  # Now cd into the symlinked source directory within $srcdir
  cd "$srcdir/$pkgbase"

  echo "Setting config..."
  # Copy the config file from $srcdir (where makepkg copied it)
  cp "$srcdir/config" .config
  # Update .config with defaults for any new options
  make olddefconfig
  # Optional: show config diff
  # diff -u ../config .config || : # Note: ../config won't work here, use $srcdir/config

  echo "Setting version..."
  # Use pkgrel for localversion suffix
  echo "-$pkgrel" > localversion.10-pkgrel
  # Use pkgbase suffix for localversion suffix (e.g., -cix)
  echo "${pkgbase#linux}" > localversion.20-pkgname

  # Update EXTRAVERSION in Makefile if needed
  # sed -i -r "s/EXTRAVERSION =.*/EXTRAVERSION = -${_artix}/" "Makefile"

  # Generate the final kernel version string and store it
  make -s kernelrelease > version
  echo "Prepared $pkgbase version $(<version)"
}

build() {
  # Change directory into the source tree (via the symlink in $srcdir)
  cd "$srcdir/$pkgbase"

  # Build kernel image and modules
  make all

  # Build bpftool helpers if needed
  # make -C tools/bpf/bpftool vmlinux.h feature-clang-bpf-co-re=1
}

# Helper function for the main kernel package
_package() {
  pkgdesc="The $pkgdesc kernel and modules"
  depends=(
    coreutils
    kmod
    mkinitcpio # Explicit dependency
  )
  optdepends=(
    'linux-firmware: firmware images needed for some devices'
    'wireless-regdb: to set the correct wireless channels of your country'
  )
  provides=(
    "linux=$pkgver" # Provide the generic linux capability
  )
  conflicts=(
  )
  replaces=()

  # Change directory into the source tree (via the symlink in $srcdir)
  cd "$srcdir/$pkgbase"
  # Get the exact kernel version string generated during prepare()
  local kernel_version="$(<version)"
  local modulesdir="$pkgdir/usr/lib/modules/$kernel_version"

  echo "Installing boot image..."
  # Install the main kernel image to /boot/
  install -Dm644 "$(make -s image_name)" "$pkgdir/boot/vmlinuz-$pkgbase"

  # Used by mkinitcpio - informational file
  echo "$pkgbase" | install -Dm644 /dev/stdin "$modulesdir/pkgbase"

  echo "Installing modules..."
  # Install modules to the correct path
  make INSTALL_MOD_PATH="$pkgdir/usr" INSTALL_MOD_STRIP=1 \
     DEPMOD=/doesnt/exist modules_install # Suppress depmod

  # Remove build/source symlinks if present
  rm -f "$modulesdir"/build
  rm -f "$modulesdir"/source

  # --- MODIFIED PRESET INSTALL PATH ---
  # Install the preset file (copied to $srcdir by makepkg)
  install -Dm644 "$srcdir/linux-cix.preset" "$pkgdir/etc/mkinitcpio.d/$pkgbase.preset"
}

# Helper function for the headers package
_package-headers() {
  pkgdesc="Headers and scripts for building modules for the $pkgdesc kernel"
  depends=(pahole)
  provides=("linux-headers=$pkgver" 'linux-headers')
  # Change directory into the source tree (via the symlink in $srcdir)
  cd "$srcdir/$pkgbase"
  # Get the exact kernel version string generated during prepare()
  local kernel_version="$(<version)"
  local builddir="$pkgdir/usr/lib/modules/$kernel_version/build"

  echo "Installing build files..."
  # Install essential files for building external modules
  install -Dt "$builddir" -m644 .config Makefile Module.symvers System.map \
     localversion.* version vmlinux # tools/bpf/bpftool/vmlinux.h # If built

  install -Dt "$builddir/kernel" -m644 kernel/Makefile
  # Adjust arch path for aarch64
  install -Dt "$builddir/arch/arm64" -m644 arch/arm64/Makefile
  cp -t "$builddir" -a scripts

  echo "Installing headers..."
  cp -t "$builddir" -a include
  # Adjust arch path for aarch64
  cp -t "$builddir/arch/arm64" -a arch/arm64/include

  echo "Installing KConfig files..."
  find . -name 'Kconfig*' -exec install -Dm644 {} "$builddir/{}" \;

  # Install Rust files only if CONFIG_RUST is enabled
  # if [ -d "rust" ]; then ... fi

  echo "Removing unneeded architectures..."
  local arch
  # Keep arm64 and arm, remove others
  for arch in "$builddir"/arch/*/; do
    local arch_base=$(basename "$arch")
    if [[ $arch_base != "arm64" && $arch_base != "arm" ]]; then
       echo "Removing $(basename "$arch")"
       rm -r "$arch"
    fi
  done

  # Remove Documentation if it exists
  if [ -d "$builddir/Documentation" ]; then
     rm -r "$builddir/Documentation"
  fi

  echo "Removing broken symlinks..."
  find -L "$builddir" -type l -printf 'Removing %P\n' -delete

  echo "Removing loose objects..."
  find "$builddir" -type f -name '*.o' -printf 'Removing %P\n' -delete

  echo "Stripping build tools..."
  # (Stripping logic remains the same)
  local file
  while read -rd '' file; do
    case "$(file -Sib "$file")" in
      application/x-sharedlib\;*) strip -v $STRIP_SHARED "$file" ;;
      application/x-archive\;*) strip -v $STRIP_STATIC "$file" ;;
      application/x-executable\;*) strip -v $STRIP_BINARIES "$file" ;;
      application/x-pie-executable\;*) strip -v $STRIP_SHARED "$file" ;;
    esac
  done < <(find "$builddir" -type f -perm -u+x ! -name vmlinux -print0)

  echo "Stripping vmlinux..."
  strip -v $STRIP_STATIC "$builddir/vmlinux"

  echo "Adding source symlink..."
  # Link to the headers location for DKMS etc.
  mkdir -p "$pkgdir/usr/src"
  ln -sr "$builddir" "$pkgdir/usr/src/$pkgbase-$kernel_version" # Use version in symlink name
}


# Define the packages this PKGBUILD creates
pkgname=(
  "$pkgbase"
  "$pkgbase-headers"
)

# Dynamically create the package_xxx() functions from the _package() helpers
for _p in "${pkgname[@]}"; do
  eval "package_$_p() {
    $(declare -f "_package${_p#$pkgbase}")
    _package${_p#$pkgbase}
   }"
done
