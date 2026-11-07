# SD Card App Data (sd_appdata)

Magisk module for the Redmi Pad SE (HyperOS, Android 14) that moves app data to the external SD card, to reduce writes to the internal eMMC.

## How it works

SD cards are exFAT/FAT32, which can't hold Android's file owners, permissions and SELinux labels. So the module:

1. Creates an ext4 image file on the SD (`<SD>/.sd_appdata/appdata.img`).
2. Mounts it at `/mnt/sdappdata` using a loop device.
3. Copies each selected app's data into the image (first time only).
4. Bind-mounts the image folders back over the original paths, so the app still sees its normal folders but all reads and writes go to the SD.

Paths moved per app:

| Original path | Content |
|---|---|
| `/data/data/<pkg>` | App data (CE) |
| `/data/user_de/0/<pkg>` | Device-encrypted data (`MOVE_DE=1`) |
| `/sdcard/Android/data/<pkg>` | External app data (`MOVE_ANDROID_DATA=1`) |

The module also mounts into `/data_mirror` and zygote's mount namespace, so newly launched apps see the SD data.

## Install

Flash `sd_appdata.zip` in Magisk and reboot. Wait about a minute after unlocking: the module waits for boot, unlock and the SD card.

## Config (`/data/adb/modules/sd_appdata/config.txt`)

```
IMAGE_SIZE_MB=16384          # image size, used only when the image is first created
MODE=user                    # user = all third-party apps, list = only PACKAGES
PACKAGES="mark.via.gp"       # separate packages with spaces (or newlines)
EXCLUDE="com.topjohnwu.magisk"
MOVE_DE=1
MOVE_ANDROID_DATA=1
```

Add a second app:

```
MODE=list
PACKAGES="mark.via.gp com.second.app"
```

Then reboot. System apps (like the official YouTube) are skipped by `MODE=user`; use `MODE=list` for them.

## Image size

- The size is fixed when the image is created. Changing `IMAGE_SIZE_MB` later does nothing to an existing image.
- It must fit on the SD. For a 1 GB SD use something like `IMAGE_SIZE_MB=900`. On FAT32 it is capped to 4000 MB.
- All moved apps share the image's free space. When it is full, apps get "no space left" and may crash or lose data. Internal storage and the system are not affected.
- Check usage: `su -c 'df -h /mnt/sdappdata'`
- Keep the image small-app only on a small SD. Games like Minecraft fill it fast.

### Change the size without losing data

1. `su -c sh /data/adb/modules/sd_appdata/restore.sh`
2. Delete `.sd_appdata/appdata.img` on the SD.
3. Set the new `IMAGE_SIZE_MB` in `config.txt`.
4. Reboot.

### Swapping or wiping the SD

A new or wiped SD has no image, so the module creates a fresh empty one. Data that was only in the old image is lost, and apps start as if freshly installed. Run `restore.sh` before swapping.

## What happens on each reboot

Nothing is recopied or reformatted. The module mounts the existing image and binds the folders. An app is copied only the first time, when it has no folder in the image. Apps are force-stopped before being mounted, so one may restart once at boot.

If the SD is missing at boot, the module does nothing and the apps run with empty data until the SD is back.

## Verify it works (Termux)

```
su -c 'cat /data/adb/modules/sd_appdata/log.txt'
su -c 'df -h /mnt/sdappdata'
su -c 'grep mark.via.gp /proc/self/mountinfo'
su -c 'ls /mnt/sdappdata/data'
```

- The log should contain `SD:`, `image mounted`, `moved ...` and `done`.
- The `grep` should print lines with `loop` and `/mnt/sdappdata` style roots (`/data/<pkg>`, `/data_de/<pkg>`, `/android_data/<pkg>`).
- Harmless log lines: `mke2fs: invalid option -- t` (first formatter fails, the fallback works) and `cmd: Failure calling service activity` (force-stop failed right after boot; reboot once).

## Downloads

Only app data is moved, not shared storage.

- Files saved with a browser's download button (like Via) go to `/sdcard/Download` and stay on internal storage. Change the download folder in the browser's settings to the SD card (the browser needs "All files access").
- Offline videos that a website or app stores inside its own storage (browser storage, the YouTube app's offline downloads) are in app data, so they are moved if the app is selected.

## Restore / uninstall

Run as root, then reboot:

```
su -c sh /data/adb/modules/sd_appdata/restore.sh
```

It copies everything back to internal storage, unmounts the image and disables the module. Don't just remove the module, or the data stays on the SD and the apps appear empty.

## Where the features come from

- **Linux kernel:** loop device, ext4, bind mounts, mount namespaces.
- **Android:** vold mounts the SD at `/mnt/media_rw/...`; `/data_mirror` and `/mnt/androidwritable` provide the per-app storage views (Android 11+); SELinux labels are copied so apps can access the files; `pm` and `am` list and force-stop apps.
- **Magisk:** runs the script as root at boot.

Nothing in the kernel or system partition is modified.

## Speed

The `.img` is slightly slower than writing to the SD directly, and the SD is slower than the internal eMMC. The goal is fewer eMMC writes, not speed. A real ext4 partition on the SD would be a bit faster but needs repartitioning.

## Files

- `service.sh` main logic, runs at boot
- `common.sh` mount helpers
- `config.txt` settings
- `restore.sh` move everything back
- `log.txt` last boot log
