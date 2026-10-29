# Goodix TLS 511 cancellation fix

This patch fixes delayed cancellation of active fingerprint verification on
the Goodix TLS 511 reader (`27c6:5110`). It was tested on Arch Linux with
`fprintd` and a lock screen that uses the fprintd D-Bus API.

Without the fix, stopping verification after failed scans can block a password
unlock while the Goodix driver waits for its pending read. With the fix,
`VerifyStop` cancels the read, allows the next operation to restart the read
loop, and completes the driver state machine promptly.

## Requirements

- Goodix TLS 511 reader: `lsusb` reports `27c6:5110`
- Arch Linux
- A Goodix TLS 511 libfprint build, for example
  [`libfprint-goodixtls511-git`](https://aur.archlinux.org/packages/libfprint-goodixtls511-git)
- `fprintd`

This is not a patch for stock upstream `libfprint`, which does not currently
support this reader.

## Apply to the AUR build directory

Build and install the AUR package once so its source directory exists:

```sh
yay -S libfprint-goodixtls511-git
```

Apply the patch:

```sh
cd ~/.cache/yay/libfprint-goodixtls511-git/src/libfprint-goodixtls511-git
git apply /path/to/goodix-5110-cancel-fix/patches/0001-cancel-active-goodix-511-operation.patch
```

Rebuild the package from its parent directory:

```sh
cd ~/.cache/yay/libfprint-goodixtls511-git
makepkg -e --nocheck --force
sudo pacman -U libfprint-goodixtls511-git-r*.pkg.tar.zst
sudo systemctl restart fprintd.service
```

`--nocheck` is only used because the package's AppStream validation can fail
when its upstream URL is unreachable. The library itself should compile
normally; run `ninja -C build test` from the source directory if the external
metadata check is available on your network.

## Verify

1. Start a fingerprint verification or lock the session.
2. Make several failed scans.
3. Switch immediately to password unlock.

The password should be processed without the previous delay. Check that the
installed package has the patched local release:

```sh
pacman -Q libfprint-goodixtls511-git
```

## Scope and caveats

- The patch only changes cancellation for the Goodix TLS 5xx operation state
  machine.
- It does not configure PAM. Keep PAM changes separate from this driver fix.
- This is an out-of-tree Goodix driver; future AUR updates may overwrite the
  source build directory. Reapply the patch after updates until it is merged
  upstream.
