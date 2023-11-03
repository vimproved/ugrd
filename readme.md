# µgRD

> Microgram Ramdisk is a framework used to generate ramdisks using TOML definitions and python functions

## Project goal

ugrd is designed to generate very custom initramfs environments. The final environment will be left in `build_dir` where it can be explored or modified.

The original goal of this project was to create an initramfs suitable for decrypting LUKS volumes, currently it supports the following:

* OpenPGP Smartcards (YubiKey)
* GPG encrypted LUKS keyfiles
* LUKS with detatched headers
* BTRFS subvolumes and RAID
* Key entry over serial
* Automatic CPIO generation

## Usage

To install `ugrd`, clone the repo and run `pip install .`, setting `--break-system-packages` may be required.

Once installed, either set the config file in `/etc/ugrd/config.toml`, or pass it to the script:

`ugrd example_config.toml`

> Debug mode can be enabled with `-d` or verbose debugging with `-dd`

## Output

Unless the `ugrd.base.cpio` module is included, an initramfs environment will be generated at `build_dir` which defaults to `/tmp/initramfs/`.

This directory can be embedded into the Linux kernel using `CONFIG_INITRAMFS_SOURCE="/tmp/initramfs"`.
`CONFIG_INITRAMFS_SOURCE` can also be pointed at a CPIO archive, but is easiest to use with a directory.

If a CPIO file is generated, it can be passed to the bootloader. Embedding the initramfs into the kernel is preferred, as the entire kernel image can be signed.

## Configuration

At runtime, ugrd will try to read `config.toml` for configuration options unless another file is specified..

### Base modules

Several basic modules are provided for actions such as mounts, config processing, and other basic parameters.

Modules write to a shared config dict that is accessible by other modules.

#### base.base

> The main module, mostly pulls basic binaries and pulls the `core` and `mounts` modules

* `shebang` (#!/bin/bash) sets the shebang on the init script.

#### base.core

* `build_dir` (/tmp/initramfs) Defines where the build will take place.
* `out_dir` (/tmp/initramfs_out) Defines where packed files will be placed.
* `clean` (true) forces the build dir to be cleaned on each run.
* `file_owner` (portage) sets the owner for items pulled into the initramfs on the build system
* `binaries` is a list used to define programs to be pulled into the initrams. `which` is used to find the path of added entries, and `lddtree` is used to resolve dependendies.
* `paths` is a list of directores to create in the `build_dir`. They do not need a leading `/`.

##### symlink creation

Symlinks are defined in the `symlinks` dict. Each entry must have a name, `source` and `target`:

```
[symlinks.pinentry]
source = "/usr/bin/pinentry-tty"
target = "/usr/bin/pinentry"
```


##### Copying files to a different destination

Using the `dependencies` list will pull files into the initramfs using the same path on the host system.

To copy files to a different path:

```
[copies.my_key]
source = "/home/larry/.gnupg/pubkey.gpg"
destination = "/etc/ugrd/pub.gpg"
```

##### Device node creation

Device nodes can be created by defining them in the `nodes` dict using the following keys:

* `mode` (0o600) the device node, in octal.
* `path` (/dev/node name) the path to create the node at.
* `major` Major value.
* `minor` Minor value.

Example:

```
[nodes.console]
mode = 0o644
major = 5
minor = 1
```

Creates `/dev/console` with permissions `0o644`

> Using `mknod_cpio` from `ugrd.base.cpio` will not create the device nodes in the build dir, but within the CPIO archive

#### base.console

This module creates an agetty session. This is used by the `ugrd.crypto.gpg` module so the tty can be used for input and output.

Consoles are defined by name in the `console` dict using teh following keys:

* `type` (tty) specifies the console type, such as `tty` or `vt100`.
* `baud` Specified the baud rate for serial devices.
* `local` (false) specifies whether or not the `-L` flag should be passed to agetty.

ex:

```
[console.tty0]
type = "tty"
```

Defines the default `/dev/tty0` console.

```
[console.ttyS1]
baud = 115_200
type = "vt100"
local = true
```

Defines /dev/ttyS1 as a local `vt100` terminal with a `115200` baud rate.

##### General console options

`primary_console` (tty0)  is used to set which console will be initialized with agetty on boot.

#### base.cpio

This module assists in creating a CPIO out of the initramfs `build_dir`.

> This module is run during the `pack` phase.

The following parameters can be set to alter CPIO functionality:

* `mknod_cpio` (true) Only create the device nodes within the CPIO file.
* `cpio_filename` (ugrd.cpio) can be set to change the final CPIO filename in the `out_dir`.
* `cpio_list_name` (cpio.list) can be used to change the filename of the CPIO list for `gen_init_cpio`.
* `_gen_init_cpio_path` The path to this tool can be specified. If not, it is included and will be built at runtime if needed.

#### base.debug

This module contains debug programs such as `cp`, `mv`, `rm`, `grep`, `dmesg`, `find`, and `nano`,

Setting `start_shell` to `true` will start a bash shell in `init_debug`.

### Kernel modules

`ugrd.kmod.kmod` is the core of the kernel module loading..

> Modules can use `_kmod_depend` to add required modules. Simply using the `ugrd.crypto.cryptsetup` module, for example, will try to add the `dm_crypt` kmod.

#### ugrd.kmod.kmod confugration parameters

The following parameters can be used to change the kernel module pulling and initializing behavior:

* `kernel_version` (uname -r) is used to specify the kernel version to pull modules for, should be a directory under `/lib/modules/<kernel_version>`.
* `kmod_init`  is used to specify kernel modules to load at boot. If set, ONLY these modules will be loaded with modprobe.
* `kmod_autodetect_lspci` (false) if set to `true`, will populate `kernel_modules` with modules listed in `lspci -k`.
* `kmod_autodetect_lsmod` (false) if set to `true`, will populate `kernel_modules` with modules listed in `lsmod`.
* `kernel_modules` is used to define a list of kernel module names to pull into the initramfs. These modules will not be `modprobe`'d automatically if `kmod_init` is also set.
* `kmod_ignore` is used to specify kernel modules to ignore. If a module depends on one of these, it will throw an error and drop it from being included.
* `kmod_ignore_softdeps` (false) ignore softdeps when checking kernel module dependencies.
* `_kmod_depend` is meant to be used within modules, specifies kernel modules which should be added to `kmod_init` when that `ugrd` module is imported.

#### Kernel module helpers

Some helper modules have been created to make importing required kernel modules easier.

`base.ugrd.kmod_nvme`, `kmod_usb`, and `kmod_fat` can be used to load modules for NVME's, USB storage, and the FAT file system respectively.

Similarly `base.ugrd.kmod_novideo` and `kmod_nosound` exist to ignore video and sound devices that may appear when autodetecting modules.

### Filesystem modules

#### ugrd.fs.mounts

`mounts`: A dictionary containing entries for mounts, with their associated config.

`mounts.root` is predefined to have a destination of `/mnt/root` and defines the root filesystem mount, used by `switch_root`.

Each mount has the following available parameters:

* `type` (auto) the mount or filesystem type.
  - Setting the `type` to `vfat` includes the `vfat` kernel module automatically.
  - Setting the `type` to `btrfs` imports the `ugrd.fs.btrfs` module automatically.
* `destination` (/mount name) the mountpoint for the mount, if left unset will use /mount_name.
* `source` The source string or a dict with a key containing the source type, where the value is the target.
  - `uuid` Mount by the filesystem UUID.
  - `partuuid` Mount by the partition UUID.
  - `label` Mount by the device label.
* `options` A list of options to add to the mount.
* `base_mount` (false) is used for builtin mounts such as `/dev`, `/sys`, and `/proc`. Setting this to mounts it with a mount command in `init_pre` instead of using `mount -a` in `init_main`.
* `skip_unmount` (false) is used for the builtin `/dev` mount, since it will fail to unmount when in use. Like the name suggests, this skips running `umount` during `init_final`.
* `remake_mountpoint` will recreate the mountpoint with mkdir before the `mount -a` is called. This is useful for `/dev/pty`.

The most minimal mount entry that can be created must have a name, which will be used as the `destination`, and a `source`.

The following configuration mounts the device with `uuid` `ABCD-1234` at `/boot`:

```
[mounts.boot.source]
uuid = "ABCD-1234"
```

The following configuration mounts the `btrfs` subvolume `stuff`  with `label` `extra` to `/mnt/extra`:

```
[mounts.extra]
options = [ "subvol=stuff" ]
type = "btrfs"
destination = "/mnt/extra"

[mounts.extra.source]
label = "extra"
```

##### General mount options

These are set at the global level and are not associated with an individual mount:

* `mount_wait` (false) waits for user input before attenmpting to mount the generated fstab at `init_main`.
* `mount_timeout` timeout for `mount_wait` to automatically continue.


#### ugrd.fs.btrfs

Importing this module will run `btrfs device scan` and pull btrfs modules. No config is required.

### Cryptographic modules

Several cryptographic modules are provided, mostly to assist in mounting encrypted volumes and handling keyfiles.

#### ugrd.crypto.gpg

This module is required to perform GPG decryption within the initramfs. It depends on the `ugrd.base.console` module for agetty, which is required for input. Additionally, it depends on the `ugrd.crypt.cryptsetup` module, so both do not need to be defind.

`gpg_agent_args` is an append-only list which defines arguments passed to `gpg-agent`.

This module sets the `primary_console` to `tty0` and creates the console entry for it.
This configuration can be overriden in the specified user config if an actual serial interface is used, this is demonstrated in `config_raid_crypt_serial.toml`

#### ugrd.crypto.smartcard

Depends on the `ugrd.crypto.gpg` submodule, meant to be used with a YubiKey.

> Sets `cryptsetup_autoretry` to false

`sc_public_key` should point to the public key associated with the smarcard used to decrypt the GPG protected LUKS keyfile.
This file is added as a dependency and pulled into the initramfs.

#### ugrd.crypto.cryptsetup

This module is used to decrypt LUKS volumes in the initramfs.

> Modules such as the GPG and smartcard modules pull this automatically

Cryptsetup mounts can be configured with the following options:
* `key_type` - The type of key being used, if one is being used.
* `key_file` - The path of a key file.
* `key_command` - The command used to unlock or use the key.
* `reset_command` - The command to be used between unlock attempts.
* `header_file` - The path of the luks header file.
* `partuuid` - The partition UUID of the LUKS volume.
* `uuid` - The UUID fo the LUKS filesystem.
* `retries` (5) - The number of times to attempt to unlock a key or cryptsetup volume.
* `try_nokey` - (false) - Whether or not to attempt unlocking with a passphrase if key usage fails

`cryptsetup` is a dictionary that contains LUKS volumes to be decrypted.

A minimal defintion to decrypt a volume protected by a passphrase:

```
[cryptsetup.root]
uuid = "9e04e825-7f60-4171-815a-86e01ec4c4d3"
```

A cryptsetup mount which retries only 3 times, uses the key file `/boot/luks.gpg` with header file `/boot/luks_headers.img`:

```
[crytpsetup.root]
retries = 3
partuuid = "9e04e825-7f60-4171-815a-86e01ec4c4d3"
header = "/boot/luks_headers.img"
key_file = "/boot/luks.gpg"
key_type = "gpg"
```

#####

Cryptsetup global config:

* `cryptsetup_key_type` (keyfile) Used to determine how a key is unlocked, setting it globally changes the default for definitions
* `cryptsetup_retries` (5) The default number of times to try to unlock a device.
* `cryptsetup_autoretry` (false) Whether or not to automatically retry mount attempts.

##### Key type definitions

New key types can defined using the `cryptsetup_key_types` dict. At least `key_command` must be specified. The name of the key file is added to the end of this command:

```
[cryptsetup_key_types.gpg]
key_command = "gpg --decrypt {key_file} >"
```

Gets turned into:

```
gpg --decrypt /boot/luks.gpg > /run/key_root
```

When used with:

```
[cryptsetup.root]
key_type = "gpg"
key_file = "/boot/luks.gpg"
```

## Modules

The modules config directive should contain a list with names specifying the path of which will be loaded, such as `ugrd.base.base`, `ugrd.base.console` or `ugrd.crypto.crypsetup`.

Another directory for modules can be created, the naming scheme is similar to how python imports work.

When a module is loaded, `initramfs_dict.py` will try to load the toml file for that module, parsing it in the same manner `config.yaml` is parsed.

The order in which modules/directives are loaded is very important!

Modules can load other modules using the `modules` directive, be careful considering loading orders.

If a module depends on another module, it can be added to the `mod_depends` list in the module config. A `ValueError` will be thrown if the module is not present.

#### imports

The most powerful part of a module is the `imports` directive.

Imports are used to hook into the general processing scheme, and become part of the main `InitramfsGenerator` object.

Portions are loaded into the InitramfsGenerator's `config_dict` which is an `InitramfsConfigDict`

`imports` are defined like:

```
[imports.<hook>]
"module_dir.module_name" = [ "function_to_inject" ]
```

For example:

```
[imports.build_tasks]
"ugrd.fs.mounts" = [ "generate_fstab" ]
```

Is used in the base module to make the initramfs generator generate a fstab durinf the `build_tasks` phase.

Imported functions have access to the entire `self` scope, giving them full control of whatever other modules are loaded when they are executed, and the capability to dynamically create new functions.

This script should be executed as root, to have access to all files and libraries required to boot, so special care should be taken when loading and creating modules. 

#### masks

To mask an import used by another module, the mask parameter can be used:

```
[mask]
init_final = ['mount_root']

```

This will mask the `mount_root` function pulled by the base module, if another mount function is being used.

#### config_processing

These imports are very special, they can be used to change how parameters are parsed by the internal `config_dict`.

A good example of this is in `base.py`:

```
def _process_mounts_multi(self, key, mount_config):
    """
    Processes the passed mounts into fstab mount objects
    under 'mounts'
    """
    if 'destination' not in mount_config:
        mount_config['destination'] = f"/{key}"  # prepend a slash

    try:
        self['mounts'][key] = FstabMount(**mount_config)
        self['paths'].append(mount_config['destination'])
    except ValueError as e:
        self.logger.error("Unable to process mount: %s" % key)
        self.logger.error(e)
```

This module manages mount management, and loads new mounts into fstab objects, also defined in the base module.

The name of `config_prcessing` functions is very important, it must be formatted like `_process_{name}` where the name is the root variable name in the yaml config.

If the function name has `_multi` at the end, it will be called using the `handle_plural` function, iterating over passed lists/dicts automatically.

A new root varaible named `oops` could be defined, and a function `_process_oops` could be created and imported, raising an error when this vlaue is found, for example.

This module is loaded in the imports section of the `base.yaml` file:

```
[imports.config_processing]
"ugrd.fs.mounts" = [ "_process_mounts_multi" ]
```

#### build_tasks

Build tasks are functions which will be executed after the directory structure has been generated using the specified `paths`.

The base module includes a build task for generating the fstab, which is activated with:

```
[imports.build_tasks]
"ugrd.fs.mounts" = [ "generate_fstab" ]
```

#### Packing tasks

Packing facts, such as CPIO generation can be defined in the `pack` import.

The `cpio` module imports the `make_cpio_list` packing function with:

```
[imports.pack]
"ugrd.base.cpio" = [ "make_cpio_list" ]
```

#### init hooks

By default, the specified init hooks are:
* `init_pre` - Where the base initramfs environment is set up, such as creating a devtmpfs.
* `init_debug` - Where a shell is started if `start_shell` is enabled in the debug module.
* `init_early` - Where early actions such as checking for device paths, mounting the fstab take place.
* `init_main` - Most important initramfs activities should take place here.
* `init_late` - Space for additional checks, stuff that should run later in the init process.
* `init_prenount` - Where filesystem related commands such as `btrfs device scan` can run.
* `init_mount` - Where the root filesystem mount takes place
* `init_cleanup` - Where filesystems are unmounted and the system is prepared to `exec switch_root`
* `init_final` - Where `switch_root` is executed.

> These hooks are defined under the `init_types` list in the `InitramfsGenerator` object.

When the init scripts are generated, functions under dicts in the config defined by the names in this list will be called to generate the init scripts.

Init functions should return a string or list of strings that contain shell lines to be added to the `init` file.

The `InitramfsGenerator.generate_init_main()` function (often called from `self`) can be used to output all init hook levels but `init_pre` and `init_final`.

A general overview of the procedure used for generating the init is to write the chosen `shebang`, then every init hook. The `custom_init` import can be used for more advanced confugrations, such as running another script in `agetty`.

#### custom_init

To change how everything but `init_pre` and `init_file` are handled at runtime, `custom_init` can be used.

The `console` module uses the `custom_init` hook to change the init creation procedure.

Like with the typical flow, it starts by creating the base `init` file with the shebang and `init_pre` portions. Once this is done, execution is handed off to all fucntions present in the `custom_init` imports.

Finally, like the standard init build, the `init_final` is written to the main `init` file.

```
[imports.custom_init]
"ugrd.base.console" = [ "custom_init" ]
```

The custom init works by creating an `init_main` file and returning a config line which will execute that file in a getty session.
This `init_main` file contains everything that would be in the standard init file, but without the `init_pre` and `init_final` portions. 


```

def custom_init(self):
    """
    init override
    """
    custom_init_contents = [self.config_dict['shebang'],
                            f"# Console module version v{__version__}"]
    custom_init_contents += self.generate_init_main()

    self._write("init_main.sh", custom_init_contents, 0o755)

    return console_init(self)


def console_init(self):
    """
    start agetty
    """
    name = self.config_dict['primary_console']
    out_str = f"agetty --autologin root --login-program /init_main.sh {name}"

    console = self.config_dict['console'][name]

    if console.get('local'):
        out_str += " -L"

    console_type = console.get('type', 'tty')

    if console_type != 'tty':
        baud_rate = console['baud']
        out_str += f" {console_type} {baud_rate}"
    else:
        out_str += f" {console_type}"

    return out_str
```


`self._write(file_name, contents, chmod)` should be used instead of manually writing to a file.
