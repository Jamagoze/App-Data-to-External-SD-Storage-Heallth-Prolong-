# SD Card App Data (sd_appdata)

Magisk module for the Redmi Pad SE (HyperOS, Android 14) that moves app data to the external SD card, to reduce writes to the internal eMMC.

## How it works

SD cards are exFAT/FAT32, which can't store Android's file owners, permissions and SELinux labels. So the module:

1. Creates an ext4 image on the SD (`<SD>/.0.img`).
2. Mounts it at `/mnt/sdappdata` using a loop device.
3. Protects the image file from deletion (see below).
4. Copies each selected app's data into the image (first time only).
5. Bind-mounts the image folders back over the original paths, so the app still sees its normal folders but all reads and writes land on the SD.

Paths moved per app:

| Original path | Content |
|---|---|
| `/data/data/<pkg>` | App data (CE) |
| `/data/user_de/0/<pkg>` | Device-encrypted data (`MOVE_DE=1`) |
| `/sdcard/Android/data/<pkg>` | External app data (`MOVE_ANDROID_DATA=1`) |

The module also mounts into `/data_mirror` and zygote's mount namespace, so newly launched apps see the SD data. Your file manager shows the same `/Android/data/<pkg>` path as before, because the SD folder is mounted over it.

## Install

Flash `sd_appdata.zip` in Magisk and reboot. Wait about a minute after unlocking: the module waits for boot, unlock and the SD card. The first boot can take a long time if an app has a lot of data (several GB), because it is copied to the SD. Don't reboot until the log shows `done`.

## Direct read and write

Nothing is staged on internal storage. Once an app is set up, its reads and writes go straight to the image on the SD. Writes are held in RAM briefly and flushed to the SD in batches, which doesn't touch the eMMC. Internal storage is used only for the one-time first copy, after which the internal originals are deleted.

Anything an app saves outside its own folders still goes to internal storage, for example files saved to `/sdcard/Download`.

## Image protection

While the module is active, `.0.img` is bind-mounted onto itself. File managers and apps that try to delete it get "device or resource busy".

### Check it without deleting

```
su -c 'grep "\.0\.img" /proc/self/mountinfo'
su -c 'grep "\.0\.img" /proc/$(pidof zygote64)/mountinfo'
```

Both should print lines with `/mnt/media_rw/<SD id>/.0.img` (and a `/mnt/pass_through/...` one). The zygote line is what apps and file managers inherit. If nothing prints, the protection isn't active; check that the log has `image mounted` and `done`.

A rename test also works. It fails with "Device or resource busy" when protected, but renames the file if it isn't, so move it back right away:

```
su -c 'mv /mnt/media_rw/<SD id>/.0.img /mnt/media_rw/<SD id>/.0.img.x'
```

### Limits

- The protection is only a temporary mount that exists while the module is running. Nothing is saved on the SD to block deletion.
- After the module is removed or disabled, or after a reboot without the module active, the file can be deleted like any other.
- On another device (or a PC), `.0.img` is an ordinary hidden file: it can be deleted, copied or renamed, and formatting the SD removes it.
- The data inside is not encrypted. Anyone can mount the image on a Linux PC and read it.
- If the SD is lost or the image is deleted while the module is installed, the moved app data is lost. The module creates a new empty image and the app starts fresh.

## Removing the module

Remove it in the Magisk app, then wait. The module notices the removal and:

1. Checks that internal storage has enough free space for all the data.
2. Copies every app's data back to internal storage.
3. Unmounts and deletes `.0.img`.

Wait until the log says `image deleted - you can reboot now`, then reboot:

```
su -c 'tail -n 3 /data/adb/modules/sd_appdata/log.txt'
```

Rebooting before that leaves the app data half restored. If internal storage doesn't have enough space, or a copy fails, the removal is cancelled and the image is kept, and the log says so. Free some space and remove again. The Magisk app may still show the module as pending removal; check the log.

## Config (`/data/adb/modules/sd_appdata/config.txt`)

```
IMAGE_SIZE_MB=16384          # image size, used only when the image is first created
MODE=list                    # user = all third-party apps, list = only PACKAGES
PACKAGES="mark.via.gp"       # separate packages with spaces (or newlines)
EXCLUDE="com.topjohnwu.magisk"
MOVE_DE=1
MOVE_ANDROID_DATA=1
```

Add a second app: `PACKAGES="mark.via.gp com.second.app"` and reboot. Only spaces (or newlines) separate packages; commas don't work. System apps (like the official YouTube) are skipped by `MODE=user`; use `MODE=list` for them. Don't add WebView: it is a shared system component, and each app's WebView data is already inside that app's own data.

## Image size and resizing

- The size is fixed when the image is created. Changing `IMAGE_SIZE_MB` and rebooting does nothing to an existing image.
- It must fit on the SD. For a 1 GB SD use something like `IMAGE_SIZE_MB=900`. On FAT32 it is capped to 4000 MB.
- The new size must hold all the moved data with room to spare. For example, 4.7 GB of app data won't fit in a 4 GB image.
- All moved apps share the image's free space. When it is full, apps get "no space left" and may crash or lose data. Internal storage and the system are not affected.
- Check usage: `su -c 'df -h /mnt/sdappdata'`

### Change the size without losing data

1. `su -c sh /data/adb/modules/sd_appdata/restore.sh` (needs enough free internal space for the data).
2. Delete `.0.img` on the SD (the restore script unprotects it).
3. Set the new `IMAGE_SIZE_MB` in `config.txt`, and enable the module again if it shows disabled.
4. Reboot.

### Swapping or wiping the SD

A new or wiped SD has no image, so the module creates a fresh empty one. Data that was only in the old image is lost. Remove the module or run `restore.sh` before swapping.

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
- `df` should show `/dev/block/loopNN` mounted, with used space growing while the first copy runs.
- The `grep` should print lines with the loop device and roots like `/data/<pkg>`, `/data_de/<pkg>`, `/android_data/<pkg>`.
- Harmless log lines: `mke2fs: invalid option -- t` (first formatter fails, the fallback works) and `cmd: Failure calling service activity` (force-stop failed right after boot; reboot once).
- If you paste command output back into Termux, you'll get `bash:` errors. Ignore them.

## Downloads

Only app data is moved, not shared storage.

- Files saved with a browser's download button (like Via) go to `/sdcard/Download` and stay on internal storage. Change the download folder in the browser's settings to the SD card (the browser needs "All files access").
- Offline videos that a website or app stores inside its own storage are in app data, so they are moved if the app is selected.

## Manual restore

```
su -c sh /data/adb/modules/sd_appdata/restore.sh
```

Copies everything back to internal storage, unprotects and unmounts the image, and disables the module. It does not delete the image. Reboot afterwards. Just removing the module without waiting for the automatic restore leaves the data on the SD and the apps appear empty.

## Speed and fragmentation

- The `.img` is slightly slower than writing to the SD directly, and the SD is slower than the internal eMMC. The goal is fewer eMMC writes, not speed.
- The SD is mounted through FUSE (`fuseblk`), so every read of the image goes through a userspace driver, which adds overhead.
- Fragmentation is not a big issue. The image file has a fixed size and its position on the card is decided once, when it is created. Inside the image, ext4 can fragment, but flash storage has no seek penalty. Small random reads are slow on SD cards regardless.
- The image name or folder depth (`.0.img` vs a longer path) has no effect on speed. It is only used once at boot.

## Compatibility

Not HyperOS-specific; it uses standard Android and Linux features. Only the Redmi Pad SE has been tested.

- Magisk 20.4+ (KernelSU or APatch may need different paths).
- Android 11+ (uses the per-app storage views).
- Kernel with loop device and ext4 support.
- An SD card that Android mounts under `/mnt/media_rw/`.
- An `mke2fs` binary (Android ships one in `/system/bin`).

## Where the features come from

- **Linux kernel:** loop device, ext4, bind mounts, mount namespaces.
- **Android:** vold mounts the SD at `/mnt/media_rw/...`; `/data_mirror` and `/mnt/androidwritable` provide the per-app storage views (Android 11+); SELinux labels are copied so apps can access the files; `pm` and `am` list and force-stop apps.
- **Magisk:** runs the script as root at boot and creates the `remove` flag when you remove the module.

Nothing in the kernel or system partition is modified.

## Files

- `service.sh` main logic, runs at boot and waits for removal
- `common.sh` mount, protect and restore helpers
- `config.txt` settings
- `restore.sh` manual restore
- `log.txt` last boot log

## Version history

### v1.1
- `.0.img` is protected from deletion while the module is active (bind-mounted onto itself).
- Removing the module in Magisk now restores all app data to internal storage automatically, then deletes `.0.img`. It checks free internal space first and cancels the removal if the data won't fit or a copy fails.
- `restore.sh` shares the same restore code: it checks free space, restores SELinux labels, unprotects the image, and disables the module only after a successful restore.
- An intermediate build also labelled v1.1 (clean install: no copy, no restore) was dropped. This version replaces it.

### v1.0
- First release: ext4 image on the SD, app data copied into it, bind mounts over the original paths, and a manual `restore.sh`.
- Image first at `<SD>/.sd_appdata/appdata.img`, then renamed to `<SD>/.0.img` (same version number, no other change).
