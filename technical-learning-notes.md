from pathlib import Path

content = r"""# Technical Learning Notes
## Android ROM Extraction, WSL 2, Ubuntu, EROFS, and Virtual Disk Cleanup

**Purpose:** A personal reference documenting the tools, concepts, commands, errors, and solutions learned while extracting partitions from an Android ROM on Windows.

---

## 1. Project Overview

### Goal

The goal of this exercise was to inspect and extract files from an Android ROM package on a Windows computer. The ROM archive contained Android partition images, including a large `super.img`. The work involved:

1. Extracting the ROM archive.
2. Unpacking Android dynamic partitions from `super.img`.
3. Identifying the filesystem used inside a partition image.
4. Using Linux tools through WSL 2 to extract an EROFS filesystem.
5. Copying extracted files back to a Windows drive.
6. Understanding where WSL stores its Linux filesystem.
7. Cleaning up temporary Linux files and compacting the WSL virtual disk.

### Environment

- **Host operating system:** Windows
- **Linux environment:** Ubuntu through WSL 2
- **Android device context:** POCO F3 (`alioth`)
- **ROM package:** `JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM.zip`
- **Main partition tool:** `lpunpack` for Windows AMD64
- **Filesystem tool:** `erofs-utils` / `fsck.erofs`
- **Disk management tool:** Windows DiskPart

> These notes document the commands and results from this particular exercise. Exact paths, partition sizes, and results will differ between ROMs and computers.

---

## 2. Important Concepts

### 2.1 What is an Android ROM?

An Android ROM package contains software used to install or update Android on a device. Depending on the package, it may include partition images, installation scripts, metadata, firmware components, and other files.

A ROM package is not necessarily one single filesystem image. It can contain many images, each serving a different purpose.

### 2.2 What is a partition image (`.img`)?

A partition image is a file representing data arranged for a storage partition or partition-like image. The `.img` extension does **not** tell you which filesystem it contains.

Examples include:

- `system_a.img`
- `vendor_a.img`
- `product_a.img`
- `odm_a.img`
- `mi_ext_a.img`

Some images contain Android filesystem data. Others may use different formats or serve different purposes. Choose the extraction tool after identifying the image's actual format.

### 2.3 What is `super.img`?

Modern Android devices may use **dynamic partitions**. Instead of requiring every logical partition to have its own fixed physical partition on the device, Android can store multiple logical partitions inside a larger container partition called `super`.

Depending on the ROM, logical partitions can include:

- `system`
- `vendor`
- `product`
- `system_ext`
- `odm`
- `mi_ext`
- related partitions such as `vendor_dlkm` or `system_dlkm`

The `_a` suffix in a filename such as `system_a.img` commonly refers to slot A in an A/B-style partition arrangement. The exact partition layout depends on the device and ROM.

**Key distinction:** `super.img` is a container for logical partitions. It is not the same thing as the `system` filesystem itself.

### 2.4 What is `lpunpack`?

`lpunpack` reads Android dynamic-partition metadata from a suitable `super` image and writes the logical partition images into separate files.

In this project, the Windows AMD64 executable was:

```text
E:\poco_f3\extsu\lpunpack-windows-amd64.exe
```

The command used was:

```powershell
lpunpack-windows-amd64.exe -v "E:\poco_f3\JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM\joeroms\super\super" "E:\poco_f3\JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM\joeroms\super_unpacked"
```

The first argument after `-v` is the input image; the final argument is the output directory.

**Important:** The file called `super` in this command was the expanded/raw super image created during the earlier extraction workflow. File extensions are not always reliable indicators of format.

### 2.5 What is EROFS?

**EROFS** stands for *Enhanced Read-Only File System*. It is a filesystem used in Android and other Linux-based environments. Android partition images may use EROFS or other filesystem formats.

A file such as `product_a.img` can contain an EROFS filesystem even though Windows Explorer or a normal archive tool does not treat it as a conventional ZIP archive.

For this particular image, an EROFS signature was observed at offset `0x400` (1024 bytes). That observation helped identify the filesystem format.

Do not assume every Android `.img` is EROFS. Identify the format first.

### 2.6 What is `fsck.erofs`?

`fsck.erofs` is a tool included in `erofs-utils`. Despite the `fsck` name, it can also be used to inspect an EROFS image and extract its files.

The help output showed support for extraction using the `--extract` option.

### 2.7 What is WSL 2?

**Windows Subsystem for Linux (WSL)** lets you run a Linux environment alongside Windows without setting up a traditional full Linux desktop installation.

WSL 2 uses a lightweight managed virtual machine and a real Linux kernel. You can use Linux commands and tools while still using your normal Windows desktop.

It is not exactly the same as running a full VirtualBox virtual machine:

- WSL is integrated closely with Windows.
- Linux can access Windows drives through paths such as `/mnt/c` and `/mnt/e`.
- Windows can access Linux files through the WSL network namespace, commonly via `\\wsl$\Ubuntu\...` or `\\wsl.localhost\Ubuntu\...`.
- WSL 2 stores its Linux filesystem in a virtual disk file.

WSL is useful for development, scripting, Linux utilities, and Android image analysis.

### 2.8 What is `ext4.vhdx`?

WSL 2 commonly stores a Linux distribution's filesystem inside a virtual hard disk file with the `.vhdx` extension. In this installation, the file was found at:

```text
C:\Users\SM\AppData\Local\wsl\{5924d7a2-31d9-4fbd-9f60-c871627b8c1c}\ext4.vhdx
```

This VHDX file holds the Ubuntu Linux filesystem and its files. It is **not** the same thing as the Android ROM's `super.img`.

The VHDX file can grow when Linux files are added. Deleting files inside Ubuntu may free space inside the Linux filesystem without immediately shrinking the physical VHDX file on Windows. Compaction can reclaim some of that host-side space.

**Never delete `ext4.vhdx` manually as a cleanup method.** Doing so can destroy the WSL distribution's data.

---

## 3. Folder and File Layout

The main ROM working directory was:

```text
E:\poco_f3\JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM\
```

The extracted ROM content included:

```text
joeroms\
├── super\
│   └── super
└── super_unpacked\
    ├── mi_ext_a.img
    ├── odm_a.img
    ├── product_a.img
    ├── system_a.img
    ├── system_dlkm_a.img
    ├── system_ext_a.img
    ├── vendor_a.img
    └── vendor_dlkm_a.img
```

After successful EROFS extraction, the product files were copied into a Windows folder named:

```text
E:\poco_f3\JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM\joeroms\product_unpacked\
```

The exact folder structure may vary if the ROM package is updated or extracted differently.

---

## 4. Unpacked Logical Partition Images

The `lpunpack` operation produced the following images:

| Partition image | Size in bytes |
|---|---:|
| `mi_ext_a.img` | 1,453,830,144 |
| `odm_a.img` | 300,945,408 |
| `product_a.img` | 4,099,530,752 |
| `system_a.img` | 849,666,048 |
| `system_dlkm_a.img` | 45,056 |
| `system_ext_a.img` | 711,184,384 |
| `vendor_a.img` | 966,352,896 |
| `vendor_dlkm_a.img` | 45,056 |

The original `super.img` was about 6.0 GB, and the expanded/raw image was approximately 9.13 GB. Image sizes can differ depending on whether the file is sparse, expanded, compressed, or otherwise packaged.

Some small images may be metadata-oriented or have special layout details; do not assume a 45,056-byte file is a full ordinary filesystem without inspecting it.

---

## 5. Why 7-Zip Did Not Properly Extract `product_a.img`

I first tried opening `product_a.img` with 7-Zip. It detected something resembling an embedded ZIP fragment, but that did not provide the actual Android product filesystem.

### Why this can happen

A filesystem image is not necessarily an archive. An archive tool may occasionally identify data patterns inside a binary image without understanding the image's filesystem layout.

The file extension `.img` alone does not identify the internal format.

### What I learned

1. Do not rely only on the extension.
2. Do not assume a partition image is a ZIP file.
3. Identify the filesystem before selecting a tool.
4. For EROFS images, use an EROFS-aware utility such as `fsck.erofs`.

---

## 6. Installing Ubuntu Through WSL

Ubuntu was installed with this command from Windows PowerShell or Command Prompt:

```powershell
wsl --install -d Ubuntu
```

The installation prompted for a Linux username and password. The username used was:

```text
sm
```

The Linux password is separate from the Windows login password. When entering a password in a Linux terminal, the cursor usually does not show letters or asterisks. This is normal.

If the password is mistyped during setup, follow the available setup/reset process rather than repeatedly guessing.

### Useful WSL commands

Run these from **Windows PowerShell or Command Prompt**:

```powershell
wsl --list --verbose
```

Lists installed distributions and whether they use WSL 1 or WSL 2.

```powershell
wsl --status
```

Shows general WSL status information.

```powershell
wsl -d Ubuntu
```

Launches the Ubuntu distribution explicitly.

```powershell
wsl --update
```

Updates WSL when an update is available.

```powershell
wsl --shutdown
```

Stops running WSL distributions and the WSL virtual machine. This is useful before virtual-disk maintenance.

```powershell
wsl ~
```

Opens the default WSL distribution at its Linux home directory. If the default distribution is not Ubuntu, use `wsl -d Ubuntu` to select Ubuntu explicitly.

---

## 7. Understanding the Two Environments: Windows and Ubuntu

Commands must be run in the correct terminal.

### Windows PowerShell

Typical prompt:

```text
PS C:\Users\SM>
```

Use it for Windows paths and tools such as PowerShell commands, Windows executables, and DiskPart.

### Ubuntu terminal inside WSL

Typical prompt:

```text
sm@SM-PC:/mnt/c/Users/SM$
```

Use it for Linux commands such as `sudo`, `apt`, `ls`, `du`, `cp`, and `fsck.erofs`.

### Windows drives inside Ubuntu

WSL normally mounts Windows drives under `/mnt`:

| Windows path | Ubuntu path |
|---|---|
| `C:\Users\SM` | `/mnt/c/Users/SM` |
| `E:\poco_f3` | `/mnt/e/poco_f3` |

For example, this Windows path:

```text
E:\poco_f3\example\product_a.img
```

is typically accessed in Ubuntu as:

```text
/mnt/e/poco_f3/example/product_a.img
```

Linux uses forward slashes (`/`) in paths.

### Where are Ubuntu's own files?

The Linux home directory for the `sm` user is:

```bash
/home/sm
```

A shortcut for the current user's home directory is:

```bash
~
```

Files stored under `~/...` belong to the Ubuntu Linux filesystem. Files under `/mnt/e/...` are on the Windows E: drive.

---

## 8. Installing EROFS Utilities

The package name was initially mistyped as `erofs-utills`. The correct package name is:

```bash
erofs-utils
```

In the Ubuntu terminal:

```bash
sudo apt update
sudo apt install erofs-utils
```

- `sudo` runs a command with administrator/root privileges.
- `apt update` refreshes the package index.
- `apt install` installs a package.
- Ubuntu may request the Linux user's password for `sudo`.

To view the tool's supported options:

```bash
fsck.erofs --help
```

The installed version's help output should be treated as the authority for which options are available.

---

## 9. Extracting `product_a.img` With EROFS Utilities

### 9.1 The first attempt and the problem

The first extraction attempt wrote the output directly to a Windows-mounted directory under `/mnt/e`. It failed with an error similar to:

```text
failed to set permissions ... (Operation not permitted)
```

### Why it failed

Linux filesystems support permissions and metadata that do not always map directly to Windows-mounted filesystems. Some Linux tools try to set file permissions or metadata during extraction. A mounted Windows directory may not support those operations in the way the tool expects.

### 9.2 The successful approach

Extract into Ubuntu's own Linux filesystem first.

In the Ubuntu terminal:

```bash
mkdir -p ~/rom_extract/product_unpacked
```

This creates the destination directory. `-p` also creates missing parent directories and avoids an error if the directory already exists.

Then extract the EROFS image:

```bash
fsck.erofs --extract="$HOME/rom_extract/product_unpacked" "/mnt/e/poco_f3/JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM/joeroms/super_unpacked/product_a.img"
```

Explanation:

- `fsck.erofs` invokes the EROFS utility.
- `--extract=...` specifies the output directory.
- `$HOME` expands to the current Linux user's home directory, `/home/sm`.
- The quoted `/mnt/e/.../product_a.img` is the input image on the Windows E: drive.
- Quotation marks help protect paths containing spaces.

The extraction succeeded. The output contained directories such as:

```text
ai
app
bin
data-app
etc
fonts
framework
lib
lib64
media
opcust
overlay
prebuilts
priv-app
usr
vm-system
```

This structure was evidence that the image's files had been extracted rather than merely opened as an archive.

### 9.3 Copying the extracted files back to Windows

After extraction, the files could be copied from Ubuntu's filesystem to the Windows E: drive.

In the Ubuntu terminal:

```bash
mkdir -p "/mnt/e/poco_f3/JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM/joeroms/product_unpacked"
```

Then:

```bash
cp -a ~/rom_extract/product_unpacked/. "/mnt/e/poco_f3/JoeROM_Nezha_on_Alioth_EU_OS3.0.307.0.WPAKCNXM/joeroms/product_unpacked/"
```

Explanation:

- `cp` copies files and directories.
- `-a` asks `cp` to copy recursively while preserving as much metadata as supported.
- `source/.` means copy the contents of the source directory, including hidden entries, rather than creating an extra nested `product_unpacked` directory.
- The destination is the Windows E: drive mounted in Ubuntu.

After copying, inspect the Windows destination in File Explorer to confirm the expected folders and files are present.

**Important:** The `cp -a` option cannot preserve every Linux metadata attribute on every Windows-mounted filesystem. It is suitable for moving the extracted file tree for inspection, but it is not a guarantee of an exact Linux-filesystem clone.

---

## 10. Understanding the Ubuntu Virtual Disk Size

The WSL virtual disk was found at:

```text
C:\Users\SM\AppData\Local\wsl\{5924d7a2-31d9-4fbd-9f60-c871627b8c1c}\ext4.vhdx
```

Before cleanup and compaction, its reported size was:

```text
7,864,320,000 bytes
```

This is approximately:

- 7.86 decimal GB, or
- 7.32 GiB.

The exact number displayed depends on the unit used by the tool. Windows PowerShell's `1GB` arithmetic constant is based on 1,073,741,824 bytes, which is technically a GiB-sized unit even though the variable name is `GB`.

### Why can the VHDX remain large after deleting files?

Think of the VHDX as a large container holding Ubuntu's virtual filesystem.

Deleting files inside Ubuntu frees space **inside** that filesystem. However, the VHDX file on Windows may not automatically shrink to match the amount of live data. Previously allocated blocks can remain allocated in the host file.

This is why there are two related but different measurements:

1. **Free space inside Ubuntu:** how much space the Linux filesystem reports as available.
2. **VHDX file size on Windows:** how much space the virtual disk file currently occupies on the Windows drive.

They do not always change by the same amount.

---

## 11. Cleaning Temporary Files Inside Ubuntu

The large temporary extraction directory was no longer needed after the files had been copied to Windows. The temporary Linux copy was removed or confirmed absent.

### Check whether the temporary directory exists

In Ubuntu:

```bash
du -sh ~/rom_extract 2>/dev/null
```

- `du` reports disk usage.
- `-s` reports a summary.
- `-h` displays human-readable units.
- `2>/dev/null` hides error messages sent to standard error, such as the message shown if the directory does not exist.

If this command produces no output, that does not prove every temporary file has been deleted. It may simply mean that this particular directory is absent or could not be read.

### Remove a temporary directory only after confirming the Windows copy

If the extracted files are safely copied and verified on Windows, the temporary Linux directory can be removed with:

```bash
rm -rf ~/rom_extract
```

**Caution:** `rm -rf` permanently removes the specified directory and its contents without prompting. Check the path carefully before running it. Never run a broad or uncertain deletion command as a disk-cleanup shortcut.

### Optional: clear the APT package cache

In Ubuntu:

```bash
sudo apt clean
```

This clears downloaded package files from APT's cache. It does not uninstall installed packages.

Optionally, remove packages that APT considers no longer needed:

```bash
sudo apt autoremove
```

Read the proposed package list before confirming. Do not approve removal if you do not understand what is listed.

---

## 12. What `fstrim` Does

The command run in Ubuntu was:

```bash
sudo fstrim -av
```

The output included:

```text
/: 1005.2 GiB (1079319896064 bytes) trimmed on /dev/sde
```

### What does TRIM mean?

TRIM/discard tells the underlying storage system which blocks are no longer in use by the filesystem. This information can help the storage layer reclaim or optimize unused blocks.

The options mean:

- `sudo`: run with elevated privileges.
- `fstrim`: request discard/TRIM for eligible unused blocks on mounted filesystems.
- `-a`: process eligible mounted filesystems.
- `-v`: display a report.

### Important interpretation

The reported `1005.2 GiB` is the amount of discardable space reported as trimmed by that filesystem operation. It is **not** proof that Windows recovered 1005.2 GiB of physical disk space, and it is not the size of the VHDX shrink.

In a virtualized filesystem, reported capacity and discardable ranges may look surprising. Judge the actual host-side result by checking the VHDX file size before and after compaction.

---

## 13. Why the Wrong WSL Environment Opened

Running:

```powershell
wsl
```

opened a prompt resembling:

```text
SM-PC:/mnt/host/c/Users/SM#
```

The environment was `docker-desktop`, not the intended Ubuntu distribution. Its filesystem capacity was very small compared with Ubuntu's.

### Why did that happen?

The bare `wsl` command launches the default WSL distribution. If Docker Desktop or another distribution is configured as the default, that environment can open instead of Ubuntu.

### How to check the installed distributions

Run in Windows PowerShell:

```powershell
wsl --list --verbose
```

The list identifies installed distributions and their WSL versions. The default distribution is normally marked with an asterisk.

### How to launch Ubuntu explicitly

Use:

```powershell
wsl -d Ubuntu
```

To check disk usage inside Ubuntu:

```powershell
wsl -d Ubuntu -- df -h /
```

Or, after entering Ubuntu:

```bash
df -h /
```

`df -h /` reports the capacity, used space, and available space for the filesystem containing `/`.

**Lesson:** When performing Linux maintenance, confirm that you are in the intended distribution. Do not assume every terminal launched by `wsl` is Ubuntu.

---

## 14. Compacting the WSL Virtual Disk With DiskPart

After cleanup and `fstrim`, the Ubuntu WSL instance was shut down so its virtual disk could be compacted.

### 14.1 Shut down WSL

Run in Windows PowerShell or Command Prompt:

```powershell
wsl --shutdown
```

This stops running WSL distributions and the WSL virtual machine. It is important because a virtual disk generally should not be compacted while it is actively being used.

### 14.2 Start DiskPart as Administrator

Open **Windows Terminal or PowerShell as Administrator**, then run:

```text
diskpart
```

DiskPart is a Windows command-line utility for managing disks and volumes. Use it carefully; some DiskPart commands can modify or erase data.

### 14.3 Select the exact VHDX

At the DiskPart prompt, the command used was:

```text
select vdisk file="C:\Users\SM\AppData\Local\wsl\{5924d7a2-31d9-4fbd-9f60-c871627b8c1c}\ext4.vhdx"
```

This selects the specified virtual hard disk file. Confirm the path matches the actual VHDX on your computer.

### 14.4 Compact the selected virtual disk

```text
compact vdisk
```

This requests compaction of the selected virtual disk. The amount reclaimed depends on its allocation state, the virtual-disk format, and whether unused blocks were made available for reclamation.

Then exit DiskPart:

```text
exit
```

### Why use DiskPart?

The PowerShell command `Optimize-VHD` was checked but was not available in that environment. DiskPart provided an alternative method that successfully compacted the VHDX in this case.

Do not run `compact vdisk` against an uncertain or unintended virtual disk. Select and verify the exact VHDX first.

---

## 15. Result: Before and After

The Windows PowerShell check reported:

| Measurement | Approximate size |
|---|---:|
| VHDX before compaction | 7.32 GiB |
| VHDX after compaction | 2.06 GiB |
| Approximate space reclaimed | 5.26 GiB |

The figures are rounded. The reduction was approximately 5.26 GiB in the host-side VHDX file.

This result shows that the cleanup, discard preparation, and compaction process reclaimed a substantial amount of Windows disk space.

### What remained intact?

The goal was to compact Ubuntu's virtual disk, not remove Ubuntu itself. The distribution remained installed. The ROM files on the Windows E: drive were separate from the Ubuntu VHDX and were not the target of the compaction operation.

A VHDX can grow again when new files are added to Ubuntu. That is normal.

---

## 16. Verification Commands

These checks are useful after maintenance.

### List distributions in Windows

```powershell
wsl --list --verbose
```

### Check Ubuntu's internal filesystem

```powershell
wsl -d Ubuntu -- df -h /
```

### Check the VHDX file size in PowerShell

Use the exact path for your installation:

```powershell
Get-Item "C:\Users\SM\AppData\Local\wsl\{5924d7a2-31d9-4fbd-9f60-c871627b8c1c}\ext4.vhdx" |
    Select-Object Name, @{Name="SizeGiB";Expression={[math]::Round($_.Length / 1GB, 2)}}
```

This PowerShell calculation divides bytes by 1,073,741,824, so the result is technically in GiB despite the familiar `GB` constant name.

If your WSL VHDX is stored at a different path, substitute that exact path.

---

## 17. Troubleshooting Summary

| Problem | Likely explanation | What helped |
|---|---|---|
| 7-Zip did not properly extract `product_a.img` | It was a filesystem image, not a normal ZIP archive | Identify the filesystem and use an EROFS-aware tool |
| `erofs-utills` package was not found | The package name was misspelled | Use `erofs-utils` |
| EROFS extraction failed while writing to `/mnt/e` with a permissions error | Linux metadata/permissions did not map as expected to the Windows-mounted destination | Extract inside Ubuntu's filesystem first |
| `wsl` opened `docker-desktop` | Docker Desktop was the default WSL distribution | Run `wsl -d Ubuntu` |
| `Optimize-VHD` was unavailable | The PowerShell command/module was not available in that environment | Use the supported DiskPart compaction method |
| VHDX stayed large after deleting files | Free space inside the Linux filesystem does not automatically guarantee a smaller host VHDX | Clean up, make discardable blocks available, shut down WSL, then compact |
| `fstrim` reported an unexpectedly large amount | The figure reflects filesystem-reported discard, not direct host-space savings | Compare the VHDX size before and after compaction |

---

## 18. Safety and Good Practices

1. **Keep a backup of important ROM files.** Do not modify your only copy of a ROM package.
2. **Verify the output before deleting temporary files.** Open the Windows destination and confirm the expected directories exist.
3. **Identify image formats before extraction.** Different Android partitions may use different filesystem types.
4. **Quote paths that may contain spaces.** This reduces path parsing errors in shell commands.
5. **Use the correct terminal.** Windows PowerShell and Ubuntu have different commands and path formats.
6. **Launch the intended WSL distribution explicitly.** `wsl -d Ubuntu` avoids accidentally working in another distribution.
7. **Be careful with destructive commands.** `rm -rf`, DiskPart, and partition tools can cause data loss if used on the wrong target.
8. **Do not manually delete the WSL VHDX.** It contains the Linux distribution's filesystem.
9. **Do not flash extracted partitions to a phone just because extraction succeeded.** Flashing requires understanding the device's partition layout, ROM compatibility, firmware requirements, and recovery procedure.
10. **Keep notes of commands that worked.** Record the input file, output directory, tool version, and errors so the process is reproducible.

---

## 19. What I Learned

By completing this exercise, I learned how to:

- Explain the difference between an Android ROM package, a partition image, and a dynamic-partition container.
- Use `lpunpack` to split a suitable Android `super` image into logical partition images.
- Recognize that `.img` does not specify a filesystem format.
- Understand why archive tools may not correctly extract filesystem images.
- Install Ubuntu through WSL 2 and use Linux utilities from Windows.
- Navigate between Windows paths and Linux paths.
- Install a package using APT and inspect a command's help output.
- Extract an EROFS filesystem with `fsck.erofs`.
- Work around Linux permission/metadata issues by extracting inside Ubuntu's own filesystem before copying files to Windows.
- Understand the difference between free space inside Ubuntu and the physical size of the WSL VHDX.
- Use `fstrim` and understand why its reported number is not the same as reclaimed Windows space.
- Identify the intended WSL distribution and avoid accidentally working in `docker-desktop`.
- Shut down WSL and compact its virtual disk with DiskPart.
- Verify the result by comparing the VHDX size before and after compaction.

These are useful skills for Linux command-line work, software development, system administration, and Android ROM analysis.

---

## 20. Useful Command Cheat Sheet

### Windows PowerShell / Command Prompt

```powershell
# List installed WSL distributions
wsl --list --verbose

# Launch Ubuntu explicitly
wsl -d Ubuntu

# Check WSL status
wsl --status

# Shut down WSL before disk maintenance
wsl --shutdown
```

### Ubuntu terminal

```bash
# Update package lists
sudo apt update

# Install EROFS utilities
sudo apt install erofs-utils

# Display EROFS utility options
fsck.erofs --help

# Check filesystem space
df -h /

# Summarize a directory's disk usage
du -sh ~/rom_extract

# Clean downloaded APT package files
sudo apt clean

# Request discard/TRIM on eligible mounted filesystems
sudo fstrim -av
```

### DiskPart (run from an elevated Windows terminal)

```text
diskpart
select vdisk file="FULL_PATH_TO_THE_CORRECT_ext4.vhdx"
compact vdisk
exit
```

Replace `FULL_PATH_TO_THE_CORRECT_ext4.vhdx` with the exact VHDX path. Do not copy the placeholder literally.

---

## 21. Next Learning Steps

A sensible next sequence for future Android ROM analysis is:

1. Learn how to identify common Android filesystem formats and image types.
2. Inspect other extracted logical partition images one at a time.
3. Learn the purpose of `system`, `vendor`, `product`, `system_ext`, and `odm`.
4. Learn how Android dynamic-partition metadata describes logical partitions.
5. Learn the difference between read-only inspection, filesystem extraction, image modification, repacking, and flashing.
6. Keep original images and create working copies before experimenting.
7. Document each experiment with the exact command, output, and result.

**Final reminder:** Successful extraction is an analysis milestone, not proof that a modified image is safe to flash. Always verify compatibility and maintain a recovery plan before changing a phone's partitions.
"""
path = Path("/mnt/data/technical-learning-notes.md")
path.write_text(content, encoding="utf-8")
print(f"Created: {path}")
print(f"Size: {path.stat().st_size:,} bytes")
