<link rel="stylesheet" type="text/css" href="styles.css">

---
layout: post
title: Creating an APFS Restore DMG from an iOS Root Filesystem Tar
date: 2026-04-14 16:43 +1300
description: Create a GUID/APFS DMG from an iOS root filesystem tar, resize it exactly, convert it to UDZO, and ASR-scan it.
---

# Creating an APFS restore-style DMG from an iOS root filesystem tar

This guide shows how to:

1. Create a blank writable DMG
2. Use a GUID partition map
3. Format the main partition as APFS
4. Extract an **iOS root filesystem tar** into the mounted APFS volume
5. Check the minimum possible image size
6. Resize the image to **minimum + 30 MiB**, using exact sector counts
7. Convert the result to **UDZO**
8. Run `asr imagescan`

This is useful when you want a restore-style image with a structure similar to Apple restore images: a GUID-partitioned DMG with an EFI partition and a main APFS partition containing an extracted iOS root filesystem.

## Requirements

You need:

- macOS
- `hdiutil`
- `asr`
- `tar`
- an **iOS root filesystem tar** archive

Examples in this guide use the following names:

- working writable image: `work-rw.dmg`
- final compressed image: `final.dmg`
- mounted volume name: `System`

You can rename the image files if you want, but this guide uses `System` as the APFS volume name throughout.

## Important concepts

### Blank image creation

When creating a blank image with `hdiutil create`, use:

- `-type UDIF` for a writable UDIF image
- `-layout GPTSPUD` for a GUID-partitioned single-partition layout
- `-fs APFS` for an APFS main partition

Do **not** use `-format UDRW` here. For blank image creation, `-type` is the correct option.

### Resizing

`hdiutil resize -limits` reports values in **512-byte sectors**.

If you want to resize an image to:

- **minimum size + 30 MiB**

then you must add exactly:

- `30 × 1024 × 1024 / 512 = 61440 sectors`

So the formula is:

```text
target_sectors = min_sectors + 61440
````

### Conversion

The working image should stay writable until:

* the iOS root filesystem tar has been extracted
* any final changes are complete
* the image has been resized

Only then should it be converted to:

* `UDZO` for a compressed read-only final DMG

### ASR scanning

Run `asr imagescan` on the **final UDZO image**, not the writable one.

## Step 1: Create the blank writable image

This creates a 5 GiB writable image with:

* GUID partition map
* EFI system partition
* APFS main partition
* volume name `System`

```bash
hdiutil create \
  -size 5g \
  -type UDIF \
  -layout GPTSPUD \
  -fs APFS \
  -volname System \
  work-rw.dmg
```

### Notes

* `-size 5g` sets the total image size
* `-type UDIF` creates a writable UDIF image
* `-layout GPTSPUD` creates a GUID-partitioned layout suitable for a single main partition plus EFI
* `-fs APFS` formats the main partition as APFS
* `-volname System` sets the APFS volume name

If you need a different starting size, replace `5g` with whatever is appropriate.

## Step 2: Attach the writable image

Mount the image:

```bash
hdiutil attach -owners on work-rw.dmg
```

The main APFS volume should appear at:

```text
/Volumes/System
```

If you used a different volume name, use that path instead.

### Why `-owners on`?

This helps preserve normal ownership behavior on the mounted image, which is useful when extracting an iOS root filesystem tar that relies on ownership and permissions.

## Step 3: Extract the iOS root filesystem tar into the APFS volume

For a plain `.tar` archive:

```bash
sudo tar -xpf /path/to/rootfs.tar -C /Volumes/System
```

For a `.tar.gz` or `.tgz` archive:

```bash
sudo tar -xzpf /path/to/rootfs.tar.gz -C /Volumes/System
```

For a `.tar.xz` archive:

```bash
sudo tar -xJpf /path/to/rootfs.tar.xz -C /Volumes/System
```

### Notes

* `sudo` is usually appropriate if you want ownership and permissions preserved as accurately as possible
* `-C /Volumes/System` extracts directly into the mounted APFS volume
* make sure the target path matches the volume name you used when creating the image

At this point, the iOS root filesystem has been extracted into the main APFS partition of the DMG.

## Step 4: Detach the writable image

Before resizing or converting, unmount it:

```bash
hdiutil detach /Volumes/System
```

If the volume is busy, close any Finder windows or Terminal sessions using it, then try again.

## Step 5: Check the minimum size of the writable image

To see the minimum, current, and maximum size of the image:

```bash
hdiutil resize -limits work-rw.dmg
```

Example output:

```text
min      cur       max
7176192  7318848   7318848
```

These values are in **512-byte sectors**.

### Meaning of the columns

* `min` = the smallest the image can currently be resized to
* `cur` = the current size
* `max` = the largest allowed size in the current context

For this workflow, the important value is `min`.

## Step 6: Calculate exact size for minimum + 30 MiB

Because one sector is 512 bytes:

```text
30 MiB = 30 × 1024 × 1024 bytes
30 MiB = 31,457,280 bytes
31,457,280 / 512 = 61,440 sectors
```

So:

```text
target_sectors = min_sectors + 61440
```

### Example

If:

```text
min = 7176192
```

then:

```text
7176192 + 61440 = 7237632
```

So the exact resize target becomes:

```text
7237632 sectors
```

## Step 7: Resize the writable image exactly

Using the example above:

```bash
hdiutil resize -sectors 7237632 work-rw.dmg
```

In general:

```bash
hdiutil resize -sectors <min_plus_61440> work-rw.dmg
```

### Important

Resize the **writable** image, not the final compressed image.

This step should happen **before** conversion to UDZO.

## Step 8: Optionally verify the resized writable image

You can inspect the working image after resizing:

```bash
hdiutil imageinfo work-rw.dmg
```

Or re-check its bounds:

```bash
hdiutil resize -limits work-rw.dmg
```

This is optional, but useful if you want to confirm the image was resized exactly as expected.

## Step 9: Convert the writable image to UDZO

Once the iOS root filesystem is in place and resizing is complete, convert the image to a compressed read-only DMG:

```bash
hdiutil convert work-rw.dmg -format UDZO -o final.dmg
```

This produces the final compressed image:

* read-only
* zlib-compressed
* suitable for ASR scanning

## Step 10: Run ASR image scan

Now scan the final image:

```bash
sudo asr imagescan --source final.dmg
```

This prepares the image for restore-style use.

## Step 11: Verify the finished image

Inspect the final image:

```bash
hdiutil imageinfo final.dmg
```

A successful result should show, at minimum:

* `Format Description: UDIF read-only compressed (zlib)`
* `Format: UDZO`
* `partition-scheme: GUID`
* an EFI System Partition
* a main `Apple_APFS` partition

The following values will vary depending on the extracted iOS root filesystem:

* checksums
* partition UUIDs
* compression ratio
* compressed byte count
* resize limits

That is normal.

## Step 12: Confirm ASR validity

To check the image after scanning:

```bash
asr info --source final.dmg
```

If it reports the image is valid as an ASR source, the process is complete.

# Full example workflow

```bash
hdiutil create \
  -size 5g \
  -type UDIF \
  -layout GPTSPUD \
  -fs APFS \
  -volname System \
  work-rw.dmg

hdiutil attach -owners on work-rw.dmg

sudo tar -xpf /path/to/rootfs.tar -C /Volumes/System

hdiutil detach /Volumes/System

hdiutil resize -limits work-rw.dmg
# Read the "min" value, then add 61440 sectors

hdiutil resize -sectors <min_plus_61440> work-rw.dmg

hdiutil convert work-rw.dmg -format UDZO -o final.dmg

sudo asr imagescan --source final.dmg

hdiutil imageinfo final.dmg
asr info --source final.dmg
```

# Exact sizing reference

## Sector size

`hdiutil resize` uses:

```text
1 sector = 512 bytes
```

## 30 MiB in sectors

```text
30 MiB = 30 × 1024 × 1024 = 31,457,280 bytes
31,457,280 / 512 = 61,440 sectors
```

## Formula

```text
target_sectors = min_sectors + 61440
```

# Troubleshooting

## Error: `create: -format requires -srcfolder or -srcdevice`

Cause:

You used `-format` when creating a blank image.

Fix:

Use `-type UDIF` instead.

Correct:

```bash
hdiutil create -size 5g -type UDIF -layout GPTSPUD -fs APFS -volname System work-rw.dmg
```

Incorrect:

```bash
hdiutil create -size 5g -format UDRW ...
```

## `hdiutil detach` says the volume is busy

Cause:

Something is still using the mounted image.

Fix:

* close Finder windows showing the volume
* leave the mounted directory in Terminal
* stop any process using files inside the volume
* retry the detach command

## The final image is bigger than expected

Cause:

The initial image may have been larger than necessary, and the resize step may have been skipped.

Fix:

Run:

```bash
hdiutil resize -limits work-rw.dmg
```

then resize to:

```text
min + 61440 sectors
```

before converting to UDZO.

## The image structure does not exactly match another DMG

Cause:

Checksums, UUIDs, partition lengths, compression ratio, and some metadata depend on the extracted filesystem contents.

Fix:

Compare the important structural properties instead:

* `Format: UDZO`
* GUID partition map
* EFI System Partition
* main APFS partition
* successful `asr imagescan`

Those are the meaningful indicators.

# Summary

The reliable workflow is:

1. Create a blank writable APFS/GUID image
2. Attach it
3. Extract the iOS root filesystem tar into the mounted APFS volume
4. Detach it
5. Check the minimum size with `hdiutil resize -limits`
6. Add `61440` sectors for exactly `30 MiB`
7. Resize with `hdiutil resize -sectors`
8. Convert to `UDZO`
9. Run `asr imagescan`
10. Verify with `hdiutil imageinfo` and `asr info`

The exact sector formula is:

```text
target_sectors = min_sectors + 61440
```
