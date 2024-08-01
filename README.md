# tofugarden.secureboot

This repository contains an Ansible roles to configure secure boot. 

## Overview

Currently, only omne role is provided by this collection: `uki_config`. It does the following
high-level things:

1. Enroll a valid machine owner key (MOK) for image signing
2. Configure `kernel-install` to generate a unified kernel image (UKI) instead of a separate
   kernel, command line and initrd
3. Configure a tool to automatically sign generated UKIs using the enrolled MOK
4. Configure the host to boot straight from `shim` to the UKI, skipping GRUB entirely

## Usage

### Requirements

This role requires that secure boot be enabled on each host. There are not many reasons to
use UKIs without secure boot, so this was assumed. If you would like support for unsigned
UKIs, please submit an issue/PR.

### Layout

This repository does not (yet?) use the standardized collection directory structure. Instead,
the role is stored in `./roles/uki_config` relative to the project root. This should make it
easy to import for use in your own playbook.

### Examples

A test playbook is provided in the project root, under the name `playbook.yaml`. It is configued
to run the role with default arguments for all hosts in a "test" group. An inventory file is not
provided.

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
