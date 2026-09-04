# Linux-tkg

This repository provides scripts to automatically download, patch, and compile any version ≥ 5.4 of the Linux kernel.

`linux-tkg` also ships with a selection of patches that can improve the desktop and gaming experience. Most options can be configured by editing the ./customization.cfg file or by following the interactive installation script.

> [!NOTE]
> This repository is a customized fork of [Frogging-Family/linux-tkg](https://github.com/Frogging-Family/linux-tkg).
>
> It retains the standard linux-tkg build and configuration system while including repository-specific maintenance and automated synchronization of selected external kernel patches.
>
> The installation instructions below clone the `Sn0whax/linux-tkg` repository.

- #linux-tkg
  - #important-information
  - #about-this-fork
  - #customization-options
    - #overview
    - #cpu-task-schedulers
      - #runtime-scheduler-swap-sched-ext
      - #build-time-scheduler-swap
    - #bring-your-own-patches
    - #kernel-config-options
  - #install-procedure
    - #arch-and-derivatives
    - #deb-and-rpm-based-distributions
    - #generic-install
    - #gentoo
  - [Updating an existing installation
  - #upstream-project-and-credits

## Important information

- **Support for non-pacman distributions can be considered experimental. You are invited to report any issues you encounter.**
- **If your distribution does not use systemd, set `_configfile="running-kernel"` in `customization.cfg`, or you might end up with a non-bootable kernel.**
- Building recent Linux kernels with GCC requires approximately 20 to 25 GB of disk space. Using LLVM/Clang, LTO, ccache, or enabling additional drivers in the defconfig can increase that requirement.
- NVIDIA drivers may need to be patched to build or work correctly with newer kernels. [Frogging-Family/nvidia-all](https://github.com/Frogging-Family/nvidia-all) may help withs and external patches are not available for every kernel version. The build script displays options based on the selected kernel version.
- Custom kernels can fail to boot because of incompatible patches, missing drivers, unsuitable configuration options, or unsupported external modules. Keep a known-working kernel installed as a fallback.

## About this fork

This repository is based on [Frogging-Family/linux-tkg](https://github.com/Frogging-Family/linux-tkg) and retains its standard build, patching, configuration, and packaging workflow.

This fork additionally maintains selected external patch sets for supported kernel versions. GitHub Actions periodically check the configured upstream patch sources and update the corresponding patch files when newer revisions become available.

Automatically synchronized patch sources include:

- BORE scheduler patches from https://github.com/firelzrd/bore-scheduler
- Linux-hardened patches from https://github.com/anthraxx/linux-hardened

Patch availability depends on whether a compatible patch exists for the selected kernel version.

Unless otherwise noted, the configuration variables and build procedures in this repository remain compatible with upstream linux-tkg.

## Customization options

Most customizations can be configured by:

- Editing variables in ./customization.cfg
- Providing an external configuration file through the `_EXT_CONFIG_PATH` variable
- Setting configuration variables in the shell environment
- Following the interactive installation script, although it does not prompt for every available option

Configuration values use the following order of priority, from lowest to highest:

1. Values in ./customization.cfg
2. Values in the external configuration file
3. Values set in the shell environment

The external configuration file defaults to:

```text
~/.config/frogminer/linux-tkg.cfg
```

Using the external configuration file lets you keep personal build settings outside the Git repository and helps prevent repository updates from overwriting those settings.

### Overview

Here is an overview of several options available in ./customization.cfg:

- `_version`
  - Select the kernel version or branch to build.

- `_timer_freq`
  - Select the kernel timer frequency.
  - Available or recommended values may depend on the selected scheduler.

- `_cpusched`
  - Select a different #cpu-task-schedulers.

- `_processor_opt`
  - Tune the compiled code for a specified processor family or architecture.
  - For example:

    ```shell
    _processor_opt="znver4"
    ```

- `_compileroptlevel`
  - Compile with GCC or Clang using supported optimization levels such as `O2` or `O3`.

- `_lto_mode`
  - Enable a supported LLVM/Clang Link Time Optimization mode.

- `_kernel_on_diet`
  - Use a prepared reduced list of kernel modules.
  - This can reduce compilation time, temporary storage requirements, and final package size.

- `_modprobeddb`
  - Build using a module list generated from modules observed on the current system.

- `_modprobeddb_db_path`
  - Specify the path to the modprobed-db database.

- `_user_patches`
  - Enable detection and application of user-provided patches.

- `_configfile`
  - Select the kernel configuration used as the baseline for the build.

Review ./customization.cfg for the complete and current list of available options.

#### Reduced module builds

Building a kernel with fewer modules can:

- Reduce compilation time
- Reduce temporary storage and memory requirements
- Produce smaller kernel packages
- Reduce the number of unnecessary modules installed on the system

The `_kernel_on_diet` option uses a prepared, reduced module configuration.

Advanced users can instead use `_modprobeddb` and `_modprobeddb_db_path` with https://github.com/graysky2/modprobed-db.

Read the [Arch Linux modprobed-db documentation](https://wiki.archlinux.org/title/Modprobed-db) before relying on a generated module list. A list that is too restrictive can omit storage, filesystem, networking, input, graphics, or other drivers required during boot or normal operation.

#### NTsync and Fsync

Linux-tkg includes options related to NTsync and Fsync support, including support for older kernel versions that do not provide the desired functionality by default.

A compatible or patched Wine build may also be required. See https://github.com/Frogging-Family/wine-tkg-git for the related Wine build project.

### CPU task schedulers

A CPU task scheduler is the algorithm that decides:

- Which task runs
- When each task runs
- How long each task runs
- Which CPU core runs each task
- How processor time is shared among applications, games, background services, and system processes

These decisions can affect throughput and latency.

Throughput describes how much work is completed over a period of time. Latency describes how long a task waits before it can run again.

Gaming and interactive desktop workloads are often sensitive to latency and frame-time consistency. Compilation, rendering, and server workloads may place more importance on sustained throughput.

Linux-tkg supports two general approaches to scheduler customization:

1. Runtime scheduler switching through sched-ext
2. Build-time selection of an alternative scheduler patch

#### Runtime scheduler swap: sched-ext

Starting with kernel 6.12, it is possible to switch supported CPU schedulers at runtime while keeping the kernel's built-in scheduler as a fallback by using https://github.com/sched-ext/scx.

Sched-ext provides several scheduler implementations designed for different workloads.

LAVD is a sched-ext scheduler intended for gaming, interactive, and other latency-sensitive workloads.

Notes:

- Arch users can install sched-ext schedulers from the `scx-scheds` package or from the https://aur.archlinux.org/packages/scx-scheds-git.
- On supported systemd installations, persistent scheduler configuration is commonly stored in:

  ```text
  /etc/default/scx
  ```

- The sched-ext service can be enabled with:

  ```shell
  sudo systemctl enable --now scx
  ```

Runtime scheduler switching makes it possible to test compatible schedulers without rebuilding the entire kernel for each scheduler change.

#### Build-time scheduler swap

EEVDF is the scheduler used by upstream Linux kernels beginning with kernel 6.6. Earlier kernel versions use CFS.

Additional information is available from:

- https://lwn.net/Articles/925371/
- https://en.wikipedia.org/wiki/Completely_Fair_Scheduler

Alternative build-time schedulers are optionally available in linux-tkg.

The availability of each scheduler depends on the selected kernel version.

##### BORE

BORE stands for Burst-Oriented Response Enhancer.

BORE modifies CFS or EEVDF behaviour with the goal of improving responsiveness for bursty and interactive workloads.

Source repository:

- https://github.com/firelzrd/bore-scheduler

##### Project C, PDS, and BMQ

Project C includes scheduler work by Alfred Chen, including the PDS and BMQ scheduler implementations.

Related resources:

- [Project C development blog](http://cchalpha.blogspot.com/)
- https://gitlab.com/alfredchen/projectc

##### MuQSS

MuQSS is a scheduler developed by Con Kolivas.

Related resources:

- http://ck-hack.blogspot.com/
- https://github.com/ckolivas/linux

##### Undead PDS

Undead PDS is linux-tkg's maintained port of the earlier PDS-mq scheduler by Alfred Chen.

PDS-mq was dropped with kernel 5.1 in favour of its BMQ evolution and rework. Because PDS-mq continued to perform well for some gaming workloads, linux-tkg retained it as an optional scheduler for compatible kernel versions.

Alternative schedulers may offer a different balance between:

- Latency
- Responsiveness
- Fairness
- Frame-time consistency
- Background-task behaviour
- Overall throughput

No scheduler is guaranteed to perform best for every workload or hardware configuration.

The build script displays the schedulers that are compatible with the selected kernel version.

### Bring your own patches

To apply your own patches with linux-tkg:

1. Create a version-specific user-patch directory at the root of the repository.
2. Enable user patches in your customization file.
3. Place each patch in the matching directory with the `.mypatch` extension.

The user-patch directory must use the following format:

```text
linuxXY-tkg-userpatches
```

`X` and `Y` represent the kernel's major and minor version numbers.

Examples:

```text
linux65-tkg-userpatches
linux66-tkg-userpatches
linux612-tkg-userpatches
linux618-tkg-userpatches
```

Enable user patches in your configuration:

```shell
_user_patches=true
```

Place patches inside the corresponding directory using the `.mypatch` extension.

Example:

```text
linux612-tkg-userpatches/0001-my-custom-change.mypatch
```

The interactive build process detects compatible `.mypatch` files and asks whether each patch should be applied.

User patches must be compatible with:

- The selected kernel version
- The selected scheduler
- Other enabled patch sets
- Any repository-specific patches already applied during the build

### Kernel config options

Linux-tkg starts from the Arch Linux kernel configuration by default and modifies selected options according to the build configuration.

The current Arch Linux configuration can be viewed in the [Arch Linux kernel packaging repository](https://gitlab.archlinux.org/archlinux/packaging/packages/linux/-/blob/main/config.x86_64).

#### Selecting a configuration baseline

The `_configfile` customization option controls which kernel configuration is used as the baseline.

For example, distributions that do not use systemd should normally use:

```shell
_configfile="running-kernel"
```

A custom configuration can also be selected where supported by the build scripts.

The interactive script may override some configuration entries according to the selected linux-tkg options.

#### Interactive configuration

Before compilation, the interactive installer can offer access to kernel configuration interfaces such as:

```shell
make menuconfig
```

or:

```shell
make xconfig
```

Changes made through these interfaces can be saved as configuration fragments for reuse.

#### Configuration fragments

Configuration fragments override selected values from the baseline configuration immediately before compilation.

Store fragment files at the root of the linux-tkg repository and give them the `.myfrag` extension.

Example:

```text
gaming-options.myfrag
```

The interactive installer detects these files and asks whether each fragment should be applied.

A fragment should contain only the kernel configuration entries that need to be added or overridden.

## Install procedure

For all supported Linux distributions, linux-tkg must be cloned with Git.

It is recommended to clone the repository only once and update it with `git pull`. Linux-tkg retains a relatively large clone of the kernel sources in the `linux-src-git` directory, which is created during the first build from a fresh clone.

Reusing an existing clone can avoid downloading and preparing all source data again.

### Arch and derivatives

Install the required build tools:

```shell
sudo pacman -S --needed base-devel git
```

Clone this repository:

```shell
git clone https://github.com/Sn0whax/linux-tkg.git
cd linux-tkg
```

Optionally edit the configuration:

```shell
nano customization.cfg
```

Build and install the kernel packages:

```shell
makepkg -si
```

The options selected at build time are installed to:

```text
/usr/share/doc/$pkgbase/customization.cfg
```

The exact `$pkgbase` value depends on the package name generated by the selected configuration.

The `base-devel` package group is expected to be installed. See the [Arch Linux makepkg documentation](https://wiki.archlinux.org/title/Makepkg) for additional information.

### DEB and RPM based distributions

The interactive `install.sh` script creates packages appropriate for the selected distribution.

Supported package formats include:

- `.deb` packages for Debian, Ubuntu, and related distributions
- `.rpm` packages for Fedora, RHEL, SUSE, and related distributions

Clone this repository:

```shell
git clone https://github.com/Sn0whax/linux-tkg.git
cd linux-tkg
```

Optionally edit the configuration:

```shell
nano customization.cfg
```

Run the interactive installer:

```shell
./install.sh install
```

Depending on the selected distribution, generated packages are placed in one of the following directories:

```text
DEBS
RPMS
```

The script then offers to install the generated packages using the distribution's package manager.

Support for non-pacman distributions is experimental. Required build dependencies must be installed before starting the build.

Uninstalling custom kernels installed through the script must be performed manually. The script can provide relevant uninstall information:

```shell
cd /path/to/linux-tkg
./install.sh uninstall-help
```

### Generic install

The interactive `install.sh` script can perform a generic installation.

Select `Generic` when prompted for the distribution or select the corresponding generic option in your customization configuration.

Clone this repository:

```shell
git clone https://github.com/Sn0whax/linux-tkg.git
cd linux-tkg
```

Optionally edit the configuration:

```shell
nano customization.cfg
```

Run the installer:

```shell
./install.sh install
```

The script compiles the kernel and asks before performing operations equivalent to:

```shell
sudo cp -R . /usr/src/linux-tkg-${kernel_flavor}
cd /usr/src/linux-tkg-${kernel_flavor}
sudo make modules_install
sudo make install
```

Important notes:

- All dependencies needed to patch, configure, compile, and install the kernel must be installed beforehand.
- Files installed through the generic method may not be tracked by the distribution's package manager.
- Uninstalling a generic installation may require manual intervention.
- Run `./install.sh uninstall-help` for information based on the expected generic installation layout.
- The script uses the Arch Linux kernel configuration as its baseline unless another configuration is selected through `_configfile`.
- The `_libunwind_replace` option can replace `libunwind` with `llvm-libunwind` where supported.
- `${kernel_flavor}` is the default naming scheme, but it can be changed through the `_kernel_localversion` customization option.
- Running `make install` calls `/sbin/installkernel` where available.
- Initramfs, Unified Kernel Image, and bootloader handling depend on the host distribution.

If you only want the script to patch and configure the sources in `linux-src-git`, run:

```shell
./install.sh config
```

For additional information about `installkernel`, see the [Linux kernel installkernel documentation](https://docs.kernel.org/kbuild/kbuild.html#installkernel).

Distribution-specific references:

- [Arch Linux kernel installation documentation](https://wiki.archlinux.org/title/Kernel-install)
- [Gentoo installkernel documentation](https://wiki.gentoo.org/wiki/Installkernel)

### Gentoo

The interactive installer supports Gentoo by following the generic installation procedure with additional Gentoo-specific operations.

Clone this repository:

```shell
git clone https://github.com/Sn0whax/linux-tkg.git
cd linux-tkg
```

Optionally edit the configuration:

```shell
nano customization.cfg
```

Run the interactive installer:

```shell
./install.sh install
```

The Gentoo installation process can:

1. Apply supported Gentoo patches.
2. Install the kernel sources under `/usr/src`.
3. Create or update the `/usr/src/linux` symbolic link.
4. Offer to run `emerge @module-rebuild`.

If you are using an init system other than systemd, set:

```shell
_gentoo_init="script"
```

For a minimal or non-systemd kernel configuration, select an appropriate configuration through `_configfile`.

Otherwise, the default Arch Linux configuration may include systemd-related options that are unnecessary for the Gentoo installation.

## Updating an existing installation

You normally do not need to clone the repository again.

Enter the existing repository directory:

```shell
cd /path/to/linux-tkg
```

Check for local changes:

```shell
git status
```

Pull the latest changes without creating an automatic merge commit:

```shell
git pull --ff-only
```

If you keep personal changes directly in `customization.cfg`, review them before pulling repository updates.

For easier updates, store personal configuration in the external configuration file:

```text
~/.config/frogminer/linux-tkg.cfg
```

The external configuration file can be selected through `_EXT_CONFIG_PATH`.

This keeps personal build settings separate from tracked files and reduces conflicts when pulling updates.

After updating, rebuild the desired kernel using the installation procedure for your distribution.

For Arch and derivatives:

```shell
makepkg -si
```

For supported interactive installation paths:

```shell
./install.sh install
```

## Upstream project and credits

This repository is a customized fork of [Frogging-Family/linux-tkg](https://github.com/Frogging-Family/linux-tkg).

The original linux-tkg project, its maintainers, and its contributors provide the underlying build framework, configuration system, patch integration, packaging support, and documentation on which this fork is based.

Related projects and patch sources include:

- [Frogging-Family/linux-tkg](https://github.com/Frogging-Family/linux-tkg)
- https://github.com/Frogging-Family/wine-tkg-git
- https://github.com/Frogging-Family/nvidia-all
- https://github.com/firelzrd/bore-scheduler
- https://github.com/anthraxx/linux-hardened
- https://github.com/sched-ext/scx
- https://gitlab.com/alfredchen/projectc
- https://github.com/ckolivas/linux
- https://github.com/graysky2/modprobed-db

Refer to the individual patch files and source repositories for applicable authorship, licensing, attribution, compatibility, and support information.
