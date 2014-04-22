# supermin-wrapper

This program is intended to be friendly to a Continuous Integration / Delivery build pipeline. It should be used after a versioned RPM repository has been produced in order to create reproducible machine images, suitable for use as Phoenix (immutable) servers. As much as possible, it allows one to capture configuration in source control. Where it can't be, it allows the use of scriptlets to generate it.

A wrapper around supermin that creates virtual machine images (disk images) based on a set of repositories. Machine settings are captured in source control and are defined using a mixture of RPM repositories, packages to install, local files to install and programs to generate files to install.


## Syntax Notes

The syntax `${variable}` is used in this document to refer to variables. These refer to the values set when calling supermin-wrapper:-

See `supermin-wrapper -h` for more detailed help. For reference, the variables we refer to are:-

* `${configPath}`, which is the location of all configuration (machines, etc)
* `${cachePath}`, which is the location of a cache, normally at `${configPath}/cache`
* `${functionsPath}`, which is the location of shared functions (lib), normally at `${configPath}/functions`
* `${pathsPath}`, which defines a configuration folder containing the paths to all binaries needed, normally at `${configPath}/paths`

The `${cachePath}` should contain a .gitignore file with the value `*` in it, so that cached data is not accidentally checked in.

The variable `${machineName}` is used to refer to a machine. The value is the machine's hostname (DNS label) without domain. We don't currently validate that it is DNS-valid.

The variable `${machineGroupName}` is used to refer to a group of machines. This need not be a DNS label.


### Configuration
If a file `/etc/supermin-wrapper/configuration` exists, then this is loaded by sourcing - the path variables listed above are replaced and need not be specified on the command line. If the file `~/.supermin-wrapper` exists, then this is sourced after any file `/etc/supermin-wrapper/configuration`.


## Machine templates

Many aspects of a machine's profile are common. To enable this, you can symlink most per-machine files to a common one. By convention, this common folder is at `${configPath}/machine=template`, but it really doesn't matter what it's called, and you could conceivably have several.


## Defining a new machine

Machines are defined in a folder at `${configPath}/${machineGroupName}/machines/${machineName}`.


### Packages

A machine is defined as a set of RPM packages. The packages to install are in a text file at `${configPath}/${machineGroupName}/machines/${machineName}/packages`. They are listed one-per-line (LF line endings), without version numbers or architectures. For example, a file containing:-

    bash
    coreutils

Will install the packages `bash` and `coreutils`, and all their dependencies. The package `setup` is always installed regardless (Contents listed at <http://pkgs.org/centos-6/centos-i386/setup-2.8.14-20.el6_4.1.noarch.rpm.html>).

The `packages` file may be a symlink. The source of these packages is controlled by Yum and Repository Configuration (see below).


#### Yum and Repository Configuration

RPM packages are installed using `yum`. A special configuration is used per-machine, to ensure isolation from the build machine. This needs to exist either as a folder or a symlink to a folder at `${configPath}/${machineGroupName}/machines/${machineName}/yum`. Conventionally, this is symlinked to `${configPath}/templates/machines/yum` as it rarely changes per-machine. This folder contains contents as follows:-

    yum.conf.template  (file template)
    repositories       (folder)
    plugins            (folder, normally empty)


#### `yum.conf.template`

This file is a template `yum.conf`, which has variables marked `${machine_configPath}` and `${machine_cachePath}` substituted before it is used (standard `$YUM0`, etc variables don't work). There should not normally be any need to deviate from that supplied in `${configPath}/templates/machines/yum/yum.conf.template`.


#### `repositories`

This folder contains yum repo configuration snippets (end `.repo`) (examples can be find in `/etc/yum.repos.d` on most RedHat-derived systems). It is recommended that these configuration snippets be changed from those supplied in `${configPath}/templates/machines/yum/repositories` to point to a versioned repository produced by a Continuous Deployment / Integration build pipeline.


### Init scripts

To start a machine, it needs to have an `init` script. This should be an executable file or symlink defined at `${configPath}/${machineGroupName}/machines/${machineName}/init`. This file rarely changes, and is commonly symlinked to `${configPath}/templates/machines/init`.


### Additional Files

Additional files are installed after the RPM packages as overlays onto the image. There are two main kinds:-

* devices, and
* files, known as root-overlays

The reason for the split is that devices (specifically, block device files, character device files and FIFO files) can not be stored in Git and other source control systems.


#### Devices

Devices are stored for each machine in the file `${configPath}/${machineGroupName}/machines/${machineName}/devices`. They are listed one-per-line (LF line endings) with a space-delimited format. For example:-

    /dev/ram0 b 660 1 0
    /dev/tty0 c 620 4 0
    /dev/xconsole p 0755

The format is `device path`, `device type`, `permissions`, `major` and `minor`. The `device type` is as follows:-

* `b` a block device
* `c` a character device
* `p` a FIFO

The `major` and `minor` are positive decimal integers and are zero-based. FIFO devices do not have a `major` and `minor`; specifying them will cause the machine image to fail to build.

The `devices` file may be a symlink.


#### Root Overlays

Root overlays are hierarchal folders and files that will overlay in `/` in the build machine image.

Root overlays are folders or symlinks in `${configPath}/${machineGroupName}/machines/${machineName}/root-overlays`

Due to the design of the supermin-wrapper, any files starting with a period (`.`) (also known as hidden files) in the root of the overlay are not copied. This ensures things like `.gitignore` files are not copied in. Ordinarily, this shouldn't be an issue, because it is exceedingly rare for hidden files to exist in the root (`/`) of a Linux server.

Root overlay folders may be named anything, and are applied in alphanumeric order to the file system image. However, there are three conventional names:-

* `common`, used as symlink to a folder say in `${configPath}/templates/machines/root-overlays/common`, to allow common, shared files to be installed that aren't packaged
* `fixed`, files that are checked into source control and applied without changes
* `generated`, a folder (which should contain a `.gitignore` if using Git), in which generated files can be placed

File generation could be a process before `supermin-wrapper` is invoked, or using generator-scriptlets (see below).

Please note that most source control systems do not preserve file owners or users or permissions, apart from execute. Please also note that files are installed as-is from root-overlays. That means symlinks will not be substituted (but kept as is; consequently, it is recommended that they be relative), and the behaviour of hard links is undefined.


### Generator Scriptlets

Generator scriptlets are sniplets of bash code that supermin-wrapper sources for each machine or machine-group and executes. They can be used to create hostname files, hosts, install SSH private keys, etc. They execute before the root overlays are applied. Conventionally, the output of the sniplets should be placed into the root-overlay `${configPath}/${machineGroupName}/machines/${machineName}/root-overlays/generated`, but this isn't required; any folder in `${configPath}/${machineGroupName}/machines/${machineName}/root-overlays` will do.

Generator scriptlets are run in the order they appear to bash globbing in the folder `${configPath}/${machineGroupName}/machines/${machineName}/generator-scriptlets`. They may be either files or symlinks (conventionally, to files in `${configPath}/templates/machines/generator-scriptlets`, but one can just symlink `${configPath}/templates/machines/generator-scriptlets` to `${configPath}/${machineGroupName}/machines/${machineName}/generator-scriptlets` and run all supplied).

#### Available Environment Variables
When executing a machine-group generator-scriptlet, the following environment variables are available:-

* `${machine_group_configPath}` - where to find any additional configuration
* `${machine_group_cachePath}`
* `${configMachineGroupDevicesPath}` - file or symlink to devices to install
* `${machine_configGeneratorsPath}` - generator scriptlets for machine group
* `${machine_group_configOverlaysPath}` - where to find all root overlays
* `${generatedMachineGroupRootOverlaysPath}` - where to put any generated files and folders for all machines

When executing a machine, in addition to the above, the following environment variables are available:-

* `${machine_configPath}`
* `${configMachineInitPath}` - file or symlink to init script to install
* `${configMachineDevicesPath}` - file or symlink to devices to install
* `${machine_configGeneratorsPath}` - generator scriptlets for machine
* `${machine_configOverlaysPath}` - where to find all root overlays for machine
* `${generatedMachineRootOverlaysPath}` - where to put any generated files and folders for machine
* `${yum_configYumConfTemplateFilePath}`
* `${machine_cachePath}`
* `${cacheMachineMinimalAppliancePath}`
* `${cacheMachineBuiltAppliancePath}`
* `${cacheMachineRpmInstallRootPath}`

## Output

Output consists of machine images, intermediate files, cached package data and logs. All are stored in a cache at `${cachePath}/$[machineGroup}/${machineName}`.


### Machine images

Machine images are created in a folder at `${cachePath}/$[machineGroup}/${machineName}/built-appliance`. Three files are normally present:-

    kernel
    initrd
    root

The file `root` is an ext2 raw disk image (ie `dd`-friendly), and can be inspected by mounting it loopback (`sudo mount -o loop -t ext2 ${cachePath}/$[machineGroup}/${machineName}/built-appliance/root `/mnt/my/path`). The `kernel` and `initrd` are simply copied from the build machine. They are not built. These files can be used without further ado by QEMU: `qemu-kvm -m 512 -kernel `${cachePath}/$[machineGroup}/${machineName}/built-appliance/kernel` -initrd `${cachePath}/$[machineGroup}/${machineName}/built-appliance/initrd` -append 'vga=773 selinux=0' -drive file=`${cachePath}/$[machineGroup}/${machineName}/built-appliance/root`,format=raw,if=virtio`

If the command-line option `-o yes-chroot` is used, then instead the folder `${configPath}/${machineGroupName}/machines/${machineName}/built-appliance` will contain a complete root file system on the current disk. This option doesn't work if running as root, apparently because of a bug in supermin (to do with option specified to tar).


### Intermediate Files

The folder `${cachePath}/$[machineGroup}/${machineName}/minimal-appliance` contains intermediate files.


### Cached Package Data

The folder `${cachePath}/$[machineGroup}/${machineName}/yum` contains cached package data.


### Logs

A yum log is contained in `${cachePath}/$[machineGroup}/${machineName}/yum/log`.


### Clearing the cache

The wrapper uses a cache per machine. To remove the cache for a machine, delete the folder `${cachePath}/$[machineGroup}/${machineName}`. To remove the entire cache, delete `${cachePath}`. Please note that lock files (`supermin.UID.lock`) are in `${cachePath}`, so it can only be safely removed if no instances of supermin (and by implication, supermin-wrapper), are running. The supermin-wrapper will automatically delete machines from the cache that are not defined in `${configPath}/${machineGroupName}/machines`.

# BUGS

* We aren't cleaning up the cache
* We aren't specifying capacity in OVF
* We aren't setting VMDK UUIDs
* native not supported for vmware, parallels
*+ /sbin/new-kernel-pkg --package kernel --install 2.6.32-431.11.2.el6.x86_64
	* awk: cmd. line:1: fatal: cannot open file `/etc/fstab' for reading (No such file or directory)
	* Caused by /etc/fstab not existing when installing the kernel - so we'd need to run a generator before installation
	* Implies that generators are also install programs, that need to run in a certain order

# TODO
* 
* Mixed Resources
	* Have a file detailing installer-kind, package / resource name, source URL (if remote), version (if any), arch (if any), size, crypto hashes, any keys / sigs
		* RPM (rpm-url)
		* RPM (yum-install)
		* RPM (yum-localinstall)
		* Deb (deb-url)
		* Deb (deb-local)
		* Deb (cdebootstrap)
		* Deb (apt)
		* Tarball (compression kind, configure/make/make install, not really all that straightforward, need a spec file or a script, need to specify installation kind)
		* Each install kind is named to match (by convention) a folder in functions/installers
	* Some how combine this with repos
* Documentation
	* Fix the read me, it's really out-of-date
	* Clean up generators (eg grub)
* Installation
	* support deleting files
	* extended attribute control
		* suid / guid whitelisting
		* chattr (append-only, immutable, compression)
		* file capabilities (CAP commands)
		* control timestamps of files (eg set a file's atime / mtime etc)
	* users/groups
		* install /etc/shadow et al
	* a list of urls for RPMs with crypto hashes and signatures
		* rpm supports download from a HTTP/FTP URL
	* Integrate support for a proxy
		* yum / rpm / download
	* support other installs
		* support deb / debootstrap
		* support for install tar balls over root?
		* gem installs?
		* pip / easyinstall (python)?
		* source ball installs - integrate with my lfs code
	* support installation direct to mounted disk, NOT to generated overlay
* Packages
	* comments, whitespace, empty lines in packages files like https://stackoverflow.com/questions/369758/how-to-trim-whitespace-from-bash-variable
	* strip the install list (eg remove glibc, vim)
	* add any other kernel / kernel options (eg kernel-firmware)
* Metadata
	* qemu-kvm needs boot order
	* virtualbox and OVF need boot-order
	* Support vmware VMX
	* support vmware VMC
	* support vmware teams
	* Parallels machine creation
* Wiki
	* Output build reports, etc as wiki mark-up suitable for use with git pages - allowing us to manage server estate details with git!
* Hypervisors
	* Support KVM / RHEV
	* Support Xen / AWS
* performance
	* support parallel machine building (using bash background jobs) (note that there's a max of 10 loopback devices; we use several per machine)
		* Look at 2nd comment on https://www.linuxquestions.org/questions/red-hat-31/how-to-increase-the-loop-devices-number-541717/
	* make yum installation faster
		* baseline disk image with changes
			* based on cryptographic hashes of packages, underlays?
			* done per machine-group?
		* shared cache
			* challenge is that the repos can be different per machine
		* shared downloaded rpms
		* ? use yumdownloader to download rpms (and deps) to a shared local rpm folder?
		* wget the CentOS, etc, repos?
		* http://wiki.centos.org/HowTos/CreateLocalRepos
* Disks
	* Fully support XFS
	* Fully support BTRFS
	* Creation of LVM volume groups and logical volumes
	* Creation of LUKS volumes
	* Creation of RAID volumes
	* support resizing file systems when using disk partitions
	* support compressed vmdk images
	* Can we support ZFS?
* Configuration
	* Machine configuration_machine_operatingSystemId / configuration_machine_operatingSystemName / configuration_machine_uuid / configuration_machine_lastStateChangeTimestamp
	* capture IP and DNS information
	* open-ended extra machine and machine-group configuration for generators
