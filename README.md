# tofugarden.secureboot

This repository contains an Ansible roles to configure secure boot. 

## Overview

Currently, only one role is provided by this collection: `uki_config`. It does the following
high-level things:

1. Enroll a valid machine owner key (MOK) for image signing
2. Configure `kernel-install` to generate a unified kernel image (UKI) instead of a separate
   kernel, command line and initrd
3. Configure a tool to automatically sign generated UKIs using the enrolled MOK
4. Configure the host to boot straight from `shim` to the UKI, skipping GRUB entirely


## Requirements

If you're using Fedora 40, you can satisfy this entire section with:

```sh
dnf install openssl systemd-boot sbsigntools binutils systemd-ukify virt-firmware uki-direct expect
```

The requirements for the `uki_config` role are described in the following three sections:
1. System requirements
2. Package dependencies
3. Argument-specific dependencies

### System requirements

- A UEFI firmware platform with [secure boot](https://en.wikipedia.org/wiki/UEFI#Secure_Boot)
  support. This is not technically a requirement for UKIs, but is considered so by this role.

### Package dependencies

**shim**

This role assumes `shim` is used to authenticate binaries. This should alredy come packaged in any
modern Linux distribution. Support for skipping shim is not provided, but please submit an issue
or PR if you need this.

**kernel-install**

Although a single UKI build can be done without it, this role assumes the use of systemd
`kernel-install` *version 250 or greater*. This is typically provided by the "systemd-udev"
package on modern distributions, but may be too old. To use systemd's "ukify" as a UKI generator,
kernel-install *version 255 or greater* must be installed.

**virt-firmware**

The virt-firmware python package is required to configure shim to boot straight to your UKI. On
Fedora 40, the tools needed are split into the `virt-firmware` main package and `uki-direct`
subpackage. 

On other platforms, you may be able to use your python package manager of choice, or
[get it from PyPI](https://pypi.org/project/virt-firmware/). If you do this without a system
package, you will need to make sure:

- `kernel-bootcfg` is in your path
- `99-uki-uefi-setup.install` is executable in `/etc/kernel/install.d`
- `kernel-bootcfg-boot-successful.service` is loaded

**openssl**

Openssl is required by the `community.crypto` modules used by the role. Even if you choose to
import your own MOK, the role still checks it for validity.

**systemd-stub**

The systemd kernel boot stub is required to make the unified kernel image bootable. On Fedora,
this is shipped with the "systemd-boot" package.

**sbsign**

The `sbsign` utility is required to sign UKIs. This is usually provided by the "sbsigntools"
package.

### Argument-dependent dependecies

**dracut**

If you wish to use `dracut` as your initramfs *or* UKI generator, you should have it installed.
Note that this is currently the only supported initramfs generator, but this will be expanded
soon.

**objcopy**

Using dracut as your UKI generator requires the `objcopy` tool, provided by the "binutils"
package on Fedora.

**ukify**

If you wish to use `ukify` as your UKI generator, you should have it installed. This does bump
the kernel-install version requirement to 255, so be mindful of its availability on your distro.
This is usually provided by the "systemd-ukify" package.

**mokutil**

The `mokutil` tool is used to validate the enrollment status of your generated or provided MOK.

**expect**

This role uses the `expect` executable to interact with `mokutil` *only if you need to enroll a
new MOK*. If you run the role with a MOK already enrolled through MokManager, `expect` will
never be called.

### Interaction

This playbook may require manual administrator interaction. If you choose to generate a new MOK
(the default) or import a MOK that is not already enrolled, you will be prompted twice:
1. Once to define a MOK enrollment password. This should be something easy to type but still
   secure, as explained in prompt 2.
2. Then to reboot into MokManager. The playbook will reboot for you and resume once complete,
   but only the administrator with physical access to a console/display can complete a new
   MOK enrollment.

This means this role may not be suitable for any Fedora 40 environment. Most cloud providers
for example will never provide a pre-boot console or virtual display, and therefore cannot
support custom MOK enrollment. Most hypervisors will support this however (this author uses
Hyper-V). The reboot prompt is meant to pause and allow the caller time to bring up the proper
console or display.

## Arguments

All arguments have default values, reflected in the following example:

```yaml
- name: Test playbook
  hosts: test
  roles:
    - role: uki_config
      vars:
        uki_config_initrd_generator: dracut
        uki_config_uki_generator: ukify
        uki_config_cmdline: /etc/kernel/cmdline
        uki_config_kernel_intall_config_root: /etc/kernel
        uki_config_dracut_conf_dir: /etc/dracut.conf.d
        uki_config_mok:
          private_key: /etc/kernel/MOK.priv
          certificate: /etc/kernel/MOK.cer
          owner: root
          group: root
          mode: "0600"
          selevel: s0
          seuser: system_u
          serole: object_r
          setype: cert_t
```

### Initrd Generator

The only accepted option is dracut. Please submit an issue/PR if you want support for another.

### UKI Generator

You may select:
- dracut
- ukify

### Kernel Command Line

You may substitute this path to any path readable by root. Passing the content of the kernel
command line is not supported. Please submit an issue/PR if you want support for this.

> Note: The kernel command line is ignored when dracut is the UKI generator. Please configure
        dracut yourself if you want a different command line.

### Configuration Directories

The `uki_config_kernel_install_config_root` and `uki_config_dracut_conf_dir` arguments allow
you to specify where you custom configuration should be applied. You may for example wish to
keep it under `/usr/lib/kernel` and `/usr/lib/dracut` respectively.

### MOK Information

By default, a MOK is created at the path specified under `private_key` and `certificate` if
an adequate key/certificate pair does not already exist at that path. If you wish to bring
your own MOK instead of generating a new one for each host, either write your files to the
default paths, or provide custom paths.

> Note: The author considered adding support for `pesign` as a signing engine (this would
        only be available with the "ukify" UKI generator), but this was refused for time's
        sake and for the ability to import private keys. If you want support for this,
        submit an issue/PR and we'll talk about it.
