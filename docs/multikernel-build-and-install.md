# Multikernel on one machine: build, install, run

## 1. Read this first

### 1.1 What this document does

This document builds multikernel from source and runs it on **one
machine**. The same machine compiles the code and runs the result. It
covers four components: the kernel, the `lazy_cma` module, the `daxfs`
module, and the `kerf` tool. It ends with one running kernel instance.

Every command here is checked against the source code of the four
repositories. The source code is the authority. The vendor web page at
multikernel.io is wrong in several places, and section 24 lists the
configuration symbols it omits.

### 1.2 The three rules

1. **Each step ends with a check. Run the check. If it fails, stop.** Do
   not continue to the next step. Section 25 gives the cause for each
   failure that this document knows about.
2. **Do not improvise a command.** If a command fails and section 25
   does not name the cause, report the exact error and stop.
3. **The machine is a workstation, not a test host.** Section 23 lists
   the commands that can make it unbootable. Read that section before
   step 10.

### 1.3 What is not in this document

- The control panel. A separate design document exists at
  `docs/kerf-a-panel-that-fakes-only-the-syscalls.md`.
  It is a design only. Do not build it from this document.
- The assignment of a PCI device to an instance. `kerf init --devices`
  does this, and it needs its own procedure.
- Any change to the four repositories. This document only builds them.

## 2. The machine

| Property | Value |
|---|---|
| CPU | AMD Ryzen 9 7900X, 12 cores, 24 threads |
| Memory | 62 GiB |
| Architecture | x86_64. Multikernel supports x86_64 only. |
| Operating system | Ubuntu 24.04 on hardware |
| Free disk space needed | 40 GB |

The build takes approximately 20 minutes on 24 threads.

This machine also has a WSL2 installation. **The WSL2 filesystem is a
different filesystem.** Files in it are not available after a start into
Ubuntu. Clone the source again, as step 3 says.

## 3. Glossary

| Term | Meaning in this document |
|---|---|
| The machine | The computer of section 2, started into Ubuntu. |
| Pool | The CPUs, the memory, and the devices that multikernel can allocate. |
| Instance | One record of resources that a spawn kernel can use. |
| Spawn kernel | The kernel that runs inside an instance. |
| Insert a module | To put a kernel module into the running kernel. |
| Load a kernel | To put a kernel image into an instance with `kerf load`. |
| Check | To read a value and to compare it with an expected value. |
| APIC ID | The physical identifier of a CPU. It is not the logical CPU number. |

## 4. Where to resume

Run these commands to find the first step that is not complete. Start at
that step.

```bash
uname -r                                  # ends with -mk after step 11
ls ~/multikernel/linux 2>/dev/null        # step 3 is complete
ls ~/multikernel/linux-image-*-mk_*.deb   # step 5 is complete
ls ~/multikernel/lazy_cma/lazy_cma.ko     # step 6 is complete
ls ~/multikernel/daxfs/daxfs/daxfs.ko     # step 7 is complete
ls ~/multikernel/kerf/dist/*.whl          # step 8 is complete
grep GRUB_DEFAULT=saved /etc/default/grub # step 9 is complete
lsmod | grep -E '^(lazy_cma|daxfs) '      # step 12 is complete
command -v kerf                           # step 14 is complete
sudo kerf show                            # step 15 made the pool
```

## 5. Step 1 — Check the firmware

Secure Boot makes `kexec_file_load()` fail with `EPERM`. Multikernel
cannot start an instance without that syscall.

```bash
mokutil --sb-state
```

The expected output is `SecureBoot disabled`.

If the output is `SecureBoot enabled`, stop. Disable Secure Boot in the
firmware setup of the machine. That is a manual action at the next start,
and this document cannot do it.

**Check:** the kernel reports no lockdown.

```bash
cat /sys/kernel/security/lockdown 2>/dev/null || echo "no lockdown file: good"
```

An expected output holds `[none]` in the brackets, or reports no file.

## 6. Step 2 — Install the build packages

```bash
sudo apt update
sudo apt install -y build-essential flex bison bc rsync cpio kmod zstd \
    libelf-dev libssl-dev libncurses-dev dwarves debhelper dwz fakeroot \
    device-tree-compiler python3-dev python3-venv python3-pip swig git \
    initramfs-tools mokutil
```

`dwarves` gives the `pahole` program. The Ubuntu kernel configuration
enables BTF debug information, and that option needs `pahole`.

**Check:** every tool reports a path.

```bash
for t in gcc make flex bison bc pahole dpkg-buildpackage dtc swig; do
    command -v "$t" >/dev/null || echo "MISSING: $t"
done
echo "check complete"
```

The expected output is `check complete` with no `MISSING` line.

## 7. Step 3 — Get the source code

```bash
mkdir -p ~/multikernel
cd ~/multikernel
git clone --depth=1 https://github.com/multikernel/linux.git
git clone --depth=1 https://github.com/multikernel/lazy_cma.git
git clone --depth=1 https://github.com/multikernel/daxfs.git
git clone --depth=1 https://github.com/multikernel/kerf.git
```

The `linux` clone needs approximately 2 GB of disk space. The four
repositories need approximately 3 GB together. The kernel build needs
approximately 30 GB more.

**Check:** the four directories exist.

```bash
ls -d ~/multikernel/linux ~/multikernel/lazy_cma \
      ~/multikernel/daxfs ~/multikernel/kerf
```

## 8. Step 4 — Make the kernel configuration

Start from the configuration of the running Ubuntu kernel. Then enable
the symbols that multikernel needs.

```bash
cd ~/multikernel/linux
cp "/boot/config-$(uname -r)" .config
./scripts/config --file .config \
    --enable  OF \
    --enable  MULTIKERNEL \
    --enable  MKTTY \
    --enable  DMABUF_HEAPS \
    --enable  KEXEC_FILE \
    --enable  MEMORY_HOTPLUG \
    --enable  MEMORY_HOTREMOVE \
    --enable  CRASH_DUMP \
    --disable MODULE_SIG_FORCE \
    --disable DEBUG_INFO_BTF \
    --set-str SYSTEM_TRUSTED_KEYS "" \
    --set-str SYSTEM_REVOCATION_KEYS ""
make olddefconfig
```

Four of these lines need an explanation.

`SYSTEM_TRUSTED_KEYS` and `SYSTEM_REVOCATION_KEYS` name certificate
files of Canonical in an Ubuntu configuration. Those files are not in the
source tree, and the build stops when they are absent.

`DEBUG_INFO_BTF` makes the build slower and multikernel does not need it.

`OF` prevents a spawn kernel that cannot start. It is disabled in an
Ubuntu configuration, and `CONFIG_MULTIKERNEL` does not select it. A
kernel without it builds correctly and starts correctly as a host. Only
the spawn kernel fails, and section 25 gives the message.

**Check:** the command must print `9`.

```bash
grep -cE '^CONFIG_(OF|OF_EARLY_FLATTREE|MULTIKERNEL|MKTTY|DMABUF_HEAPS|CONTIG_ALLOC|KEXEC_CORE|KEXEC_FILE|MEMORY_HOTREMOVE)=y$' .config
```

If the output is less than 9, print the lines and find the symbol that
is absent.

```bash
grep -E '^(CONFIG_(OF|OF_EARLY_FLATTREE|MULTIKERNEL|MKTTY|DMABUF_HEAPS|CONTIG_ALLOC|KEXEC_CORE|KEXEC_FILE|MEMORY_HOTREMOVE)=|# CONFIG_(OF|MULTIKERNEL|MKTTY|DMABUF_HEAPS|KEXEC_FILE)\b)' .config
```

## 9. Step 5 — Build the kernel packages

`LOCALVERSION` puts a name on the kernel release. That name keeps the new
kernel separate from the kernel of Ubuntu.

```bash
cd ~/multikernel/linux
make -j"$(nproc)" LOCALVERSION=-mk bindeb-pkg
```

The build writes the packages into the parent directory.

**Check:** two packages exist.

```bash
ls ~/multikernel/linux-image-*-mk_*.deb ~/multikernel/linux-headers-*-mk_*.deb
```

## 10. Step 6 — Build the lazy_cma module

Build the module against the tree of step 5. Do not build it against the
running kernel.

```bash
cd ~/multikernel/lazy_cma
make KDIR="$HOME/multikernel/linux"
```

The module calls `alloc_contig_range()`. The multikernel tree exports
that function, and the Ubuntu kernel does not export it. Therefore this
module builds against the multikernel tree only.

**Check:** three files exist.

```bash
ls lazy_cma.ko lazy_kdump.ko lazy_cma_tool
```

## 11. Step 7 — Build the daxfs module

```bash
cd ~/multikernel/daxfs
make KDIR="$HOME/multikernel/linux"
```

**Check:** the module exists.

```bash
ls daxfs/daxfs.ko
```

## 12. Step 8 — Build the kerf package

`kerf` is a Python package. Build a wheel in a virtual environment.

```bash
cd ~/multikernel/kerf
python3 -m venv .venv
. .venv/bin/activate
pip install build
python -m build --wheel
deactivate
```

**Check:** the wheel exists.

```bash
ls dist/kerf_multikernel-*.whl
```

## 13. Step 9 — Keep the Ubuntu kernel as the default

**Do this step before step 10.** The kernel package writes a GRUB entry.
GRUB starts the kernel with the highest version, so the multikernel
kernel becomes the default without a question. This machine is a
workstation, so the Ubuntu kernel must stay the default.

Ubuntu also hides the GRUB menu when it holds one operating system.
Without the menu, no other kernel can be selected.

```bash
sudo cp /etc/default/grub /etc/default/grub.backup
sudo sed -i 's/^GRUB_DEFAULT=.*/GRUB_DEFAULT=saved/' /etc/default/grub
sudo sed -i 's/^GRUB_TIMEOUT_STYLE=.*/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
grep -q '^GRUB_TIMEOUT_STYLE=' /etc/default/grub || \
    echo 'GRUB_TIMEOUT_STYLE=menu' | sudo tee -a /etc/default/grub
sudo sed -i 's/^GRUB_TIMEOUT=.*/GRUB_TIMEOUT=10/' /etc/default/grub
sudo update-grub
```

Now read the menu entries and save the Ubuntu one as the default.

```bash
sudo awk -F"'" '/menuentry /{print NR": "$2}' /boot/grub/grub.cfg
```

Choose the entry that names the current kernel release. Read that release
with `uname -r`. Then save it. Use the exact text from the command above.

```bash
sudo grub-set-default "EXACT_ENTRY_TEXT"
```

**Check:** the saved default names the Ubuntu kernel, and the menu is
visible.

```bash
sudo grub-editenv list
grep -E '^GRUB_(DEFAULT|TIMEOUT|TIMEOUT_STYLE)=' /etc/default/grub
```

The expected output holds `saved_entry=` with the Ubuntu entry, plus
`GRUB_DEFAULT=saved`, `GRUB_TIMEOUT=10` and `GRUB_TIMEOUT_STYLE=menu`.

## 14. Step 10 — Install the kernel

```bash
cd ~/multikernel
sudo dpkg -i linux-image-*-mk_*.deb linux-headers-*-mk_*.deb
```

**Check:** the new kernel has a GRUB entry, and the default did not
change.

```bash
sudo awk -F"'" '/menuentry /{print $2}' /boot/grub/grub.cfg | grep -- -mk
sudo grub-editenv list
```

The first command must print an entry that holds `-mk`. The second must
still name the Ubuntu kernel.

## 15. Step 11 — Start the multikernel kernel

```bash
sudo reboot
```

At the GRUB menu, select `Advanced options for Ubuntu`. Then select the
entry that holds `-mk`. This selection is for one start only. The next
start returns to the Ubuntu kernel.

**Check:** the release name ends with `-mk`, the kernel registers the
filesystem, and the kernel gives the DMA heap.

```bash
uname -r
grep multikernel /proc/filesystems
ls /dev/dma_heap/multikernel
```

All three must succeed. If `/dev/dma_heap/multikernel` is absent, read
section 25 and build the kernel again.

## 16. Step 12 — Install the modules

Install the modules into the module directory of the new kernel. Then
insert them.

```bash
sudo mkdir -p "/lib/modules/$(uname -r)/updates"
sudo cp ~/multikernel/lazy_cma/lazy_cma.ko \
        ~/multikernel/lazy_cma/lazy_kdump.ko \
        ~/multikernel/daxfs/daxfs/daxfs.ko \
        "/lib/modules/$(uname -r)/updates/"
sudo depmod -a
sudo modprobe lazy_cma
sudo modprobe daxfs
```

Make the modules persistent for the next start of this kernel.

```bash
printf 'lazy_cma\ndaxfs\n' | sudo tee /etc/modules-load.d/multikernel.conf
```

**Check:** the two modules are present.

```bash
lsmod | grep -E '^(lazy_cma|daxfs) '
```

## 17. Step 13 — Make an initramfs that holds daxfs

A spawn kernel reads its root filesystem through `daxfs`. The `daxfs`
module is out of tree, so it is not inside the kernel image. A spawn
kernel that starts from a Docker image needs an initramfs that inserts
the module.

```bash
echo daxfs | sudo tee -a /etc/initramfs-tools/modules
sudo update-initramfs -c -k "$(uname -r)"
```

**Check:** the module is inside the initramfs.

```bash
lsinitramfs "/boot/initrd.img-$(uname -r)" | grep daxfs
```

## 18. Step 14 — Install kerf

`kerf` needs two Python packages that compile C code at install time.
Those packages are `pylibfdt` and `rdtsc`. Step 2 installed the compiler
and `swig` for them.

```bash
sudo pip install --break-system-packages ~/multikernel/kerf/dist/kerf_multikernel-*.whl
```

`kerf` writes to the kernel, so it runs as root. The option
`--break-system-packages` installs it for root. Ubuntu 24.04 needs this
option.

**Check:** the program reports its version.

```bash
sudo kerf --version
```

The expected output is `kerf, version 0.2.0`.

## 19. Step 15 — Make the pool

`kerf init` takes physical APIC IDs. It does not take logical CPU
numbers. Read the pairs first. `kerf` reads the `apicid` field of
`/proc/cpuinfo`, so this command shows exactly what `kerf` sees.

```bash
grep -E '^(processor|apicid)' /proc/cpuinfo | paste - -
```

Read the thread siblings too. A core has two threads on this machine.
Give both threads of a core to the pool, or neither.

```bash
lscpu -e=CPU,CORE,SOCKET,NODE
```

**Start small.** This machine has 24 threads and 62 GiB. A first pool of
8 threads and 8 GiB leaves the workstation 16 threads and 54 GiB.

Choose the 8 highest APIC IDs. Do not give APIC ID 0 to the pool. The
host needs CPU 0.

```bash
sudo kerf init --cpus=16-23 --memory=8GB --report
```

Replace `16-23` with the APIC IDs that the two commands above reported.

`kerf init` mounts the multikernel filesystem at `/sys/fs/multikernel`
without help. A manual `mount` command is not necessary.

**Check:** the pool exists in two places.

```bash
sudo kerf show
grep -i "Multikernel Memory Pool" /proc/iomem
```

## 20. Step 16 — Start an instance

Three commands start an instance. The first reserves resources. The
second loads the kernel image. The third starts the kernel.

```bash
sudo kerf create web --cpus=22-23 --memory=2GB

sudo kerf load web \
    --kernel="/boot/vmlinuz-$(uname -r)" \
    --initrd="/boot/initrd.img-$(uname -r)" \
    --image=nginx:latest \
    --console=mktty0 \
    --ip=dhcp

sudo kerf exec web
```

Replace `22-23` with two APIC IDs from the pool of step 15.

`kerf exec` accepts an instance in the state `loaded` only. `kerf load`
needs a Docker daemon for `--image`.

**Check:** the status is `active`, and the kernel is in the image table.

```bash
sudo kerf show web
cat /proc/kimage
```

## 21. Step 17 — Attach the console

```bash
sudo kerf console web
```

To detach, press `Ctrl+]`. Then press the period key.

The console closes without help when the instance stops to be `active`.

## 22. Step 18 — Stop the instance and return the resources

Run the three commands in this order.

```bash
sudo kerf kill web      # active -> ready
sudo kerf unload web    # remove the kernel image
sudo kerf delete web    # remove the instance
```

To return the whole pool to the workstation, run `kerf init` with the
value `none`.

```bash
sudo kerf init --cpus=none --memory=none
```

**Check:** no instance is present.

```bash
sudo kerf show
```

## 23. Do not do these things

| Do not | Reason |
|---|---|
| `make install` or `make modules_install` in the kernel tree. | It writes into `/boot` and `/lib/modules` outside the package manager. Step 5 and step 10 use a `.deb` instead. |
| Skip step 9. | GRUB makes the multikernel kernel the default, and a workstation then starts into it every time. |
| Give APIC ID 0 to the pool. | The host needs CPU 0. |
| Give most of the CPUs or most of the memory to the pool. | The pool takes them from the running workstation. |
| Run `kerf init` while an instance holds resources. | A second baseline is refused, and the state becomes hard to read. |
| Build `lazy_cma` or `daxfs` against the running kernel. | The symbols they need exist in the multikernel tree only. |
| Use `/proc/config.gz` from the WSL2 installation. | That configuration is for Hyper-V. A kernel built from it can fail to find the disks. |
| Enable `MODULE_SIG_FORCE`. | The out-of-tree modules are unsigned, and the kernel then refuses them. |
| Treat the instance status `active` as health. | Nothing reports a dead instance. Section 25 explains. |

## 24. Required configuration symbols

| Symbol | Value | Reason | Selected by MULTIKERNEL |
|---|---|---|---|
| `OF` | `y` | The spawn kernel reads its resources from a device tree. | No |
| `OF_EARLY_FLATTREE` | `y` | The kernel unflattens the boot tree. `OF` selects it. | No |
| `MULTIKERNEL` | `y` | The feature itself. | — |
| `MKTTY` | `y` | The console of an instance, at `/dev/mktty`. | No |
| `DMABUF_HEAPS` | `y` | Gives `/dev/dma_heap/multikernel` for DAXFS images. | No |
| `CONTIG_ALLOC` | `y` | The pool allocates contiguous memory. | Dependency |
| `KEXEC_CORE` | `y` | `kerf load` calls `kexec_file_load()`. | Dependency |
| `KEXEC_FILE` | `y` | Gives the `kexec_file_load()` syscall. | No |
| `MEMORY_HOTPLUG` | `y` | The host gives memory to an instance. | Dependency |
| `MEMORY_HOTREMOVE` | `y` | The host takes memory back. | Dependency |

Three symbols cause most failures. `CONFIG_OF`, `CONFIG_MKTTY` and
`CONFIG_DMABUF_HEAPS` are disabled in an Ubuntu configuration, and
`CONFIG_MULTIKERNEL` selects none of them. A kernel without them builds
correctly and starts correctly. The failure comes later.

## 25. Troubleshooting

| Symptom | Cause | Action |
|---|---|---|
| The build stops at `certs/x509_certificate_list`. | The configuration names a certificate file of Canonical. | Clear `SYSTEM_TRUSTED_KEYS` and `SYSTEM_REVOCATION_KEYS`. See step 4. |
| The build reports `pahole: not found`. | The package `dwarves` is absent. | Install `dwarves`, or disable `DEBUG_INFO_BTF`. |
| `dmesg` reports `Boot device tree from the manifest was not unflattened`. | `CONFIG_OF` is disabled. | Enable `CONFIG_OF`. Build the kernel again from step 4. |
| `/dev/dma_heap/multikernel` is absent. | `CONFIG_DMABUF_HEAPS` is disabled. | Enable `CONFIG_DMABUF_HEAPS`. Build the kernel again from step 4. |
| `/dev/mktty` is absent. | `CONFIG_MKTTY` is disabled. | Enable `CONFIG_MKTTY`. Build the kernel again from step 4. |
| `kerf load` reports that `kexec_file_load` failed with `EPERM`. | Secure Boot restricts the kernel. | Disable Secure Boot. See step 1. |
| The spawn kernel starts, and it cannot mount its root filesystem. | The spawn kernel has no `daxfs`. | Give `--initrd` with the initramfs of step 13. |
| `insmod` reports `Unknown symbol alloc_contig_range`. | The module was built against the Ubuntu kernel. | Build it again with `KDIR` set to the multikernel tree. See step 6. |
| `modprobe` reports `Invalid module format`. | The module was built against a different kernel version. | Build the module again after every kernel build. |
| `kerf` reports `Another kerf operation may be in progress`. | The lock file remains after a killed process. | Check that no `kerf` process runs. Then delete `/var/run/kerf.lock`. |
| The status of a dead instance stays `active`. | The kernel reports no liveness. No code sets the state `failed`. | Read the console or `dmesg`. Do not use the status as a health value. |
| `kerf init` reports that the baseline is already applied. | A pool exists and it is in use. | Change the pool with a second `kerf init`. Do not write a second baseline. |
| `kerf create` reports that it cannot grow or allocate memory. | The pool chunk holds no free space at that address. | Read `sudo kerf show`. Choose a smaller size. |
| `pip` refuses to install the wheel. | Ubuntu 24.04 protects the system Python. | Add `--break-system-packages`. See step 14. |
| The machine starts into the multikernel kernel without a question. | Step 9 was skipped or `GRUB_DEFAULT` is not `saved`. | Restore `/etc/default/grub.backup`. Repeat step 9. |
| The GRUB menu does not appear. | `GRUB_TIMEOUT_STYLE` is `hidden`. | Set it to `menu`. Repeat step 9. |
